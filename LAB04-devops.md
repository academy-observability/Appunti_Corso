# LAB04 - Deploy FE/BE su Azure Container Apps con Azure DevOps


<details>
  <summary> <h2>⚠️ Attenzione, modifica build immagini Docker nelle Pipelines</h2></summary>
  Prima di andare avanti, modifica il comando per la build delle immagini Docker nelle Pipelines con questo:
  
  BACKEND
  ```bash
   echo "Build backend image: ${BACKEND_IMAGE}"
   docker buildx build \
   --platform linux/amd64 \
   --provenance=false \
   -t ${BACKEND_IMAGE} \
   --push \
   ./backend
  ```

FRONTEND
  ```bash
  echo "Build frontend image: ${FRONTEND_IMAGE}"
  docker buildx build \
  --platform linux/amd64 \
  --provenance=false \
  -t ${FRONTEND_IMAGE} \
  --push \
  ./frontend
 ```
  
  
</details>

## Dal monorepo multi-image alla distribuzione reale dei due servizi nel cloud

---

# 1. Obiettivo del laboratorio

In questo laboratorio completerai il passaggio dal progetto FE/BE preparato in LAB03 al suo **deploy reale nel cloud**.

Partendo dal monorepo GitHub e dalla pipeline multi-image già pronti, realizzerai:

* preparazione dell’ambiente **Azure Container Apps**
* aggiornamento della pipeline Azure DevOps
* build e push delle immagini **frontend** e **backend**
* deploy del **backend** su Azure Container Apps
* recupero del suo FQDN
* deploy del **frontend** con configurazione della variabile `BACKEND_URL`
* verifica finale del collegamento **frontend → backend**
* controllo delle **revisioni** create dal rilascio

Alla fine del laboratorio dovrai essere in grado di:

* spiegare perché Azure Container Apps è più adatto di ACI per il progetto FE/BE
* distribuire due servizi distinti nello stesso environment
* capire il ruolo delle revisioni
* verificare che il frontend chiami correttamente il backend
* leggere i primi segnali operativi del deploy in Azure

---

# 2. Scopo di questo laboratorio

Nel LAB03 hai preparato bene:

* la struttura del monorepo
* il frontend
* il backend
* la pipeline multi-image
* il push delle immagini in ACR

Mancava però il passaggio più importante: usare quelle immagini per creare una **vera applicazione distribuita in Azure**.

Qui facciamo questo salto in modo graduale:

* non torniamo più ad ACI
* non introduciamo ancora tutta la parte osservabile di Application Insights
* ci concentriamo prima sul **deploy corretto** di FE e BE su una piattaforma coerente con il progetto

Azure Container Apps supporta il deploy di nuove versioni tramite **revisioni**, e ogni aggiornamento significativo genera una nuova revisione del servizio. ([Microsoft Learn][2])

---

# Parte 1 - Spiegazione dei termini e degli strumenti

# 3. Architettura del laboratorio

L’architettura finale del laboratorio è questa:

```text
Client
  |
  v
Frontend Container App
  |
  v
Backend Container App

Pipeline Azure DevOps
  |
  +--> build frontend image
  +--> build backend image
  +--> push frontend image to ACR
  +--> push backend image to ACR
  +--> deploy backend to Azure Container Apps
  +--> read backend FQDN
  +--> deploy frontend with BACKEND_URL
```

Questa è la prima volta nel modulo in cui la struttura applicativa FE/BE viene effettivamente distribuita nel cloud come due servizi distinti.

---

# 4. Che cos’è Azure Container Apps

**Azure Container Apps** è un servizio Azure per eseguire applicazioni containerizzate senza dover gestire direttamente l’orchestratore sottostante. Dal punto di vista del corso, è il punto di arrivo naturale dopo ACI: è più adatto a un progetto con più servizi, aggiornamenti e revisioni. ([Microsoft Learn][3])

## 4.1 Perché lo usiamo qui

Lo usiamo perché il progetto non è più una sola app containerizzata, ma due componenti distinti:

* frontend
* backend

E vogliamo distribuirli in un contesto più realistico e più coerente con i passaggi successivi del percorso.

## 4.2 Perché non continuiamo con ACI

ACI è ottimo per i primi deploy semplici, ma diventa meno didattico quando il progetto richiede:

* più servizi
* aggiornamenti ordinati
* configurazione di variabili per il collegamento tra servizi
* visibilità delle versioni deployate

Azure Container Apps copre bene questi aspetti.

---

# 5. Che cos’è un Container Apps Environment

Un **Container Apps Environment** è il contenitore logico in cui vivono una o più container app.

Nel nostro laboratorio l’environment sarà la “casa comune” di:

* frontend
* backend

L’ambiente può essere creato con Azure CLI e può essere collegato a un **Log Analytics Workspace**. Inoltre il gruppo di comandi `az containerapp env` è documentato come gruppo di gestione degli ambienti Container Apps; Microsoft segnala anche che questo gruppo è definito sia in Azure CLI sia in almeno un’estensione, quindi conviene installare o aggiornare l’estensione quando serve. ([Microsoft Learn][4])

---

# 6. Che cosa sono le revisioni

Una **revisione** in Azure Container Apps rappresenta una versione distribuita dell’app.

Ogni volta che cambi qualcosa di rilevante, come:

* immagine
* configurazione
* variabili d’ambiente

Container Apps crea una nuova revisione. Microsoft documenta che Container Apps supporta revision modes e che la gestione delle revisioni è parte nativa del modello di deploy. La lista delle revisioni si può vedere con `az containerapp revision list`. ([Microsoft Learn][2])

## 6.1 Perché sono utili

Le revisioni servono a:

* capire che cosa è stato rilasciato
* distinguere le versioni
* osservare la storia dei deploy
* preparare il terreno a ragionamenti futuri su rollback e gestione del traffico

Nel nostro corso ci basta, per ora, capire che **ogni nuovo deploy crea una nuova revisione**.

---

# 7. Che ruolo ha ACR in LAB04

ACR continua a essere il **registry immagini**.

La pipeline:

* costruisce il backend
* costruisce il frontend
* pubblica entrambe le immagini in ACR

Azure Container Apps poi userà quelle immagini come base del deploy. La documentazione CLI di Container Apps mostra esplicitamente l’uso di `--registry-server`, `--registry-username` e `--registry-password` in `az containerapp create` quando si usa un registry privato. ([Microsoft Learn][3])

## 7.1 Nota importante

Microsoft raccomanda, per scenari più robusti, di usare **managed identity** e ruolo **AcrPull** per permettere alla container app di fare pull dal registry. Anche la documentazione di deploy con Azure Pipelines lo evidenzia come approccio raccomandato. In questo laboratorio, però, usiamo ancora una modalità più semplice con credenziali registry, perché è più lineare da spiegare ai beginners. ([Microsoft Learn][1])

---

# 8. Che ruolo ha Azure DevOps in LAB04

Azure DevOps continua a essere il motore del processo CI/CD.

In questo laboratorio la pipeline deve fare qualcosa di nuovo rispetto al LAB03:

* non solo build e push
* ma anche **deploy dei due servizi**
* con una dipendenza logica tra backend e frontend

Per restare coerenti con i laboratori precedenti, useremo ancora `AzureCLI@2`, che esegue script Bash contro Azure tramite una Azure Resource Manager connection. ([Microsoft Learn][5])

---

# 9. Che cos’è `BACKEND_URL`

`BACKEND_URL` è una **variabile d’ambiente** che useremo per dire al frontend come raggiungere il backend.

Nel deploy del frontend, la pipeline:

1. recupera il FQDN del backend
2. costruisce un URL del tipo:

```text
https://<backend-fqdn>
```

3. lo passa al frontend come variabile d’ambiente

Azure Container Apps consente di impostare variabili d’ambiente sia alla creazione con `--env-vars`, sia in aggiornamento con `az containerapp update --set-env-vars`. Microsoft documenta anche che modificare le environment variables crea una nuova revisione. ([Microsoft Learn][6])

---

# 10. Perché in LAB04 non introduciamo ancora Application Insights

Perché qui il focus è sul **deploy corretto del progetto distribuito**.

Se introducessimo nello stesso laboratorio anche:

* tracing
* telemetria applicativa
* query
* portale osservabile

il laboratorio diventerebbe troppo carico.

Qui chiudiamo bene il pezzo DevOps del deploy FE/BE.
Nel laboratorio successivo torneremo sulla parte observability cloud.

---

# 11. Concetti chiave da ricordare prima della parte pratica

Prima di passare allo step-by-step, devi avere chiaro questo:

* **GitHub** conserva il monorepo FE/BE
* **Azure DevOps** esegue la pipeline
* **ACR** conserva le immagini
* **Azure Container Apps** distribuisce frontend e backend
* il **backend** deve essere deployato prima del **frontend**
* il **frontend** deve ricevere `BACKEND_URL`
* ogni nuovo deploy crea una **revisione**

---

# Parte 2 - Step-by-step guidato

# 12. Prerequisiti del laboratorio

Per eseguire LAB04 devono essere già disponibili:

* repository GitHub FE/BE
* organizzazione Azure DevOps
* progetto Azure DevOps
* pipeline Azure DevOps del LAB03 funzionante
* service connection GitHub funzionante
* service connection Azure Resource Manager funzionante
* Azure Container Registry già esistente
* Azure CLI installata e funzionante
* Docker funzionante
* Git funzionante

È inoltre utile che l’admin account di ACR sia già abilitato, perché in questo laboratorio continuiamo a usare una modalità semplice di autenticazione verso il registry.

---

# 13. Apertura del progetto locale

Apri il terminale WSL:

```bash id="k56q86"
cd ~/corso_obs/lab03-fe-be-devops
git status
find . -maxdepth 2 -type f | sort
```

Dovresti vedere almeno:

* `frontend/app.py`
* `frontend/requirements.txt`
* `frontend/Dockerfile`
* `backend/app.py`
* `backend/requirements.txt`
* `backend/Dockerfile`
* `azure-pipelines.yml`

---

# 14. Verifica minima del codice prima del deploy

Esegui:

```bash id="7fu0sh"
python3 -m py_compile frontend/app.py
python3 -m py_compile backend/app.py
```

Se non compare output, la sintassi è corretta.

---

# 15. Preparazione dell’infrastruttura Azure una tantum

## 15.1 Login Azure

```bash id="6v3jv0"
az login
az account show
```

## 15.2 Installazione o aggiornamento estensione `containerapp`

Microsoft segnala che il gruppo di comandi `az containerapp` e `az containerapp env` è definito sia in Azure CLI sia in almeno un’estensione. Per evitare problemi di comando mancante o funzionalità non allineate, aggiorna l’estensione. ([Microsoft Learn][4])

```bash id="m8q7vq"
az extension add --name containerapp --upgrade || az extension update --name containerapp
```

## 15.3 Creazione del Log Analytics Workspace

Crea il workspace che userai per collegare l’environment di Container Apps.

```bash id="3r3t0j"
az monitor log-analytics workspace create \
  --resource-group NOME_RESOURCE_GROUP \
  --workspace-name NOME_LOG_WORKSPACE \
  --location westeurope
```

## 15.4 Recupero Workspace ID

```bash id="yek0b8"
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group NOME_RESOURCE_GROUP \
  --workspace-name NOME_LOG_WORKSPACE \
  --query customerId \
  -o tsv)

echo $WORKSPACE_ID
```

## 15.5 Recupero Workspace Key

```bash id="l9wghb"
WORKSPACE_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --resource-group NOME_RESOURCE_GROUP \
  --workspace-name NOME_LOG_WORKSPACE \
  --query primarySharedKey \
  -o tsv)

echo $WORKSPACE_KEY
```

## 15.6 Creazione del Container Apps Environment

`az containerapp env create` è il comando documentato per creare un environment, anche collegandolo a un Log Analytics Workspace esistente. ([Microsoft Learn][4])

```bash id="jl8mct"
az containerapp env create \
  --name NOME_ACA_ENV \
  --resource-group NOME_RESOURCE_GROUP \
  --location westeurope \
  --logs-workspace-id $WORKSPACE_ID \
  --logs-workspace-key $WORKSPACE_KEY
```

## 15.7 Verifica dell’environment

```bash id="69q5au"
az containerapp env show \
  --name NOME_ACA_ENV \
  --resource-group NOME_RESOURCE_GROUP \
  --query properties.provisioningState \
  -o tsv
```

Risultato atteso:

```text id="9k0dwl"
Succeeded
```

---

# 16. Verifica del registry ACR

```bash id="u8o28s"
az acr show --name NOME_ACR --resource-group NOME_RESOURCE_GROUP
az acr login --name NOME_ACR
```

---

# 17. Aggiornamento del file `azure-pipelines.yml`

Apri il file:

```bash id="xjlwmr"
code azure-pipelines.yml
```

Sostituisci il contenuto con questo, adattando i valori del tuo ambiente:

```yaml id="17l6m7"
trigger:
  - main

pr: none

pool:
  vmImage: ubuntu-latest

variables:
  azureServiceConnection: 'sc-obs-azure-rg'
  resourceGroupName: 'rg-observability-dev'
  location: 'westeurope'
  acrName: 'nomeregistry'
  containerAppsEnvironment: 'aca-obs-env'
  backendAppName: 'be-obs-app'
  frontendAppName: 'fe-obs-app'
  backendImageName: 'backend'
  frontendImageName: 'frontend'
  imageTag: '$(Build.BuildId)'

stages:
- stage: ValidateRepository
  displayName: Validate monorepo structure
  jobs:
  - job: ValidateFiles
    displayName: Check required files and Python syntax
    steps:
    - checkout: self

    - bash: |
        set -e

        test -f frontend/app.py
        test -f frontend/requirements.txt
        test -f frontend/Dockerfile
        test -f backend/app.py
        test -f backend/requirements.txt
        test -f backend/Dockerfile

        python3 -m py_compile frontend/app.py
        python3 -m py_compile backend/app.py
      displayName: Validate files and Python syntax

- stage: PublishImages
  displayName: Build and push FE/BE images
  dependsOn: ValidateRepository
  condition: succeeded()
  jobs:
  - job: BuildBackend
    displayName: Build and push backend image
    steps:
    - checkout: self

    - task: AzureCLI@2
      displayName: Build and push backend image to ACR
      inputs:
        azureSubscription: '$(azureServiceConnection)'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          set -e
          az account show
          az acr login --name $(acrName)

          BACKEND_IMAGE="$(acrName).azurecr.io/$(backendImageName):$(imageTag)"

          docker build -t ${BACKEND_IMAGE} ./backend
          docker push ${BACKEND_IMAGE}

  - job: BuildFrontend
    displayName: Build and push frontend image
    steps:
    - checkout: self

    - task: AzureCLI@2
      displayName: Build and push frontend image to ACR
      inputs:
        azureSubscription: '$(azureServiceConnection)'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          set -e
          az account show
          az acr login --name $(acrName)

          FRONTEND_IMAGE="$(acrName).azurecr.io/$(frontendImageName):$(imageTag)"

          docker build -t ${FRONTEND_IMAGE} ./frontend
          docker push ${FRONTEND_IMAGE}

- stage: DeployBackend
  displayName: Deploy backend to ACA
  dependsOn: PublishImages
  condition: succeeded()
  jobs:
  - job: DeployBackendJob
    displayName: Deploy or update backend container app
    steps:
    - task: AzureCLI@2
      displayName: Deploy backend app
      inputs:
        azureSubscription: '$(azureServiceConnection)'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          set -e

          az extension add --name containerapp --upgrade || az extension update --name containerapp
          az account show
          az acr login --name $(acrName)

          BACKEND_IMAGE="$(acrName).azurecr.io/$(backendImageName):$(imageTag)"

          ACR_USER=$(az acr credential show --name $(acrName) --query username -o tsv)
          ACR_PASS=$(az acr credential show --name $(acrName) --query "passwords[0].value" -o tsv)

          if az containerapp show --name $(backendAppName) --resource-group $(resourceGroupName) >/dev/null 2>&1; then
            echo "Backend app esiste già: aggiorno immagine"
            az containerapp update \
              --name $(backendAppName) \
              --resource-group $(resourceGroupName) \
              --image ${BACKEND_IMAGE}
          else
            echo "Backend app non esiste: creo la container app"
            az containerapp create \
              --name $(backendAppName) \
              --resource-group $(resourceGroupName) \
              --environment $(containerAppsEnvironment) \
              --image ${BACKEND_IMAGE} \
              --target-port 8000 \
              --ingress external \
              --min-replicas 1 \
              --max-replicas 1 \
              --registry-server $(acrName).azurecr.io \
              --registry-username ${ACR_USER} \
              --registry-password ${ACR_PASS}
          fi

          az containerapp show \
            --name $(backendAppName) \
            --resource-group $(resourceGroupName) \
            --query properties.configuration.ingress.fqdn \
            -o tsv

- stage: DeployFrontend
  displayName: Deploy frontend to ACA
  dependsOn: DeployBackend
  condition: succeeded()
  jobs:
  - job: DeployFrontendJob
    displayName: Deploy or update frontend container app
    steps:
    - task: AzureCLI@2
      displayName: Deploy frontend app
      inputs:
        azureSubscription: '$(azureServiceConnection)'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          set -e

          az extension add --name containerapp --upgrade || az extension update --name containerapp
          az account show
          az acr login --name $(acrName)

          FRONTEND_IMAGE="$(acrName).azurecr.io/$(frontendImageName):$(imageTag)"

          ACR_USER=$(az acr credential show --name $(acrName) --query username -o tsv)
          ACR_PASS=$(az acr credential show --name $(acrName) --query "passwords[0].value" -o tsv)

          BE_FQDN=$(az containerapp show \
            --name $(backendAppName) \
            --resource-group $(resourceGroupName) \
            --query properties.configuration.ingress.fqdn \
            -o tsv)

          BACKEND_URL="https://${BE_FQDN}"

          if az containerapp show --name $(frontendAppName) --resource-group $(resourceGroupName) >/dev/null 2>&1; then
            echo "Frontend app esiste già: aggiorno immagine e BACKEND_URL"
            az containerapp update \
              --name $(frontendAppName) \
              --resource-group $(resourceGroupName) \
              --image ${FRONTEND_IMAGE} \
              --set-env-vars BACKEND_URL=${BACKEND_URL}
          else
            echo "Frontend app non esiste: creo la container app"
            az containerapp create \
              --name $(frontendAppName) \
              --resource-group $(resourceGroupName) \
              --environment $(containerAppsEnvironment) \
              --image ${FRONTEND_IMAGE} \
              --target-port 8000 \
              --ingress external \
              --min-replicas 1 \
              --max-replicas 1 \
              --registry-server $(acrName).azurecr.io \
              --registry-username ${ACR_USER} \
              --registry-password ${ACR_PASS} \
              --env-vars BACKEND_URL=${BACKEND_URL}
          fi

          az containerapp show \
            --name $(frontendAppName) \
            --resource-group $(resourceGroupName) \
            --query properties.configuration.ingress.fqdn \
            -o tsv

- stage: VerifyDeployment
  displayName: Verify FE and BE deployment
  dependsOn: DeployFrontend
  condition: succeeded()
  jobs:
  - job: VerifyJob
    displayName: Run endpoint checks and show revisions
    steps:
    - task: AzureCLI@2
      displayName: Verify deployed apps
      inputs:
        azureSubscription: '$(azureServiceConnection)'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          set -e

          az extension add --name containerapp --upgrade || az extension update --name containerapp

          FE_FQDN=$(az containerapp show \
            --name $(frontendAppName) \
            --resource-group $(resourceGroupName) \
            --query properties.configuration.ingress.fqdn \
            -o tsv)

          BE_FQDN=$(az containerapp show \
            --name $(backendAppName) \
            --resource-group $(resourceGroupName) \
            --query properties.configuration.ingress.fqdn \
            -o tsv)

          echo "Attendo 20 secondi per warm-up"
          sleep 20

          echo "Test backend /health"
          curl --fail --silent --show-error "https://${BE_FQDN}/health"
          echo

          echo "Test frontend /health"
          curl --fail --silent --show-error "https://${FE_FQDN}/health"
          echo

          echo "Test frontend /demo"
          curl --fail --silent --show-error "https://${FE_FQDN}/demo"
          echo

          echo "Revision list backend"
          az containerapp revision list \
            --name $(backendAppName) \
            --resource-group $(resourceGroupName) \
            -o table

          echo "Revision list frontend"
          az containerapp revision list \
            --name $(frontendAppName) \
            --resource-group $(resourceGroupName) \
            -o table
```

### Nota importante sulla pipeline

Microsoft oggi mette a disposizione anche il task `AzureContainerApps@1`, pensato proprio per build e deploy su Container Apps. Noi qui **non lo usiamo**, per mantenere il percorso coerente con i laboratori precedenti e per far vedere esplicitamente i comandi Azure CLI. ([Microsoft Learn][1])

---

# 18. Perché questa pipeline è corretta per LAB04

Questa pipeline fa esattamente quello che serve in questo laboratorio:

1. verifica la struttura del monorepo
2. costruisce e pubblica frontend e backend in ACR
3. deploya prima il backend
4. legge il suo FQDN
5. deploya il frontend con `BACKEND_URL`
6. esegue una verifica finale
7. mostra le revisioni create

Questa struttura è coerente con il modo in cui Azure Container Apps gestisce nuove revisioni a ogni aggiornamento significativo. ([Microsoft Learn][2])

---

# 19. Salvataggio e push su GitHub

```bash id="bawlgf"
git add .
git commit -m "LAB04 - Deploy FE/BE to Azure Container Apps with Azure DevOps"
git push
```

Verifica che il file aggiornato sia presente su GitHub.

---

# 20. Esecuzione della pipeline in Azure DevOps

Nel portale Azure DevOps:

1. vai in **Pipelines**
2. apri la pipeline collegata al repository GitHub FE/BE
3. verifica che usi il file `/azure-pipelines.yml`
4. esegui la pipeline

Dovresti vedere chiaramente queste stage:

* `ValidateRepository`
* `PublishImages`
* `DeployBackend`
* `DeployFrontend`
* `VerifyDeployment`

---

# 21. Verifica dello stage `PublishImages`

Controlla nei log che compaiano:

* `az account show`
* `az acr login`
* `docker build`
* `docker push`

Per ulteriore verifica, puoi usare da CLI:

```bash id="4di3na"
az acr repository show-tags --name NOME_ACR --repository frontend -o table
az acr repository show-tags --name NOME_ACR --repository backend -o table
```

---

# 22. Verifica dello stage `DeployBackend`

Controlla nei log che avvengano questi passaggi:

* recupero credenziali ACR
* controllo se la backend app esiste
* `az containerapp create` oppure `az containerapp update`
* stampa del FQDN del backend

La documentazione CLI di Container Apps mostra sia `az containerapp create` con immagine e env-vars, sia l’aggiornamento dell’applicazione con `az containerapp update`. ([Microsoft Learn][3])

---

# 23. Verifica dello stage `DeployFrontend`

Controlla nei log:

* recupero FQDN del backend
* costruzione della variabile `BACKEND_URL`
* `az containerapp create` oppure `az containerapp update`
* stampa del FQDN del frontend

Questa è la parte chiave del laboratorio, perché è qui che il frontend viene collegato al backend.

---

# 24. Verifica dello stage `VerifyDeployment`

Controlla nei log:

* test `backend /health`
* test `frontend /health`
* test `frontend /demo`
* elenco revisioni del backend
* elenco revisioni del frontend

La lista revisioni usa il comando documentato `az containerapp revision list`. ([Microsoft Learn][7])

---

# 25. Verifiche manuali finali

## 25.1 Recupero FQDN backend

```bash id="t83gmf"
BE_FQDN=$(az containerapp show \
  --name NOME_BACKEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  --query properties.configuration.ingress.fqdn \
  -o tsv)

echo $BE_FQDN
```

## 25.2 Recupero FQDN frontend

```bash id="xodnqv"
FE_FQDN=$(az containerapp show \
  --name NOME_FRONTEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  --query properties.configuration.ingress.fqdn \
  -o tsv)

echo $FE_FQDN
```

## 25.3 Test backend

```bash id="5l7b0k"
curl https://$BE_FQDN/health
curl https://$BE_FQDN/work
```

## 25.4 Test frontend

```bash id="cf22kf"
curl https://$FE_FQDN/health
curl https://$FE_FQDN/demo
```

---

# 26. Verifica delle revisioni

## 26.1 Revisioni backend

```bash id="gkzj6y"
az containerapp revision list \
  --name NOME_BACKEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  -o table
```

## 26.2 Revisioni frontend

```bash id="hm88xq"
az containerapp revision list \
  --name NOME_FRONTEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  -o table
```

---

# 27. Prime verifiche di log da CLI

Container Apps permette di leggere i log da CLI con `az containerapp logs show`. Più avanti ci servirà per la parte observability, ma è utile già qui come controllo operativo minimo. ([Microsoft Learn][3])

## 27.1 Log backend

```bash id="o5n74s"
az containerapp logs show \
  --name NOME_BACKEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  --tail 20
```

## 27.2 Log frontend

```bash id="v4j4zv"
az containerapp logs show \
  --name NOME_FRONTEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  --tail 20
```

---

# 28. Troubleshooting minimo

## 28.1 Errore `az containerapp` non riconosciuto

Possibile causa:

* estensione `containerapp` non installata o non aggiornata

Soluzione:

```bash id="1r71sm"
az extension add --name containerapp --upgrade || az extension update --name containerapp
```

Microsoft segnala che questo gruppo di comandi è definito sia nella CLI sia in almeno un’estensione e raccomanda di installare le estensioni per beneficiare delle capacità estese. ([Microsoft Learn][4])

## 28.2 Errore nel deploy da ACR

Possibili cause:

* nome ACR errato
* credenziali ACR non disponibili
* admin account ACR non abilitato

Verifica:

```bash id="jlwmnp"
az acr show --name NOME_ACR --resource-group NOME_RESOURCE_GROUP
az acr credential show --name NOME_ACR
```

## 28.3 Il frontend risponde ma `/demo` fallisce

Possibili cause:

* `BACKEND_URL` errata
* backend non ancora pronto
* FQDN del backend letto male

Verifica la configurazione env del frontend:

```bash id="60dcpq"
az containerapp show \
  --name NOME_FRONTEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  --query properties.template.containers[0].env
```

Ricorda che modificare o aggiungere env-vars crea una nuova revisione della container app. ([Microsoft Learn][6])

## 28.4 Lo stage finale fallisce troppo presto

Possibile causa:

* warm-up insufficiente dopo il deploy

Soluzione:
aumenta temporaneamente:

```bash id="pu6zsa"
sleep 20
```

a:

```bash id="31pw6n"
sleep 30
```

## 28.5 Nessun log utile visibile

Possibili cause:

* stai guardando troppo presto
* l’app non ha ancora ricevuto richieste
* stai interrogando l’app sbagliata

Verifica con:

```bash id="mron4b"
az containerapp logs show --name NOME_FRONTEND_APP --resource-group NOME_RESOURCE_GROUP --tail 20
az containerapp logs show --name NOME_BACKEND_APP --resource-group NOME_RESOURCE_GROUP --tail 20
```

---

# 29. Evidenze richieste

Crea il file:

```bash id="x8k64t"
code docs/evidence_lab04.md
```

Inserisci questa struttura:

```md id="y0eonv"
# Evidence LAB04

## 1. Obiettivo compreso
Spiego con parole mie perché in questo laboratorio passiamo da ACI a Azure Container Apps.

## 2. Risorse create
Indico:
- Resource Group
- Log Analytics Workspace
- Container Apps Environment
- Backend Container App
- Frontend Container App

## 3. Pipeline
Descrivo le stage della pipeline e il loro ruolo.

## 4. Deploy backend
Spiego come è stato deployato il backend e riporto il suo FQDN.

## 5. Deploy frontend
Spiego come è stato deployato il frontend e riporto il suo FQDN.

## 6. Variabile BACKEND_URL
Spiego come il frontend raggiunge il backend.

## 7. Test finali
Incollo l’output di:
- GET /health backend
- GET /health frontend
- GET /demo frontend

## 8. Revisioni
Incollo l’output di:
- az containerapp revision list backend
- az containerapp revision list frontend

## 9. Log
Incollo alcune righe di log utili e commento ciò che osservo.

## 10. Problemi incontrati
Descrivo eventuali errori e come li ho risolti.
```

---

# 30. Consegna

Al termine del laboratorio devi consegnare:

* repository GitHub FE/BE aggiornato
* pipeline Azure DevOps funzionante
* file `docs/evidence_lab04.md`

Salva il lavoro:

```bash id="xayb0z"
git add .
git commit -m "LAB04 completato - deploy FE/BE on Azure Container Apps"
git push
```

---

# 31. Conclusione del laboratorio

In questo laboratorio hai completato un passaggio fondamentale del modulo:

* sei partito da immagini FE e BE già costruite
* le hai usate per creare due servizi distinti nel cloud
* hai collegato il frontend al backend
* hai visto comparire il concetto di **revisione**

Hai quindi preparato la base per il laboratorio successivo, in cui finalmente tornerà in primo piano il vero cuore del progetto:

**l’observability cloud del sistema distribuito**.

---

Questo è il **LAB04 corretto** nello stesso standard dei precedenti.
Il passo successivo naturale è **LAB05** con:

* Parte 1 spiegazione di Application Insights, Azure Monitor e Log Analytics
* Parte 2 step-by-step per strumentazione, deploy e verifiche osservabili.

[1]: https://learn.microsoft.com/en-us/azure/container-apps/azure-pipelines "Publish Revisions by Using Azure Pipelines in Azure Container Apps | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/azure/container-apps/revisions "Update and deploy changes in Azure Container Apps | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/cli/azure/containerapp?view=azure-cli-latest "az containerapp | Microsoft Learn"
[4]: https://learn.microsoft.com/en-us/cli/azure/containerapp/env?view=azure-cli-latest "az containerapp env | Microsoft Learn"
[5]: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/azure-cli-v2?view=azure-pipelines&utm_source=chatgpt.com "AzureCLI@2 - Azure CLI v2 task"
[6]: https://learn.microsoft.com/en-us/azure/container-apps/environment-variables "Manage environment variables on Azure Container Apps | Microsoft Learn"
[7]: https://learn.microsoft.com/en-us/cli/azure/containerapp/revision?view=azure-cli-latest&utm_source=chatgpt.com "az containerapp revision"
