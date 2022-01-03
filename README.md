# Implementation of akv2k8s
Demo implementation of akv2k8s in AKS

# Prerequisities
You need such prerequisities:
- Azure Subscription
- Laptop with CLI (Windows with Powershell, Linux, Mac, whatever) or use Azure Cloud shell, if you need install Azure CLI use https://docs.microsoft.com/en-us/cli/azure/install-azure-cli

Let's use Azure CLI for now.

# Preparation tasks

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
# Exchane the name aktestaksacr with your value!
ACR_NAME=aktestaksacr
az acr create \
    --resource-group $GROUP_NAME \
    --name $ACR_NAME \
    --sku Basic
```

## Deployment of AKS

Deploy Azure Kubernetes Services (AKS)
``` azcli
AKS_NAME=aktestaks
az aks create \
    --resource-group $GROUP_NAME \
    --name $AKS_NAME \
    --node-count 2 \
    --generate-ssh-keys \
    --attach-acr $ACR_NAME
```

## Deployment of KV

Deploy Azure Kubernetes Services (AKS)
``` azcli

```
