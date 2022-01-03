# Implementation of akv2k8s in Azure Kubernetes Services (AKS)
This is LAB implementation of akv2k8s in AKS.
LAB uses imperative way of deployment (Azure CLI commands in this case).
You will use probably declarative way of deployment resources in your environment, in a meaning of definition of service instance using Azure Resource Manager (ARM) templates, Bicep, or Terraform type of deployment.
Moreover, you will probably orchestrate such declarative way of deployment in your Continuous Integration/Continuous Deployment pipelines (Using Azure DevOps, or Github, or any other tools).

# Prerequisities
You need such prerequisities:
- Azure Subscription (RBAC assignemnt of owner resource group you will be creating)
- Laptop with CLI (Windows with Powershell, Linux, Mac, whatever) or use Azure Cloud shell, if you need install Azure CLI use https://docs.microsoft.com/en-us/cli/azure/install-azure-cli
- Azure CLI must have at least version "2.31.0", use "az upgrade" command to upgrade to latest version, it is recommended to use extention "aks-preview" with minal version "0.5.49"

Let's use Azure CLI for now.

## Preparation tasks - deployment infrastructure in Azure Resource Group

Create resource group
``` azcli
GROUP_NAME=rg-aks-test
LOCATION=northeurope
az group create \
    --location $LOCATION \
    --resource-group $GROUP_NAME
```

### Deployment of ACR

Deploy Azure Container Registry (ACR).
The name of ACR must be globally unique, so exchange the name with your own
``` azcli
# Exchange the name aktestaksacr with your value!
ACR_NAME=acraktestaks
az acr create \
    --resource-group $GROUP_NAME \
    --name $ACR_NAME \
    --sku Basic
```
### Deployment of User Assigned Managed Identities for AKS
Deploy Azure User Assigned Managed Identity
``` azcli
UAMI_NAME=uami-aks
az identity create \
    --resource-group $GROUP_NAME \
    --name $UAMI_NAME
```
Export identity id value into variable
``` azcli
UAMI_ID=$(az identity show \
    --resource-group $GROUP_NAME \
    --query "id" \
    --name $UAMI_NAME \
    -o tsv)
```
Deploy Azure User Assigned Managed Identity for Kubelet
``` azcli
UAMI_KUBELET_NAME=uami-aks-kubelet
az identity create \
    --resource-group $GROUP_NAME \
    --name $UAMI_KUBELET_NAME
```
Export kubelet identity id value into variable
``` azcli
UAMI_KUBELET_ID=$(az identity show \
    --resource-group $GROUP_NAME \
    --query "id" \
    --name $UAMI_KUBELET_NAME \
    -o tsv)
```

### Deploy Azure KeyVault (AKV) with integration to Azure RBAC
Create Azure KeyVault instance
``` azcli
KV_NAME=kv-testaks
az keyvault create \
    --location $LOCATION \
    --resource-group $GROUP_NAME \
    --enable-rbac-authorization true \
    --name $KV_NAME
```
Export KeyVault id value into variable
``` azcli
KV_ID=$(az keyvault show \
    --resource-group $GROUP_NAME \
    --query "id" \
    --name $KV_NAME \
    -o tsv)
```
Export principalId of User Assigned Managed Identity of Kubelet (it will be accessing Azure KeyVault for secrets):
``` azcli
UAMI_KUBELET_PRINCIPALID=$(az identity show \
    --resource-group $GROUP_NAME \
    --query "principalId" \
    --name $UAMI_KUBELET_NAME \
    -o tsv)
```

And add accesss rights for Secrets in Azure KeyVault.
Create such role assignment for Managed Identity, which will AKS use. 
Add built in role Azure Key Vault Secret user "https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#key-vault-secrets-user"
Vault
``` azcli
az role assignment create \
    --assignee $UAMI_KUBELET_PRINCIPALID \
    --role "Key Vault Secrets User" \
    --scope $KV_ID
```
It is good idea, to also add RBAC of role Azure Key Vault Secret Officer to any object ID of user/person/identity, which will be responsible for secret managing in Azure Key.
Now let's export environment variable of objectId actual signed in user:
``` azcli
USER_NAME_OBJECTID=$(az ad signed-in-user show \
    --query "objectId" \
    -o tsv)
```
Add built in role assignment of Azure Key Vault Secret Officer https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#key-vault-secrets-officer to user identity
``` azcli
az role assignment create \
    --assignee $USER_NAME_OBJECTID \
    --role "Key Vault Secrets Officer" \
    --scope $KV_ID
```

### Deployment of additional RBAC assignmnet

Map need access rights for managed identities, AKS cluster's User Assigned Managed Identity has to have access rights to AKS's clusters Kubelet User Assigned Managed Identity.
Export AKS cluster's User Assigned Managed Identity principalId as environment variable
``` azcli
UAMI_ID_PRINCIPALID=$(az identity show \
    --resource-group $GROUP_NAME \
    --query "principalId" \
    --name $UAMI_NAME \
    -o tsv)
```
Create needed role assignment
``` azcli
az role assignment create \
    --assignee $UAMI_ID_PRINCIPALID \
    --role "Owner" \
    --scope $UAMI_KUBELET_ID
```

### Deployment of AKS

Deploy Azure Kubernetes Services (AKS) with deployment options connected to resources created with previous steps
``` azcli
AKS_NAME=aks-aktest
az aks create \
    --resource-group $GROUP_NAME \
    --attach-acr $ACR_NAME \
    --enable-managed-identity \
    --assign-identity $UAMI_ID \
    --assign-kubelet-identity $UAMI_KUBELET_ID \
    --node-count 2 \
    --generate-ssh-keys \
    --name $AKS_NAME
```
Get access to AKS cluster
``` azcli
# Get cluster credentials
az aks get-credentials \
    --resource-group $GROUP_NAME \
    --name $AKS_NAME
```
Try cluster level access using "kubectl" utility
``` azcli
kubectl get nodes
```
You should obtain result similar to:
``` 
kolarik@Azure:~$ kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-41796624-vmss000000   Ready    agent   15m   v1.21.7
aks-nodepool1-41796624-vmss000001   Ready    agent   15m   v1.21.7
```


## Deployment of akv2k8s controller

### Deploy kubernetes namespace
``` azcli
NAMESPACE_NAME=akv2k8s
kubectl create namespace $NAMESPACE_NAME --dry-run=client -o yaml | kubectl apply -f -
```
### Deploy akv2k8s controller using helm chart
Add "spv-charts" repository
``` azcli
helm repo add spv-charts https://charts.spvapi.no
```
you should get output like this:
```
kolarik@Azure:~$ helm repo add spv-charts https://charts.spvapi.no
"spv-charts" has been added to your repositories
```
Update local cache
```
helm repo update
```
you should get output like this:
```
kolarik@Azure:~$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "spv-charts" chart repository
Update Complete. ⎈Happy Helming!⎈
```
Run "helm upgrade" with parameters
```
helm upgrade -i \
    akv2k8s \
    spv-charts/akv2k8s \
    --namespace $NAMESPACE_NAME
```
you should get output like this:
```
kolarik@Azure:~$ helm upgrade -i \
>     akv2k8s \
>     spv-charts/akv2k8s \
>     --namespace $NAMESPACE_NAME
Release "akv2k8s" does not exist. Installing it now.
W0103 11:43:23.082623     167 warnings.go:67] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
W0103 11:43:23.563168     167 warnings.go:67] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
NAME: akv2k8s
LAST DEPLOYED: Mon Jan  3 11:43:20 2022
NAMESPACE: akv2k8s
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Congratulations! You've successfully installed Azure Key Vault to Kubernetes.
For more information see the documentation at https://akv2k8s.io.
```
If you come to this point, you sucessfully deployed needed infrastructure including AKS and deployed controller akv2k8s inside kubernetes cluster

Now you can see, that there are is new type of API resources, try to run:
```
kubectl -n akv2k8s api-resources
```
and you will see in your output also akvs object type:
![image](/img/akv-API.PNG)

### Deploy test resources into Azure KeyVault
We will deploy test secrets into 
``` azcli
KV_NAME=kv-testaks
SECRET_NAME1="secretname1"
SECRET_VALUE1="TopSecretPassword"
az keyvault secret set \
    --vault-name $KV_NAME \
    --name $SECRET_NAME1 \
    --value $SECRET_VALUE1

SECRET_SQL="sqlconnectionstring"
SECRET_SQL_VALUE="TopSecretConnectionString"
az keyvault secret set \
    --vault-name $KV_NAME \
    --name $SECRET_SQL \
    --value $SECRET_SQL_VALUE
```
you should get output like this:
```
kolarik@Azure:~$ az keyvault secret set \
>     --vault-name $KV_NAME \
>     --name $SECRET_SQL \
>     --value $SECRET_SQL_VALUE
{
  "attributes": {
    "created": "2022-01-03T11:57:31+00:00",
    "enabled": true,
    "expires": null,
    "notBefore": null,
    "recoveryLevel": "Recoverable+Purgeable",
    "updated": "2022-01-03T11:57:31+00:00"
  },
  "contentType": null,
  "id": "https://kv-testaks.vault.azure.net/secrets/sqlconnectionstring/81d0db1307044c5a83da9fbeb2d115cf",
  "kid": null,
  "managed": null,
  "name": "sqlconnectionstring",
  "tags": {
    "file-encoding": "utf-8"
  },
  "value": "TopSecretConnectionString"
}
```
I let whole id in the output to let you see whole id containing name of the secret (/sqlconnectionstring/) and its version at the end (/81d0db1307044c5a83da9fbeb2d115cf).

You can list created secrets:
```
kolarik@Azure:~$ az keyvault secret list --vault-name $KV_NAME -o table
Name                 Id                                                              ContentType    Enabled    Expires
-------------------  --------------------------------------------------------------  -------------  ---------  ---------
secretname1          https://kv-testaks.vault.azure.net/secrets/secretname1                         True
sqlconnectionstring  https://kv-testaks.vault.azure.net/secrets/sqlconnectionstring                 True
```
or in GUI of Azure KeyVault instance.

### Deploy kuberentes mapping objects
Let's create namespace for application, we will put our synchronized secrets to that namespace, because, we will reference them to the application in the same namespace
``` azcli
NAMESPACE_NAME_APP=app
kubectl create namespace $NAMESPACE_NAME_APP --dry-run=client -o yaml | kubectl apply -f -
```
you should get output like this:
```
kolarik@Azure:~$ kubectl create namespace $NAMESPACE_NAME_APP --dry-run=client -o yaml | kubectl apply -f -
namespace/app created
```
Now lets look at the file /manifests/akv.yaml, there are prepared 2 objects in declarative way.
```
DIR_NAME=manifests
mkdir $DIR_NAME
cd $DIR_NAME
vi akv.yaml #and put content of file bellow
```
Change the values in yaml file to your values, focus on the fields marked with description with numbers and exchange such fields to your values
``` yaml
# secrets for Persistent storage
apiVersion: spv.no/v2beta1
kind: AzureKeyVaultSecret
metadata:
  name: akvs-secret-name1 # 1. name of akvs object
  namespace: app
spec:
  vault:
    name: kv-testaks # 2. name of key vault
    object:
      name: SECRET-NAME1 # 3. name of the akv object
      type: secret # 4. akv object type
  output: 
    secret: 
      name: akv-secret-name1 # 5. kubernetes secret name
      dataKey: secretname1 # 6. key to store object value in kubernetes apiVersion: v1
---
apiVersion: spv.no/v2beta1
kind: AzureKeyVaultSecret # 1. name of akvs object
metadata:
  name: akvs-secret-sql
  namespace: app
spec:
  vault:
    name: kv-testaks # 2. name of key vault
    object:
      name: SECRET-SQL # 3. name of the akv object
      type: secret # 4. akv object type
  output: 
    secret: 
      name: akv-sql # 5. kubernetes secret name
      dataKey: sqlconnectionstring # 6. key to store object value in kubernetes apiVersion: v1

```
Then finaly save the manifest file:
![image](/img/akv-yaml.PNG)

Now let's deploy such manifests inside kubernetes:
``` azcli
kubectl apply -f ~/manifests/akv.yaml
```
you should get output like this:
```
kolarik@Azure:~/manifests$ kubectl apply -f ~/manifests/akv.yaml
azurekeyvaultsecret.spv.no/akvs-secret-name1 created
azurekeyvaultsecret.spv.no/akvs-secret-sql created
```
### Deploy test application and reference secrets

