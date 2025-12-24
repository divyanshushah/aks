# Integrate Azure Key Vault with Azure Kubernetes Service using External Secret Operator with Workload Identity

### Create AKS Cluster with Workload Identity
```bash
RG=aks-rg
LOC=eastus2
CLUSTER_NAME=aks-cluster
UNIQUE_ID=$CLUSTER_NAME$RANDOM
KEY_VAULT_NAME=$UNIQUE_ID

# Create the resource group
az group create -g $RG -l $LOC

# Create the cluster with the OIDC Issuer and Workload Identity enabled
az aks create -g $RG -n $CLUSTER_NAME \
--node-count 1 \
--enable-oidc-issuer \
--enable-workload-identity \
--generate-ssh-keys

# Get the cluster credentials
az aks get-credentials -g $RG -n $CLUSTER_NAME
```

### Verify WorkLoad Identity is Enabled

```bash
az aks show --resource-group ${RG} --name ${CLUSTER_NAME} -o json|jq '.securityProfile.workloadIdentity'


```
### View details of Azure Kubernetes Service (AKS) cluster

```bash

az aks show \
--resource-group $RG \
--name $CLUSTER_NAME \
--output jsonc

```

### Set Up Identity
```bash
# Get the OIDC Issuer URL
export AKS_OIDC_ISSUER="$(az aks show -n $CLUSTER_NAME -g $RG --query "oidcIssuerProfile.issuerUrl" -otsv)"

# Get the Tenant ID for later
export IDENTITY_TENANT=$(az account show -o tsv --query tenantId)

# Create the managed identity
az identity create --name kvcsi-demo-identity --resource-group $RG --location $LOC

# Get identity client ID
export USER_ASSIGNED_CLIENT_ID=$(az identity show --resource-group $RG --name kvcsi-demo-identity --query 'clientId' -o tsv)

# Create a service account to federate with the managed identity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  labels:
    azure.workload.identity/use: "true"
  name: kvcsi-demo-sa
  namespace: default
EOF

# Federate the identity
az identity federated-credential create \
--name kvcsi-demo-federated-id \
--identity-name kvcsi-demo-identity \
--resource-group $RG \
--issuer ${AKS_OIDC_ISSUER} \
--subject system:serviceaccount:default:kvcsi-demo-sa

```






### Create Azure Key Vault and Set Secret

```bash
az keyvault create --name $KEY_VAULT_NAME --resource-group $RG --location $LOC --enable-rbac-authorization false

# Create a secret
az keyvault secret set --vault-name $KEY_VAULT_NAME --name "TestSecret" --value "Hello from Key Vault"

# Grant access to the secret for the managed identity
az keyvault set-policy --name $KEY_VAULT_NAME -g $RG --secret-permissions get --spn "${USER_ASSIGNED_CLIENT_ID}"

# Optionally go with RBAC

```

### Setup ESO
``` bash
helm repo add external-secrets https://charts.external-secrets.io 

helm repo update

helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace

```
### Validate ESO is Installed
``` bash
kubectl api-resources |grep external-secrets
```

### Create SecretStore    
```bash
#Create a secret store which references the serviceaccount for KV access
cat <<EOF | kubectl apply -f -
---
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: azure-store
spec:
  provider:
    azurekv:
      authType: WorkloadIdentity
      tenantId: "${IDENTITY_TENANT}"
      vaultUrl: "https://${KEY_VAULT_NAME}.vault.azure.net"
      serviceAccountRef:
        name: kvcsi-demo-sa
EOF

# Check if Secret Store is created
kubectl get secretstore

```

### create External secret which will sync secret
``` bash
cat <<EOF | kubectl apply -f - 
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: credentials
spec:
  refreshInterval: 5m
  secretStoreRef:
    kind: SecretStore
    name: azure-store # name of the secret store

  target:
    name: local-credentials # name of the kuberntes secret
    creationPolicy: Owner

  data:  
  - secretKey: kvcredentials # name of the key inside the corresponding kubernetes secret
    remoteRef:
      key: secret/TestSecret  # Give secret name from Azure KV
EOF

# Check if externalsecrets is created
kubectl get externalsecrets
```

### Validate Secret
``` bash
kubectl get  secret local-credentials -o jsonpath='{.data.kvcredentials}' | base64 -d
```


### Now change Secret in AKV and  Validate again after waiting for 5 mins
``` bash
kubectl get  secret local-credentials -o jsonpath='{.data.kvcredentials}' | base64 -d
#New secret should show up
```