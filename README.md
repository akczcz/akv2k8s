# Implementation of akv2k8s
Demo implementation of akv2k8s in AKS

# Prerequisities
You need such prerequisities:
- Azure Subscription
- Laptop with CLI (Windows with Powershell, Linux, Mac, whatever) or use Azure Cloud shell, if you need install Azure CLI use https://docs.microsoft.com/en-us/cli/azure/install-azure-cli

Let's use Azure CLI for now.

# Preparation tasks - deployment infrastructure in Azure Resource Group

Create resource group
``` azcli
GROUP_NAME=rg-aks-test
LOCATION=northeurope
az group create \
    --location $LOCATION \
    --resource-group $GROUP_NAME
```

Create resource group in WE
``` azcli
GROUP_NAME_WE=rg-aks-test-we
LOCATION_WE=westeurope
az group create \
    --location $LOCATION_WE \
    --resource-group $GROUP_NAME_WE
```


## Deployment of ACR

Deploy Azure Container Registry (ACR).
The name of ACR must be globally unique, so exchange the name with your own
``` azcli
# Exchange the name aktestaksacr with your value!
ACR_NAME=acr-aktestaks
az acr create \
    --resource-group $GROUP_NAME \
    --name $ACR_NAME \
    --sku Basic
```
## Deployment of Azure KeyVault

## Deployment of User Assigned Managed Identities for AKS
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

# Deploy Azure KeyVault (AKV) with integration to Azure RBAC
Create Azure KeyVault instance
``` azcli
KV_NAME=kv-aktestaks
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
Add role assignment of Azure Key Vault Secret Officer to user identity
``` azcli
az role assignment create \
    --assignee $USER_NAME_OBJECTID \
    --role "Key Vault Secrets Officer" \
    --scope $KV_ID
```

## Deployment of AKS

Deploy Azure Kubernetes Services (AKS) with deployment options connected to resources created with previous steps
``` azcli
AKS_NAME=aks-aktest
az aks create \
    --resource-group $GROUP_NAME_WE \
    --attach-acr $ACR_NAME \
    --enable-managed-identity \
    --assign-identity $UAMI_ID \
    --assign-kubelet-identity $UAMI_KUBELET_ID \
    --node-count 2 \
    --generate-ssh-keys \
    --name $AKS_NAME

echo $UAMI_ID
echo $UAMI_KUBELET_ID
```

