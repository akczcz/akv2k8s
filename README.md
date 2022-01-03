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
Get identity value into variable
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
Get identity value into variable
``` azcli
UAMI_KUBELET_ID=$(az identity show \
    --resource-group $GROUP_NAME \
    --query "id" \
    --name $UAMI_KUBELET_NAME \
    -o tsv)
```

# Exchange the name aktestaksacr with your value!

Deploy Azure KeyVault (AKV) with integration to Azure RBAC
``` azcli
KV_NAME=kv-aktestaks
az keyvault create \
    --location $LOCATION \
    --resource-group $GROUP_NAME \
    --enable-rbac-authorization true \
    --name $KV_NAME
```
And add accesss rights for to KeyVault
``` azcli
az keyvault create \
    --location $LOCATION \
    --resource-group $GROUP_NAME \
    --enable-rbac-authorization true \
    --name $KV_NAME
```

## Deployment of AKS

Deploy Azure Kubernetes Services (AKS) with deployment options connected to resources created with previous steps
``` azcli
AKS_NAME=aktestaks
az aks create \
    --resource-group $GROUP_NAME \
    --attach-acr $ACR_NAME \
    --assign-identity $UAMI_ID \    
    --assign-kubelet-identity $UAMI_KUBELET_ID \
    --node-count 2 \
    --generate-ssh-keys \
    --name $AKS_NAME 
```

