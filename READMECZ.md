[EN](README.md) | [CZ](READMECZ.md)  
![image](/img/thumbnail.jfif)
# Implementace akv2k8s v Azure Kubernetes Services (AKS)
Tento LAB popisuje implementaci projektu akv2k8s (viz dokumentace https://akv2k8s.io/ ) v Azure Kuberenetes Services / AKS (viz dokumentace https://docs.microsoft.com/en-us/azure/aks/).  
LAB používá imperativní způsob deploymentu (Azure CLI příkazy v našem případě).   
Ve svých prostředích pravděpodobně použijete declarativní způsob deploymentu instancí služeb, tzn. definování infrastruktury jako kód (Infrastructure As A Code) prostřednictvím Azure Resource Manager (ARM), Bicep, nebo Terraform.  
Dále budete nejspíše orchestrovat takové declarativní definice ve svých Continuous Integration/Continuous Deployment pipelines (S použitím Azure DevOps, nebo Githubu, či jiného CI/CD nástroje).  
Infrastruktura popsaná níže neobsahuje jiné bezpečnostně orientované témata, která by měla být braná v úvahu a implementovaná s ohledem na základní bezpečnostní úroveň cloud poskytovatele a bezpečnostní úrovně dle požadavků ve vašem projektu (to může být např.: private endpoints, private cluster deployment, encryption at rest, pokročilejší integrace na AAD atp.).   
Taktéž byste vždy při používání open source projektů jako je akv2k8s měli navštívit github stránky projektu (https://github.com/SparebankenVest/akv2k8s-website/blob/master/source/content/index.mdx) a popřemýšlet o všech aspektech použití vámi vybraného  OpenSource projektu před tím, než ho využijete.

# Cíl projektu
Cílem tohoto projektu je:
- test akv2k8s projektu implementací do Azure Kubernetes Services
- test funkcionality replikace tzv "secrets" mezi instancemi služeb Azure KeyVault and Azure Kubernetes Services (synchronizace secrets do kubernetes secrets, které jsou trvale uložené v etcd databázi)  
- test funkcionality ve smyslu mapování synchronizovaných secrets jako proměnných do podu, ve kterém běží aplikační kontejner 
- deployment infastruktury dle tohoto designu:  
![image](/img/akv2k8s.drawio.png)


# Co není součástí projektu
- deklarativní způsob deploymentu (který by měl být použit společně s DevOps principy, jako součást Cloud Native architektury), v tomto projektu provádíme deployment deklarativním způsobem pouze u částí deploymentu do kubernetes
- využítí Using Continuous Integration/ Continuous Deployment (CI/CD) pipelines (taktéž jako best practice pro Cloud Native architekturu)
- mapování secrets projektem akv2k8s do aplikačního kontejneru napřímo (s použitím sidecar kontejneru) bez synchronizace do kubernetes secrets (jedná se o další způsob implementace projektu akv2k8s, pro více informací využijte dokumentaci projektu)  

# Předpoklady
Pro realizaci projektu je nezbytné:
- mít přístup k Azure subscribci (potřebujete mít oprávnění owner, k resource group kterou budeme vytvářet, tj v ideálním případě je mít oprávnění owner už na subscribci ve které budeme resource group vytvářet)
- Laptop s příkazovou řádkou (Windows with Powershell, Linux, Mac, či cokoliv jiného) nebo využít Azure Cloud shell, jestli potřebujete do svého CLI instalovat Azure CLI, využijte odkaz  https://docs.microsoft.com/en-us/cli/azure/install-azure-cli
- Azure CLI musí být ve verzi minimálně "2.31.0", použijte "az upgrade" příkaz pro povýšení Azure CLI na nejnovější verzi, zároveň je nutné mít nainstalované rozšíření Azure CLI "aks-preview" v minimální verzi "0.5.49"

Níže v LABu použijeme variantu Azure CLI.

## Přípravné práce - deployment Azure infrastruktury do Azure Resource Group

Vytvořte resource group do které budete následně provádět deployment dalších instancí služeb, změňte název resource group v proměnné GROUP_NAME a vámi požadovaný azure region v proměnné LOCATION (pro seznam všech azure regionů využijte dokumentaci: https://azure.microsoft.com/en-us/global-infrastructure/geographies/#overview):
``` azcli
GROUP_NAME=rg-aks-test
LOCATION=northeurope
az group create \
    --location $LOCATION \
    --resource-group $GROUP_NAME
```

### Deployment Azure Container Registry (ACR)

Vytvořte Azure Container Registry (ACR).
Název ACR musí být globálně unikátní, proto zaměňte název ACR v proměnné ACR_NAME za vámi vybraný název:
``` azcli
# change the name acraktestaks with your value, it must be globally unique!
ACR_NAME=acraktestaks
az acr create \
    --resource-group $GROUP_NAME \
    --name $ACR_NAME \
    --sku Basic
```
### Deployment User Assigned Managed Identities pro AKS

Vytvořte  Azure User Assigned Managed Identity pro AKS cluster, zaměňte hodnotu proměnné UAMI_NAME za vlasntí název:  
``` azcli
UAMI_NAME=uami-aks
az identity create \
    --resource-group $GROUP_NAME \
    --name $UAMI_NAME
```
Vyexportujte ID managed identity do nové proměnné UAMI_ID (bude použita později):
``` azcli
UAMI_ID=$(az identity show \
    --resource-group $GROUP_NAME \
    --query "id" \
    --name $UAMI_NAME \
    -o tsv)
```
Vytvořte Azure User Assigned Managed Identity pro AKS cluster resp. Kubelet, změňte hodnotu proměnné UAMI_KUBELET_NAME za vlastní název:
``` azcli
UAMI_KUBELET_NAME=uami-aks-kubelet
az identity create \
    --resource-group $GROUP_NAME \
    --name $UAMI_KUBELET_NAME
```
Vyexportujte ID managed identity Kubeletu do nové proměnné UAMI_KUBELET_ID (bude použita později):
``` azcli
UAMI_KUBELET_ID=$(az identity show \
    --resource-group $GROUP_NAME \
    --query "id" \
    --name $UAMI_KUBELET_NAME \
    -o tsv)
```

### Deployment Azure KeyVault (AKV) s integrací na Azure RBAC
Vytvořte Azure KeyVault instanci, změňte hodnotu proměnné KV_NAME za vlastní název, musí být globálně unikátní:
``` azcli
# change the name kv-testaks with your value, it must be globally unique!
KV_NAME=kv-tstaks
az keyvault create \
    --location $LOCATION \
    --resource-group $GROUP_NAME \
    --enable-rbac-authorization true \
    --name $KV_NAME
```

Vyexportujte ID KeyVaultu do nové proměnné KV_ID:
``` azcli
KV_ID=$(az keyvault show \
    --resource-group $GROUP_NAME \
    --query "id" \
    --name $KV_NAME \
    -o tsv)
```

Vyexportujte principalId User Assigned Managed Identity Kubeletu do nové proměnné UAMI_KUBELET_PRINCIPALID (bude použita Kubeletem pro přístup k Azure KeyVaultu, resp. k secrets v něm uloženým):
``` azcli
UAMI_KUBELET_PRINCIPALID=$(az identity show \
    --resource-group $GROUP_NAME \
    --query "principalId" \
    --name $UAMI_KUBELET_NAME \
    -o tsv)
```

A zároveň přidejte takové identitě přístupové oprávnění právě pro secrets daného Azure KeyVaultu.  
To provedete tak, že vytvoříte role assignment pro Managed Identitu, kterou AKS cluster používá. 
K tomu použijte built in roli která je popsána v dokumentaci "https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#key-vault-secrets-user":
``` azcli
az role assignment create \
    --assignee $UAMI_KUBELET_PRINCIPALID \
    --role "Key Vault Secrets User" \
    --scope $KV_ID
```

V tuto chvíli je taktéž dobrý nápad přidat AAD RBAC roli Azure Key Vault Secret Officer někomu člověku/týmu/identitě, kdo bude zodpovědný za životní cyklus secretů v Azure KeyVault.
Vytvořme tedy proměnnou, která vyexportuje objectId aktuálně přihlášeného uživatele do nové proměnné USER_NAME_OBJECTID:
``` azcli
USER_NAME_OBJECTID=$(az ad signed-in-user show \
    --query "objectId" \
    -o tsv)
```

A následně přidejme této proměnné built-in roli Azure Key Vault Secret Officer https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#key-vault-secrets-officer:
``` azcli
az role assignment create \
    --assignee $USER_NAME_OBJECTID \
    --role "Key Vault Secrets Officer" \
    --scope $KV_ID
```

### Deployment další nutné RBAC role

Přidejte další nezbytné oprávnění pro managed identities. Tj přidejte oprávnění pro User Assigned Managed Identity AKS clusteru, tak aby měla roli owner nad User Assigned Managed Identity Kubelet serveru (krok nutný pro úspěšnost následného deploymentu AKS clusteru)

Vyexportujte principalId User Assigned Managed Identity AKS clusteru jako novou proměnnou UAMI_ID_PRINCIPALID:  
``` azcli
UAMI_ID_PRINCIPALID=$(az identity show \
    --resource-group $GROUP_NAME \
    --query "principalId" \
    --name $UAMI_NAME \
    -o tsv)
```

Vytvořte nový role assignment:  
``` azcli
az role assignment create \
    --assignee $UAMI_ID_PRINCIPALID \
    --role "Owner" \
    --scope $UAMI_KUBELET_ID
```

### Deployment Azure Kubernetes Services (AKS)

Vytvořte Azure Kubernetes Services (AKS) s použitím níže uvedených parametrů (využíváme dříve exportované proměnné), změňte hodnotu proměnné AKS_NAME za vlastní název, musí být globálně unikátní:
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
V případě, že dostanete chybové hlášky s upozorněním na neznámé argumenty, tak aktualizujte az cli s použitím  ```az update``` příkazu, viz minimální požadavky v úvodní sekci labu.

Vygenerujte si .kubeconfig soubor abyste měli přístup do AKS clusteru:
``` azcli
# Get cluster credentials
az aks get-credentials \
    --resource-group $GROUP_NAME \
    --name $AKS_NAME
```

Vyzkoušejte přístup do AKS clusteru s použitím "kubectl" utility:
``` azcli
kubectl get nodes
```

Měli byste obdržet výstup podobný tomuto:
``` 
kolarik@Azure:~$ kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-41796624-vmss000000   Ready    agent   15m   v1.21.7
aks-nodepool1-41796624-vmss000001   Ready    agent   15m   v1.21.7
```

## Deployment akv2k8s controlleru

### Deploy kubernetes namespace
Vytvořte kubernetes namespace:
``` azcli
NAMESPACE_NAME=akv2k8s
kubectl create namespace $NAMESPACE_NAME --dry-run=client -o yaml | kubectl apply -f -
```

Měli byste obdržet výstup podobný tomuto:
```
namespace/akv2k8s created
```
### Deploy akv2k8s controlleru s použitím helm chartu
Nyní, vytvořte akv2k8s controller s použitím helm chartu do kubernetes namespace.
Přidejte "spv-charts" do lokálního repository:
``` azcli
helm repo add spv-charts https://charts.spvapi.no
```

Měli byste obdržet výstup podobný tomuto:
```
"spv-charts" has been added to your repositories
```

Nyní aktualizujte cache lokálního helm repository:
```
helm repo update
```

Měli byste obdržet výstup podobný tomuto:
```
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "spv-charts" chart repository
Update Complete. ⎈Happy Helming!⎈
```

Nyní nainstalujte controller s využitím "helm upgrade" a parametrů:
```
helm upgrade -i \
    akv2k8s \
    spv-charts/akv2k8s \
    --namespace $NAMESPACE_NAME
```

Měli byste obdržet výstup podobný tomuto:
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

Jestli jste došli až sem, tak jste úspěšně nainstalovali potřebnou infrastrukturu, včetně AKS clusteru a controlleru akv2k8s dovnitř kubernetes clusteru.

Nyní můžete vidět, že se nám API schéma aktualizovalo o nové API, zkuste pustit následující příkaz:
```
kubectl -n akv2k8s api-resources
```
a uvidíte, že je zde nový typ objektu "akvs":
![image](/img/akv-API.PNG)

Podívejme se co vše nám helm chart vytvořil:
```
kubectl -n akv2k8s get all
```

Měli byste obdržet výstup podobný tomuto:
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

Můžete se podívat na logy controlleru nově vytvořeného controlleru akv2k8s, zaměňte ale název podu akv2k8s-controller-d4ff9564c-qc5vk za tu vaši, kterou jste získali ve výstupu z předchozího příkazu:
```
kubectl -n akv2k8s logs akv2k8s-controller-d4ff9564c-qc5vk
```

Měli byste obdržet výstup podobný tomuto:
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
!!! Pamatujte si, jak se podíváte na logy controlleru, velmi pravděpodobně budete takové logy využívat v případě řešení jakýchkoliv obtíží, uvidíte zde informace ohledně synchronizace secrets, včetně případných potíží !!!

### Vytvoření testovacích secrets v Azure KeyVault
Vytvoříme testovací secrets v Azure KeyVaultu, můžete jakkoliv změnit hodnoty proměnných, jen si pak nezapomeňte aktualizovat konfigurační scripty v následujících sekcích, jelikož ty konfigurační scripty nejsou parametrizované a mapují se přímo na tyto hodnoty: 
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

Měli byste pro každý secret obdržet výstup podobný tomuto: 
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
Záměrně jsem zde nechal celý výstup, tak abyste viděli celé id secretu, včetně jeho názvu (/sqlconnectionstring/) a jeho verze  (/80a419159fb94611a91f616295bd33bf), verze se s každou další změnou mění a  akv2k8s controller v kubernetes synchronizuje právě nové verze do kubernetes secrets.  

Můžete si zobrazit všechny secrety z keyvaultu:
```
az keyvault secret list --vault-name $KV_NAME -o table
```

Měli byste obdržet výstup podobný tomuto:
```
Name                 Id                                                             ContentType    Enabled    Expires
-------------------  -------------------------------------------------------------  -------------  ---------  ---------
secretname1          https://kv-tstaks.vault.azure.net/secrets/secretname1                         True
sqlconnectionstring  https://kv-tstaks.vault.azure.net/secrets/sqlconnectionstring                 True
```
nebo si můžete zobrazit seznam secrets v GUI Azure KeyVault instance.

### Vytvoření akvs mapovacích objektů uvnitř kubernetes (akvs)
Neprve vytvořte namespace pro aplikaci, později v tomto namespace vytvoříme secrets, je nutné mít secrets ve stejném namespace jako aplikaci z důvodu budoucích referencí:
``` azcli
NAMESPACE_NAME_APP=app
kubectl create namespace $NAMESPACE_NAME_APP --dry-run=client -o yaml | kubectl apply -f -
```

Měli byste obdržet výstup podobný tomuto:
```
namespace/app created
```

Nyní se podívejte na soubor /manifests/akv.yaml, připravil jsem zde 2 objekty deklarativním způsobem, taktéž si můžete celé repository naklonovat do vašeho lokálního gitu:
```
DIR_NAME=manifests
mkdir $DIR_NAME
cd $DIR_NAME
vi akv.yaml #and put content of file bellow
```

!!! Změnte ale hodnoty v yaml souboru, tak aby se popis správně odkazoval na existující odkazy, všímejte si těch položek, za kterými je "#" a zkontrolujte validnost dané hodnoty:
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
Následně tento manifest soubor uložte:
![image](/img/akv-yaml.PNG)

A následně proveďte deployment manifest souboru do kubernetes clusteru:
``` azcli
kubectl apply -f ~/manifests/akv.yaml
```

Měli byste obdržet výstup podobný tomuto:
```
azurekeyvaultsecret.spv.no/akvs-secret-name1 created
azurekeyvaultsecret.spv.no/akvs-secret-sql created
```

Vypište se seznam objektů akvs (mapovací objekty na kuberentes secrets dle předchozího yaml souboru): 
```
kubectl -n app get akvs
```

Jestli jste udělali někde chybu, tak pravděpodobně nevidíte objekty v synchronizovaném stavu:
```
NAME                VAULT        VAULT OBJECT   SECRET NAME   SYNCHED   AGE
akvs-secret-name1   kv-testaks   SECRET-NAME1                           20m
akvs-secret-sql     kv-testaks   SECRET-SQL                             20m
```

Jestli objekty nejsou v synchronizovaném stavu, tak se běžte podívat na logy akv2k8s controlleru, tak jak jsme si již zkoušeli v předchozích krocích a podívejte se v logu co je chybně (mohou to být přístupové práva, sítě, reference atp.).  
Jestli jste vše udělali bez chyby, tak byste měli obdržet výstup podobný tomuto:
```
NAME                VAULT        VAULT OBJECT          SECRET NAME        SYNCHED   AGE
akvs-secret-name1   kv-testaks   secretname1           akv-secret-name1   5s        5m44s
akvs-secret-sql     kv-testaks   sqlconnectionstring   akv-sql            5s        5m44s
```

Zároveň byste měli zkontrolovat, zda-li bytvořeny nějaké objekty typu kubernetes secrets:
```
kubectl -n app get secrets
```

V případě potíží uvidíte pouze default servisní token secret:
```
NAME                  TYPE                                  DATA   AGE
default-token-4fjp5   kubernetes.io/service-account-token   3      38m
```

Jestli jsou vaše akvs objekty správně synchronizované, dostanete výstup podobný tomuto: 
```
NAME                  TYPE                                  DATA   AGE
akv-secret-name1      Opaque                                1      96s
akv-sql               Opaque                                1      96s
default-token-4fjp5   kubernetes.io/service-account-token   3      72m
```

Zkuste si následně popsat nějaký secret a podívat se do něj, zda-li něco obsahuje:
```
kubectl -n app describe secret akv-secret-name1
```

Měli byste obdržet výstup podobný tomuto:
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

Jak asi nejspíše víte, tak kubernetes secrets jsou ukládány do perzistentní vrstvy kubernetes, tj do databáze etcd.  
Ta není šifrovaná, položky v ní jsou encodované base64 algoritmem, to znamená, že v případě, že v daném namespace máme příslušené oprávnění, můžeme zkusit si daný obsah zobrazit a přes base64 utilitu dekódovat:
``` azcli
kubectl -n app get secrets/akv-secret-name1 --template={{.data.secretname1}} | base64 -d
```

Z výstupu bychom měli dostat dekódovaný obsah :)

### Vytvoření testovací aplikace a mapování objektů/secrets

Nyní si zkusme vytvořit jednoduchou testovací aplikaci a namapovat pro ni secrets ve formě proměnných, abychom otestovali celý koncept:
```
# Go to the manigests folder if you did not so far
cd ~/manifests
kubectl create deployment nginx --image=nginx --dry-run=client --namespace=app -o yaml > nginx.yaml
```
Přízak použije tzv. dry-run, tj s pomocí výstupu do yaml formátu a přesměrováním do souboru provede vytvoření základního yaml konfiguračního souboru, který definuje deployment typu deployment a použije k tomu image kontejneru nginx z veřejně přístupného registru https://hub.docker.com, který je označen tagem latest, pojmenuje takový deployment "nginx" and vytvoří ho do namespace který se jmenuje "app", kde jsou již synchronizované secrets, které později použijeme.

V případě že se na obsah souboru podáváte:
```
# Go to the manigests folder if you did not so far
cd ~/manifests
cat nginx.yaml
```

měli byste vidět obsah podobný tomuto výstupu:
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

Nyní aplikujte manifest soubor do kubernetes clusteru, vytvoříte tím deployment, replicaset a pod:
```
cd ~/manifests
kubectl apply -f  nginx.yaml
```

Zkontrolujte co jste vytvořili:
```
kubectl -n app get deploy,pod,rs,service
```

měli byste vidět obsah podobný tomuto výstupu:
```
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           33s

NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-6799fc88d8-cgc6z   1/1     Running   0          33s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-6799fc88d8   1         1         1       33s
```
Jak vidíte, vytvořili jsme 1 x deployment, 1 x replicaset a 1x deployment.

Nyní zkuste pustit příkaz uvnitř kontejneru, v podu, kde zkusíme vypsat pomocí příkazu ```printenv``` všechny proměnné, nezapomeňte zaměnit název podu za ten váš:
```
# change pod name nginx-6799fc88d8-cgc6z with your real one
kubectl -n app exec nginx-6799fc88d8-cgc6z it -- printenv
```

měli byste vidět obsah podobný tomuto výstupu:
```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=nginx-6799fc88d8-cgc6z
NGINX_VERSION=1.21.5
NJS_VERSION=0.7.1
PKG_RELEASE=1~bullseye
KUBERNETES_PORT=tcp://10.0.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.0.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.0.0.1
KUBERNETES_SERVICE_HOST=10.0.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
HOME=/root
```
Pamatujte si tento výstup (zkopírujte si ten obsah někam pro budoucí porovnání) ... zkusíme nyní do proměnných přidat hodnoty z kubernetes secrets což je cíl našeho labu.

Jestli zeditujeme opět yaml manifest soubor a znova jej zaplikujeme, tak nám vytvoří nový replicaset a nové pody. (v reálném projektu to však za vás dělají agenti které spouští vaše DevOps pipelines)

Zeditujme nyní manifest soubor a přidejme reference na secrets, pro snažší orientaci jsem položky označil poznámkou s textem "added new line", zároveň můžete smazat nepotřebné řádky jako jsou timespamps a status:
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
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
      labels:
        app: nginx
    spec:
      containers:
      - env: # added new line
        - name: SECRET_NAME1 # added new line
          valueFrom: # added new line
            secretKeyRef: # added new line
              name: akv-secret-name1 # added new line
              key: secretname1 # added new line
        - name: SQL_CONNECTION_STRING # added new line
          valueFrom: # added new line
            secretKeyRef: # added new line
              name: akv-sql # added new line
              key: sqlconnectionstring # added new line
        image: nginx
        name: nginx
        resources: {}
```
Uložte změny ":wq" ve vimu.

A nyní proveďte znovu aplikaci změn:
```
cd ~/manifests
kubectl apply -f  nginx.yaml
```

Jestli neuvidíte nějakou chybovou hlášku, tak byste měli obdržet výstup podobný tomuto:
```
deployment.apps/nginx configured
```

Podívejte se co jste vytvořili:
```
kubectl -n app get deploy,pod,rs,service
```

Měli byste obdržet výstup, který ukazuje, že došlo k vytvoření nového replicasetu a podu:
```
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           41m

NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-67cfb48f7c-lwwnj   1/1     Running   0          99s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-55ddf89fb7   0         0         0       18m
replicaset.apps/nginx-6799fc88d8   0         0         0       41m
replicaset.apps/nginx-67cfb48f7c   1         1         1       99s
```

Nyní konečně zkusme znova vypsat proměnné zevnitř kontejneru, opět nezapomeňte nahradit název podu v příkazu níže za ten váš:
```
# change pod name nginx-67cfb48f7c-lwwnj with your real one
kubectl -n app exec nginx-67cfb48f7c-lwwnj it -- printenv
```

Měli byste obdržet výstup zobrazující taktéž proměnné, které mají stejné hodnoty jak ty v Azure Key Vaults:
```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=nginx-67cfb48f7c-lwwnj
NGINX_VERSION=1.21.5
NJS_VERSION=0.7.1
PKG_RELEASE=1~bullseye
SQL_CONNECTION_STRING=TopSecretConnectionString
SECRET_NAME1=TopSecretPassword
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.0.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.0.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.0.0.1
KUBERNETES_SERVICE_HOST=10.0.0.1
HOME=/root
```
![image](/img/printenv.PNG)

Jestli jste došli až se, tak jste vyzkoušeli cíl tohoto projektu, synchronizovali jste Azure KeyVault secrets do kubernetes secrets a namapovali je přes proměnné do aplikace. 
  
Gratuluji Vám !!!  

Na konec nezapomeňte smazat všechny vytvořené objekty v LABu, tj smazat celou resource group:
``` azcli
az group delete --resource-group $GROUP_NAME
```

