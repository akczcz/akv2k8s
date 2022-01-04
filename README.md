![image](/img/thumbnail.jfif)
# Implementation of akv2k8s in Azure Kubernetes Services (AKS)
This is LAB implementation of project akv2k8s (see https://akv2k8s.io/ for documentation) in Azure Kuberenetes Services / AKS (see https://docs.microsoft.com/en-us/azure/aks/ for documentation).  
LAB uses imperative way of deployment (Azure CLI commands in this case).  
You will use probably declarative way of deployment resources in your environment, in a meaning of definition of service instance using of Azure Resource Manager (ARM) templates, Bicep, or Terraform type of deployment or any other declarative infrastructure description.  
Moreover, you will probably orchestrate such declarative way of deployment in your Continuous Integration/Continuous Deployment pipelines (Using Azure DevOps, or Github, or any other tools).  
Infrastructure described bellow does not contain other security related topics, which should be beared in mind and implemented according to security baselines and needed level of security by your project (it can be e.g. using private endpoints, private cluster deployment, encryption at rest, more advanced integration to AAD, etc.).   
You should also check the source of akv2k8s project on its github pages (https://github.com/SparebankenVest/akv2k8s-website/blob/master/source/content/index.mdx) and thing about all aspects of this OpenSource project before using it in your deployment.

# Project's Goal
Goal of this project is:
- Test akv2k8s project implementation with Azure Kubernetes Services
- Test the functionality of replication secrets between Azure KeyVault and Azure Kubernetes Services (synchronization of secrets into kubernetes secrets, which are persistently stored in etcd database)
- Test the functionality with providing such synced secret as pod's environment variable
- Deployment of infastructure according to the design bellow:  
![image](/img/akv2k8s.drawio.png)


# Project's Out of scope
- Declarative way of infrastructure deployment (which should be done as DevOps principles are part of Cloud Native architecture)
- Using Continuous Integration/ Continuous Deployment (CI/CD) pipelines (as such is best practice to use in Cloud Native architecture)
- Mapping of secrets by akv2k8s to application pods directelly (using sidecar container) without syncing to kubernetes secrets (as it can be another way of deployment akv2k8s project, see akv2k8s documentation for details)

# Prerequisities
You need such prerequisities:
- Azure Subscription (RBAC assignment of owner resource group you will be creating)
- Laptop with CLI (Windows with Powershell, Linux, Mac, whatever) or use Azure Cloud shell, if you need install Azure CLI use https://docs.microsoft.com/en-us/cli/azure/install-azure-cli
- Azure CLI must have at least version "2.31.0", use "az upgrade" command to upgrade to latest version, it is recommended to use extention "aks-preview" with minal version "0.5.49"

I will use Azure CLI for this LAB.

## Preparation tasks - infrastructure deployment in Azure Resource Group

Create resource group for deployment any other resources in your subscription, change group name in GROUP_NAME variable and change location in LOCATION variable definition (see available regions at: https://azure.microsoft.com/en-us/global-infrastructure/geographies/#overview):
``` azcli
GROUP_NAME=rg-aks-test
LOCATION=northeurope
az group create \
    --location $LOCATION \
    --resource-group $GROUP_NAME
```

### Deployment of Azure Container Registry (ACR)

Deploy Azure Container Registry (ACR).
The name of ACR must be globally unique, so exchange the name of ACR defined in ACR_NAME variable with your own value:
``` azcli
# change the name acraktestaks with your value, it must be globally unique!
ACR_NAME=acraktestaks
az acr create \
    --resource-group $GROUP_NAME \
    --name $ACR_NAME \
    --sku Basic
```
### Deployment of User Assigned Managed Identities for AKS

Deploy Azure User Assigned Managed Identity of AKS cluster, change UAMI_NAME variable with your value:  
``` azcli
UAMI_NAME=uami-aks
az identity create \
    --resource-group $GROUP_NAME \
    --name $UAMI_NAME
```
Export managed identity's id value into new variable UAMI_ID (will be referenced later):
``` azcli
UAMI_ID=$(az identity show \
    --resource-group $GROUP_NAME \
    --query "id" \
    --name $UAMI_NAME \
    -o tsv)
```
Deploy Azure User Assigned Managed Identity of AKS cluster Kubelet, change UAMI_KUBELET_NAME variable with your value:
``` azcli
UAMI_KUBELET_NAME=uami-aks-kubelet
az identity create \
    --resource-group $GROUP_NAME \
    --name $UAMI_KUBELET_NAME
```
Export kubelet's managed identity id value into variable UAMI_KUBELET_ID (will be referenced later):
``` azcli
UAMI_KUBELET_ID=$(az identity show \
    --resource-group $GROUP_NAME \
    --query "id" \
    --name $UAMI_KUBELET_NAME \
    -o tsv)
```

### Deploy Azure KeyVault (AKV) with integration to Azure RBAC
Create Azure KeyVault instance, change KV_NAME variable value with your value, it must be globally unique:
``` azcli
# change the name kv-testaks with your value, it must be globally unique!
KV_NAME=kv-tstaks
az keyvault create \
    --location $LOCATION \
    --resource-group $GROUP_NAME \
    --enable-rbac-authorization true \
    --name $KV_NAME
```

Export KeyVault's id value into variable KV_ID:
``` azcli
KV_ID=$(az keyvault show \
    --resource-group $GROUP_NAME \
    --query "id" \
    --name $KV_NAME \
    -o tsv)
```

Export of Kubelet's User Assigned Managed Identity principalId into variable UAMI_KUBELET_PRINCIPALID (it will be accessing Azure KeyVault for secrets):
``` azcli
UAMI_KUBELET_PRINCIPALID=$(az identity show \
    --resource-group $GROUP_NAME \
    --query "principalId" \
    --name $UAMI_KUBELET_NAME \
    -o tsv)
```

And add accesss rights for Secrets in Azure KeyVault.
Create such role assignment for Managed Identity, which will AKS use. 
Add built in role Azure Key Vault Secret user "https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#key-vault-secrets-user":
``` azcli
az role assignment create \
    --assignee $UAMI_KUBELET_PRINCIPALID \
    --role "Key Vault Secrets User" \
    --scope $KV_ID
```

It is also good idea, to add RBAC of role Azure Key Vault Secret Officer to any object ID of user/person/identity, which will be responsible for secret management in Azure KeyVault.
Let's export environment variable objectId actually signed user into variable USER_NAME_OBJECTID:
``` azcli
USER_NAME_OBJECTID=$(az ad signed-in-user show \
    --query "objectId" \
    -o tsv)
```

Add built-in role assignment of Azure Key Vault Secret Officer https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#key-vault-secrets-officer to user identity:
``` azcli
az role assignment create \
    --assignee $USER_NAME_OBJECTID \
    --role "Key Vault Secrets Officer" \
    --scope $KV_ID
```

### Deployment of additional RBAC assignmnet

Map needed access rights for managed identities, AKS cluster's User Assigned Managed Identity has to have access rights to AKS's clusters Kubelet User Assigned Managed Identity.

Export AKS cluster's User Assigned Managed Identity principalId as environment variable UAMI_ID_PRINCIPALID:
``` azcli
UAMI_ID_PRINCIPALID=$(az identity show \
    --resource-group $GROUP_NAME \
    --query "principalId" \
    --name $UAMI_NAME \
    -o tsv)
```

Create needed role assignment:
``` azcli
az role assignment create \
    --assignee $UAMI_ID_PRINCIPALID \
    --role "Owner" \
    --scope $UAMI_KUBELET_ID
```

### Deployment of Azure Kubernetes Services (AKS)

Deploy Azure Kubernetes Services (AKS) with deployment options connected to resources created with previous steps, change AKS_NAME environment value, as it must be globally unique:
``` azcli
# change the name aks-aktest with your value, it must be globally unique!
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
If you got errors with syntax statements, then update az cli using ```az update``` command, see requirements at the beginning of this LAB.

Get access to AKS cluster for yourself:
``` azcli
# Get cluster credentials
az aks get-credentials \
    --resource-group $GROUP_NAME \
    --name $AKS_NAME
```

Try cluster level access using "kubectl" utility:
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
Deploy kubernetes namespace:
``` azcli
NAMESPACE_NAME=akv2k8s
kubectl create namespace $NAMESPACE_NAME --dry-run=client -o yaml | kubectl apply -f -
```

You should get output like this:
```
namespace/akv2k8s created
```
### Deploy akv2k8s controller using helm chart
Now, we will deploy akv2k8s controller using helm chart's to our kubernetes namespace.
Add "spv-charts" repository:
``` azcli
helm repo add spv-charts https://charts.spvapi.no
```

you should get output like this:
```
"spv-charts" has been added to your repositories
```

Next, update local helm repository cache:
```
helm repo update
```

You should get output like this:
```
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "spv-charts" chart repository
Update Complete. ⎈Happy Helming!⎈
```

Next, run "helm upgrade" with parameters:
```
helm upgrade -i \
    akv2k8s \
    spv-charts/akv2k8s \
    --namespace $NAMESPACE_NAME
```

You should get output like this:
```
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

If you come to this point, you sucessfully deployed needed infrastructure, including AKS and controller akv2k8s inside kubernetes cluster.

Now, you can see, that there is new type of API resources available, try to run:
```
kubectl -n akv2k8s api-resources
```
and you will see in your output also akvs object type:
![image](/img/akv-API.PNG)

Let's see what type of object does deployment did:
```
kubectl -n akv2k8s get all
```

You should get output similar to:
```
NAME                                       READY   STATUS    RESTARTS   AGE
pod/akv2k8s-controller-d4ff9564c-qc5vk     1/1     Running   0          109m
pod/akv2k8s-envinjector-69f4d6bcb9-2ltcj   1/1     Running   0          109m
pod/akv2k8s-envinjector-69f4d6bcb9-mpp4f   1/1     Running   0          109m

NAME                          TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                   AGE
service/akv2k8s-controller    ClusterIP   10.0.92.173   <none>        9000/TCP                  109m
service/akv2k8s-envinjector   ClusterIP   10.0.80.235   <none>        443/TCP,80/TCP,9443/TCP   109m

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/akv2k8s-controller    1/1     1            1           109m
deployment.apps/akv2k8s-envinjector   2/2     2            2           109m

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/akv2k8s-controller-d4ff9564c     1         1         1       109m
replicaset.apps/akv2k8s-envinjector-69f4d6bcb9   2         2         2       109m
```

You can look at the akv2k8s's controller logs by typing command bellow, exchange the name of pod akv2k8s-controller-d4ff9564c-qc5vk with your real name of controller pod:
```
kubectl -n akv2k8s logs akv2k8s-controller-d4ff9564c-qc5vk
```

You should see output similar to:
```
I0104 09:44:44.261444       1 main.go:96] "log settings" format="text" level="2"
I0104 09:44:44.261502       1 version.go:31] "version info" version="1.3.0" commit="a375982" buildDate="2021-08-06T06:52:07Z" component="controller"
W0104 09:44:44.261605       1 client_config.go:615] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I0104 09:44:44.263070       1 main.go:161] "Creating event broadcaster"
I0104 09:44:44.836258       1 controller.go:167] "setting up event handlers"
I0104 09:44:44.836291       1 controller.go:178] "starting azurekeyvaultsecret controller"
I0104 09:44:45.336466       1 controller.go:196] "starting azure key vault secret queue"
I0104 09:44:45.336506       1 controller.go:199] "starting azure key vault deleted secret queue"
I0104 09:44:45.336516       1 controller.go:202] "starting azure key vault queue"
I0104 09:44:45.336525       1 controller.go:205] "started workers"
```
!!! Remember how to see controller logs, you will mostly propably use such logs in case of any troubleshooting, you will see syncing information with secrets in this logs, including possible troubles !!!

### Deploy test secrets into Azure KeyVault
We will deploy test secrets into Azure KeyVault, you can change the variables and its values any way, but then you should update scripts bellow and also mapping kubernetes objects defined later in this LAB: 
``` azcli
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

You should get output's like this (for each secret definition):
```
{
  "attributes": {
    "created": "2022-01-04T09:58:02+00:00",
    "enabled": true,
    "expires": null,
    "notBefore": null,
    "recoveryLevel": "Recoverable+Purgeable",
    "updated": "2022-01-04T09:58:02+00:00"
  },
  "contentType": null,
  "id": "https://kv-tstaks.vault.azure.net/secrets/sqlconnectionstring/80a419159fb94611a91f616295bd33bf",
  "kid": null,
  "managed": null,
  "name": "sqlconnectionstring",
  "tags": {
    "file-encoding": "utf-8"
  },
  "value": "TopSecretConnectionString"
}
```
I let whole id in the output, to let you see whole id containing name of the secret (/sqlconnectionstring/) and its version at the end (/80a419159fb94611a91f616295bd33bf), version will be changing in future editing of secret and akv2k8s controller in kubernetes will be syncing such new versions inside kubernetes secrets.

You can list created secrets by typing:
```
az keyvault secret list --vault-name $KV_NAME -o table
```

You should get output similar to:
```
Name                 Id                                                             ContentType    Enabled    Expires
-------------------  -------------------------------------------------------------  -------------  ---------  ---------
secretname1          https://kv-tstaks.vault.azure.net/secrets/secretname1                         True
sqlconnectionstring  https://kv-tstaks.vault.azure.net/secrets/sqlconnectionstring                 True
```
or you can list secrets in GUI of Azure KeyVault instance.

### Deploy kubernetes mapping objects (akvs)
Let's create namespace for application, we will put our synchronized secrets to that namespace, because, we will reference them to the application in the same namespace az an application:
``` azcli
NAMESPACE_NAME_APP=app
kubectl create namespace $NAMESPACE_NAME_APP --dry-run=client -o yaml | kubectl apply -f -
```

You should get output like this:
```
namespace/app created
```

Now lets look at the file /manifests/akv.yaml, there are prepared 2 objects in declarative way, you can also clone this repo, to safe your time, if you want.
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
    name: kv-tstaks # 2. name of key vault, change IT with your value!
    object:
      name: secretname1 # 3. name of the akv object
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
    name: kv-tstaks # 2. name of key vault, change IT with your value!
    object:
      name: sqlconnectionstring # 3. name of the akv object
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

You should get output like this:
```
azurekeyvaultsecret.spv.no/akvs-secret-name1 created
azurekeyvaultsecret.spv.no/akvs-secret-sql created
```

If there is some trouble, you will not see created secrets in synced state: 
```
kubectl -n app get akvs
```

If there is some trouble, you will get:
```
NAME                VAULT        VAULT OBJECT   SECRET NAME   SYNCHED   AGE
akvs-secret-name1   kv-testaks   SECRET-NAME1                           20m
akvs-secret-sql     kv-testaks   SECRET-SQL                             20m
```

If your secrets are not synced correctly, go to the akv2k8s controller log ans see whats wrong (access rights, networking, referencing, etc).  
If there is everything well done, you should see output similar to:
```
NAME                VAULT        VAULT OBJECT          SECRET NAME        SYNCHED   AGE
akvs-secret-name1   kv-testaks   secretname1           akv-secret-name1   5s        5m44s
akvs-secret-sql     kv-testaks   sqlconnectionstring   akv-sql            5s        5m44s
```

You should also check created secrets, type:
```
kubectl -n app get secrets
```

If there is trouble, you will se only default kubernetes secret without newly created ones:
```
NAME                  TYPE                                  DATA   AGE
default-token-4fjp5   kubernetes.io/service-account-token   3      38m
```

If your akvs objects are in synced state, you will see newly created kubernetes secrets: 
```
NAME                  TYPE                                  DATA   AGE
akv-secret-name1      Opaque                                1      96s
akv-sql               Opaque                                1      96s
default-token-4fjp5   kubernetes.io/service-account-token   3      72m
```

Let's try to describe secret to see, whether there is some content:
```
kubectl -n app describe secret akv-secret-name1
```

You should see output similar to:
```
kolarik@Azure:~$ kubectl -n app describe secret akv-secret-name1
Name:         akv-secret-name1
Namespace:    app
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
secretname1:  17 bytes
```

As you may know, secrets inside kubernetes are stored in ETCD database, and they are not encrypted, they are encoded with base64 algorithm.
So, let's try to decode data value from secret and see its value:
``` azcli
kubectl -n app get secrets/akv-secret-name1 --template={{.data.secretname1}} | base64 -d
```

You will see decoded secret value :)

### Deploy test application and reference secrets

Now let's try to deploy easy application and reference such secrets from an application, to test whole concept.
```
# Go to the manigests folder if you did not so far
cd ~/manifests
kubectl create deployment nginx --image=nginx --dry-run=client --namespace=app -o yaml > nginx.yaml
```
Command will use dry run to create nginx.yaml deployment file and prepare it for deployment of nginx image from public registry of https://hub.docker.com, with latest tag, and name such deployment as "nginx" and put it into our "app" namespace, where we allready synced secrets from AzureKeyVault.

If you look at this file by cut command you should see something similar to:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

If you apply such deployment file, it will create deployment,replicaset and pods. Let's try it, run command in manifests directory:
```
cd ~/manifests
kubectl apply -f  nginx.yaml
```

Let's check what was created by running command:
```
kubectl -n app get deploy,pod,rs,service
```

You should see output similar to:
```
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           33s

NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-6799fc88d8-cgc6z   1/1     Running   0          33s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-6799fc88d8   1         1         1       33s
```
So, we created one deployment, one replicaset and one deployment.

Try to run command inside pod end look, whether there are any environment variables now, we will run ```printenv``` command inside container, do not forget to exchange pod name with your real one:
```
# change pod name nginx-6799fc88d8-cgc6z with your real one
kubectl -n app exec nginx-6799fc88d8-cgc6z it -- printenv
```

If we edit deployment file and apply file again (that is what DevOps pipelines should do after commit or pull request), then deployment will create new replicaset which will create new pods.

Let's edit deployment file and add references to its secrets:
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

