# Integrate Azure Key Vault with Azure Kubernetes Service using CSI Driver with Workload Identity

### Create AKS Cluster with Azure keayvault secrets provider` addon
```bash
RG=aks-rg
LOC=eastus2
CLUSTER_NAME=aks-cluster
UNIQUE_ID=$CLUSTER_NAME$RANDOM
KEY_VAULT_NAME=$UNIQUE_ID

# Create the resource group
az group create -g $RG -l $LOC

# Create the cluster with the OIDC Issuer and Workload Identity enabled
# azure-keyvault-secrets-provider is aks addon to enable secret provider
az aks create -g $RG -n $CLUSTER_NAME \
--node-count 1 \
--enable-oidc-issuer \
--enable-workload-identity \
--enable-addons azure-keyvault-secrets-provider \
--generate-ssh-keys

# Get the cluster credentials
az aks get-credentials -g $RG -n $CLUSTER_NAME
```

### Verify the `Azure keyvault secrets provider` CSI driver installation

```bash

kubectl get pods \
--namespace kube-system \
--selector 'app in (secrets-store-csi-driver,secrets-store-provider-azure)'

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

### Create SecretProviderClass
    
```bash
# Create a secret provider instance attached to the vault and secret
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvcsi-wi # needs to be unique per namespace
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "${USER_ASSIGNED_CLIENT_ID}" # Setting this to use workload identity
    keyvaultName: ${KEY_VAULT_NAME}       # Set to the name of your key vault
    cloudName: ""                         # [OPTIONAL for Azure] if not provided, the Azure environment defaults to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: TestSecret             # Set to the name of your secret
          objectType: secret              # object types: secret, key, or cert
          objectVersion: "${VERSION_ID}"               # [OPTIONAL] object versions, default to latest if empty
    tenantId: "${IDENTITY_TENANT}"        # The tenant ID of the key vault
EOF

```

### create a pod and mount secret  as volume
``` bash
# Create a pod to mount the secret
cat <<EOF | kubectl apply -f -
kind: Pod
apiVersion: v1
metadata:
  name: secrets-store-inline-wi
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: "kvcsi-demo-sa"
  containers:
    - name: ubuntu
      image: ubuntu:20.04
      command: [ "/bin/bash", "-c", "--" ]
      args: [ "while true; do sleep 30; done;" ]
      volumeMounts:
      - name: secrets-store01-inline
        mountPath: "/mnt/secrets-store"
        readOnly: true
  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-kvcsi-wi"
EOF

# Check the secret loaded
kubectl exec -it secrets-store-inline-wi -- cat /mnt/secrets-store/TestSecret
```

### SecretProviderClass with Kubernetes Secret
``` bash
# Create a secret provider instance attached to the vault and secret
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvcsi-wi-sync # needs to be unique per namespace
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "${USER_ASSIGNED_CLIENT_ID}" # Setting this to use workload identity
    keyvaultName: ${KEY_VAULT_NAME}       # Set to the name of your key vault
    cloudName: ""                         # [OPTIONAL for Azure] if not provided, the Azure environment defaults to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: TestSecret             # Set to the name of your secret
          objectType: secret              # object types: secret, key, or cert
          objectVersion: "${VERSION_ID}"               # [OPTIONAL] object versions, default to latest if empty
    tenantId: "${IDENTITY_TENANT}"        # The tenant ID of the key vault
  secretObjects:                              # [OPTIONAL] SecretObjects defines the desired state of synced Kubernetes secret objects
  - data:
    - key: secretvalue                           # data field to populate
      objectName: TestSecret                        # name of the mounted content to sync; this could be the object name or the object alias
    secretName: foosecret                     # name of the Kubernetes secret object
    type: Opaque     
EOF
```

### pod with secret as mount and Volume
``` bash
# Create a pod to mount the secret
cat <<EOF | kubectl apply -f -
kind: Pod
apiVersion: v1
metadata:
  name: secrets-store-inline-wi-sync
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: "kvcsi-demo-sa"
  containers:
    - name: ubuntu
      image: ubuntu:20.04
      command: [ "/bin/bash", "-c", "--" ]
      args: [ "while true; do sleep 30; done;" ]
      volumeMounts:
      - name: secrets-store01-inline
        mountPath: "/mnt/secrets-store"
        readOnly: true
      env:
      - name: SECRET_DATA
        valueFrom:
          secretKeyRef:
            name: foosecret
            key: secretvalue        
  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-kvcsi-wi-sync"
EOF
```

### Test secret
``` bash
# Check that the secret was properly mounted as a volume
kubectl exec -it secrets-store-inline-wi-sync -- cat /mnt/secrets-store/TestSecret

# Check that the Kubernetes Secret was created
kubectl get secret foosecret -o jsonpath='{.data.secretvalue}'|base64 --decode

# Check that the secret was properly mounted from the kubernetes secret as an enviornment variable
kubectl exec -it secrets-store-inline-wi-sync -- /bin/bash -c 'echo $SECRET_DATA'
```
