# LAB05 - Observability Azure sul progetto FE/BE

## Application Insights, Azure Monitor, Log Analytics e tracing distribuito

---

# 1. Obiettivo del laboratorio

In questo laboratorio completerai l’ultima parte del percorso DevOps + Observability del modulo 3.

Partendo dal progetto FE/BE già deployato su Azure Container Apps nel LAB04, realizzerai:

* creazione della risorsa **Application Insights**
* aggiornamento del codice FE e BE con **Azure Monitor OpenTelemetry**
* aggiornamento delle dipendenze Python
* aggiornamento della pipeline Azure DevOps
* deploy di nuove revisioni FE e BE con telemetria applicativa attiva
* generazione di traffico normale e di errore
* analisi di:

  * richieste applicative
  * dipendenze FE → BE
  * errori
  * dettagli end-to-end
  * log console di Azure Container Apps
  * query KQL in Application Insights e Log Analytics

Alla fine del laboratorio dovrai essere in grado di:

* spiegare perché **Application Insights** svolge nel cloud il ruolo APM/tracing che, nel LAB00 locale, avevi affidato a strumenti separati
* usare `configure_azure_monitor()` in una app Python
* capire la differenza tra telemetria applicativa e log runtime del container
* eseguire query KQL su tabelle di Application Insights e su tabelle di Container Apps nel workspace Log Analytics ([Microsoft Learn][1])

---

# 2. Perché questo laboratorio esiste

Nel LAB04 hai portato frontend e backend su **Azure Container Apps**.
Mancava però la parte decisiva: osservare davvero il comportamento dell’applicazione distribuita dopo il deploy.

Qui chiudiamo il cerchio:

* **DevOps** ha portato il software nel cloud
* **Observability** ti aiuta a capire come si comporta lì dentro

Application Insights raccoglie telemetria applicativa, mentre Azure Container Apps invia nel workspace anche log console e log di sistema. I due flussi sono collegati, ma non sono la stessa cosa. ([Microsoft Learn][1])

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

Telemetry flow
  |
  +--> Application Insights
  +--> Azure Monitor
  +--> Log Analytics
```

Nel LAB05 non cambiamo la topologia dell’applicazione.
Cambiamo il livello di visibilità che abbiamo su di essa.

---

# 4. Che cos’è Application Insights

**Application Insights** è la componente APM di Azure Monitor.
Serve a raccogliere telemetria applicativa come richieste, dipendenze, errori e dettagli delle operazioni end-to-end. Il percorso code-based documentato da Microsoft è: creare la risorsa, ottenere la connection string, aggiungere la distro OpenTelemetry e configurarla nell’app. ([Microsoft Learn][1])

## 4.1 Perché ci serve in questo laboratorio

Perché vogliamo vedere:

* richieste verso il frontend
* richieste verso il backend
* chiamate del frontend al backend
* errori
* durata delle operazioni
* collegamenti tra gli eventi della stessa richiesta

---

# 5. Che cos’è OpenTelemetry nel nostro laboratorio

Per Python useremo la libreria `azure-monitor-opentelemetry`.
La documentazione Microsoft per Python dice chiaramente che, per la strumentazione manuale, bisogna usare `configure_azure_monitor()`, e che l’automatic instrumentation non è supportata. ([Microsoft Learn][2])

## 5.1 Perché è utile

Ci permette di strumentare l’applicazione in modo coerente con Azure Monitor, senza introdurre SDK legacy separati.

## 5.2 Che cosa faremo in pratica

Nei due servizi:

* aggiungeremo `azure-monitor-opentelemetry`
* chiameremo `configure_azure_monitor(...)`
* passeremo la connection string tramite variabile d’ambiente
* useremo `service.name` per distinguere frontend e backend

---

# 6. Che cos’è la connection string di Application Insights

La connection string identifica la risorsa Application Insights a cui inviare la telemetria.
Microsoft documenta che può essere recuperata dal portale oppure con CLI usando:

```bash
az monitor app-insights component show --app <nome> --resource-group <rg> --query connectionString --output tsv
```

Inoltre i comandi `az monitor app-insights component ...` appartengono all’estensione CLI `application-insights`, che viene installata automaticamente al primo uso se necessario. ([Microsoft Learn][3])

---

# 7. Che cosa sono richieste e dipendenze in Application Insights

Il modello dati di Application Insights distingue diversi tipi di telemetria.
Per questo laboratorio ci interessano soprattutto:

* **Request**: richieste ricevute dall’applicazione
* **Dependency**: chiamate in uscita verso altri componenti

Microsoft documenta esplicitamente il mapping tra nomi “brevi” e tabelle workspace-based:

* `requests` ↔ `AppRequests`
* `dependencies` ↔ `AppDependencies` ([Microsoft Learn][4])

## 7.1 Quale forma useremo nelle query

Per evitare ambiguità, nelle query del laboratorio useremo la forma **workspace-based esplicita**:

* `AppRequests`
* `AppDependencies`

Microsoft precisa anche che, per le risorse workspace-based, esiste compatibilità retroattiva con i legacy table names, ma qui scegliamo i nomi espliciti per essere più chiari. ([Microsoft Learn][5])

---

# 8. Che cos’è `OperationId`

Nel modello dati di Application Insights, le richieste e le dipendenze condividono campi di correlazione come `OperationId`.
Questo campo serve a collegare eventi appartenenti alla stessa operazione logica. Nella tabella `AppDependencies`, Microsoft documenta `OperationId` come identificatore dell’operazione definita dall’applicazione. ([Microsoft Learn][6])

## 8.1 Perché ci serve

Perché vogliamo ricostruire il percorso:

* richiesta al frontend
* chiamata in uscita dal frontend
* richiesta ricevuta dal backend

---

# 9. Che ruolo hanno Azure Monitor e Log Analytics

**Azure Monitor** è il contenitore più ampio dell’osservabilità Azure.
**Log Analytics** è il punto in cui i dati finiscono e diventano interrogabili via KQL. Microsoft documenta che, quando Azure Container Apps usa Log Analytics come soluzione di logging, il workspace raccoglie log di sistema e log applicativi di tutte le container app dell’environment. ([Microsoft Learn][7])

---

# 10. Che cosa sono i log console e i log di sistema di Container Apps

Azure Container Apps espone due categorie principali di log nel workspace:

* **Console logs**: log scritti dall’applicazione su `stdout` o `stderr`
* **System logs**: log generati dalla piattaforma Container Apps

Microsoft documenta le due tabelle principali come:

* `ContainerAppConsoleLogs_CL`
* `ContainerAppSystemLogs_CL` ([Microsoft Learn][8])

## 10.1 Perché ci servono entrambe

Perché vogliamo distinguere:

* ciò che scrive la nostra applicazione
* ciò che segnala la piattaforma

Questo aiuta molto nel troubleshooting.

---

# 11. Perché manteniamo anche i log JSON su stdout

Anche se Application Insights raccoglie telemetria APM, manteniamo log JSON su stdout perché:

* finiscono in `ContainerAppConsoleLogs_CL`
* sono utili per query runtime molto pratiche
* permettono una continuità concettuale con il lavoro fatto nei lab precedenti

I console logs di Container Apps includono proprio ciò che l’app scrive su stdout/stderr. ([Microsoft Learn][9])

---

# 12. Concetti chiave da ricordare prima della parte pratica

Prima di passare allo step-by-step, devi avere chiaro questo:

* **GitHub** conserva il monorepo
* **Azure DevOps** esegue la pipeline
* **ACR** conserva le immagini
* **Azure Container Apps** esegue frontend e backend
* **Application Insights** raccoglie telemetria applicativa
* **Log Analytics** contiene anche i log di Container Apps
* `OperationId` serve a correlare eventi della stessa operazione
* `AppRequests` e `AppDependencies` sono le tabelle principali che useremo

---

# Parte 2 - Step-by-step guidato

# 13. Prerequisiti del laboratorio

Per eseguire LAB05 devono essere già disponibili:

* repository GitHub FE/BE
* organizzazione Azure DevOps
* progetto Azure DevOps
* pipeline Azure DevOps del LAB04 funzionante
* service connection GitHub funzionante
* service connection Azure Resource Manager funzionante
* Azure Container Registry già esistente
* Azure Container Apps Environment già creato
* frontend e backend già deployati su Azure Container Apps
* Log Analytics Workspace già creato

Inoltre devi avere una Azure CLI aggiornata. Per i comandi `az monitor app-insights component ...`, la CLI usa l’estensione `application-insights`, che può installarsi automaticamente al primo uso. ([Microsoft Learn][10])

---

# 14. Apertura del progetto locale

Apri il terminale WSL:

```bash
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

# 15. Creazione della risorsa Application Insights

## 15.1 Installazione estensione CLI

```bash
az extension add -n application-insights
```

## 15.2 Creazione della risorsa

```bash
az monitor app-insights component create \
  --app NOME_APPINSIGHTS \
  --location westeurope \
  --resource-group NOME_RESOURCE_GROUP \
  --workspace NOME_LOG_WORKSPACE
```

## 15.3 Recupero della connection string

```bash
APPINSIGHTS_CONNECTION_STRING=$(az monitor app-insights component show \
  --app NOME_APPINSIGHTS \
  --resource-group NOME_RESOURCE_GROUP \
  --query connectionString \
  -o tsv)

echo $APPINSIGHTS_CONNECTION_STRING
```

## 15.4 Verifica della risorsa

```bash
az monitor app-insights component show \
  --app NOME_APPINSIGHTS \
  --resource-group NOME_RESOURCE_GROUP \
  --query "{name:name,location:location,workspace:workspaceResourceId}" \
  -o json
```

Microsoft documenta sia la creazione della risorsa sia il recupero della connection string via CLI. ([Microsoft Learn][3])

---

# 16. Aggiornamento delle dipendenze Python

Per Python useremo `azure-monitor-opentelemetry`, che Microsoft documenta come pacchetto contenente componenti Azure Monitor e OpenTelemetry, con `configure_azure_monitor()` per la strumentazione manuale. ([Microsoft Learn][2])

## 16.1 File `backend/requirements.txt`

Apri:

```bash
code backend/requirements.txt
```

Sostituisci con:

```txt
flask==3.0.0
azure-monitor-opentelemetry
```

## 16.2 File `frontend/requirements.txt`

Apri:

```bash
code frontend/requirements.txt
```

Sostituisci con:

```txt
flask==3.0.0
requests==2.31.0
azure-monitor-opentelemetry
```

---

# 17. Aggiornamento del backend

Apri:

```bash
code backend/app.py
```

Sostituisci con:

```python
import json
import os
import random
import time
import uuid
from datetime import datetime, timezone

from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry.sdk.resources import Resource
from flask import Flask, g, jsonify, request

SERVICE_NAME = os.getenv("SERVICE_NAME", "backend")
APPINSIGHTS_CONNECTION_STRING = os.getenv("APPLICATIONINSIGHTS_CONNECTION_STRING", "")

resource = Resource.create({"service.name": SERVICE_NAME})

if APPINSIGHTS_CONNECTION_STRING:
    configure_azure_monitor(
        connection_string=APPINSIGHTS_CONNECTION_STRING,
        resource=resource,
    )

app = Flask(__name__)


def utc_now_iso() -> str:
    return datetime.now(timezone.utc).isoformat()


def write_log(status_code: int, message: str = "request_completed") -> None:
    latency_ms = round((time.perf_counter() - g.start_time) * 1000, 2)

    record = {
        "timestamp": utc_now_iso(),
        "service": SERVICE_NAME,
        "level": "INFO" if status_code < 400 else "ERROR",
        "message": message,
        "request_id": g.request_id,
        "method": request.method,
        "path": request.path,
        "status": int(status_code),
        "latency_ms": latency_ms,
        "client_ip": request.headers.get("X-Forwarded-For", request.remote_addr),
        "user_agent": request.headers.get("User-Agent"),
    }

    print(json.dumps(record), flush=True)


@app.before_request
def before_request() -> None:
    g.start_time = time.perf_counter()
    g.request_id = request.headers.get("X-Request-Id", str(uuid.uuid4()))


@app.after_request
def after_request(response):
    write_log(response.status_code)
    response.headers["X-Request-Id"] = g.request_id
    return response


@app.get("/health")
def health():
    return jsonify({"status": "ok", "service": SERVICE_NAME}), 200


@app.get("/work")
def work():
    delay = random.uniform(0.1, 0.8)
    time.sleep(delay)

    return jsonify(
        {
            "service": SERVICE_NAME,
            "message": "backend completed",
            "processing_time": round(delay, 3),
        }
    ), 200


@app.get("/work-error")
def work_error():
    return jsonify(
        {
            "service": SERVICE_NAME,
            "error": "simulated_backend_error",
            "status": 500,
        }
    ), 500


if __name__ == "__main__":
    port = int(os.getenv("PORT", "8000"))
    app.run(host="0.0.0.0", port=port)
```

---

# 18. Aggiornamento del frontend

Apri:

```bash
code frontend/app.py
```

Sostituisci con:

```python
import json
import os
import time
import uuid
from datetime import datetime, timezone

import requests
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry.sdk.resources import Resource
from flask import Flask, g, jsonify, request

SERVICE_NAME = os.getenv("SERVICE_NAME", "frontend")
BACKEND_URL = os.getenv("BACKEND_URL", "")
APPINSIGHTS_CONNECTION_STRING = os.getenv("APPLICATIONINSIGHTS_CONNECTION_STRING", "")

resource = Resource.create({"service.name": SERVICE_NAME})

if APPINSIGHTS_CONNECTION_STRING:
    configure_azure_monitor(
        connection_string=APPINSIGHTS_CONNECTION_STRING,
        resource=resource,
    )

app = Flask(__name__)


def utc_now_iso() -> str:
    return datetime.now(timezone.utc).isoformat()


def write_log(status_code: int, message: str = "request_completed") -> None:
    latency_ms = round((time.perf_counter() - g.start_time) * 1000, 2)

    record = {
        "timestamp": utc_now_iso(),
        "service": SERVICE_NAME,
        "level": "INFO" if status_code < 400 else "ERROR",
        "message": message,
        "request_id": g.request_id,
        "method": request.method,
        "path": request.path,
        "status": int(status_code),
        "latency_ms": latency_ms,
        "client_ip": request.headers.get("X-Forwarded-For", request.remote_addr),
        "user_agent": request.headers.get("User-Agent"),
    }

    print(json.dumps(record), flush=True)


@app.before_request
def before_request() -> None:
    g.start_time = time.perf_counter()
    g.request_id = request.headers.get("X-Request-Id", str(uuid.uuid4()))


@app.after_request
def after_request(response):
    write_log(response.status_code)
    response.headers["X-Request-Id"] = g.request_id
    return response


@app.get("/health")
def health():
    return jsonify({"status": "ok", "service": SERVICE_NAME}), 200


@app.get("/demo")
def demo():
    if not BACKEND_URL:
        return jsonify({"error": "backend_url_not_configured", "status": 500}), 500

    headers = {"X-Request-Id": g.request_id}
    response = requests.get(f"{BACKEND_URL}/work", headers=headers, timeout=5)

    return (
        jsonify(
            {
                "service": SERVICE_NAME,
                "message": "frontend completed",
                "backend_response": response.json(),
            }
        ),
        response.status_code,
    )


@app.get("/demo-error")
def demo_error():
    if not BACKEND_URL:
        return jsonify({"error": "backend_url_not_configured", "status": 500}), 500

    headers = {"X-Request-Id": g.request_id}
    response = requests.get(f"{BACKEND_URL}/work-error", headers=headers, timeout=5)

    return (
        jsonify(
            {
                "service": SERVICE_NAME,
                "message": "frontend completed with backend error path",
                "backend_response": response.json(),
            }
        ),
        response.status_code,
    )


if __name__ == "__main__":
    port = int(os.getenv("PORT", "8000"))
    app.run(host="0.0.0.0", port=port)
```

---

# 19. Verifica sintattica locale

```bash
python3 -m py_compile backend/app.py
python3 -m py_compile frontend/app.py
```

Se non compare output, la sintassi è corretta.

---

# 20. Aggiornamento della pipeline Azure DevOps

Apri:

```bash
code azure-pipelines.yml
```

Sostituisci con:

```yaml
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
  appInsightsName: 'appi-obs-dev'
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
  displayName: Deploy backend to ACA with App Insights
  dependsOn: PublishImages
  condition: succeeded()
  jobs:
  - job: DeployBackendJob
    displayName: Deploy backend container app
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
          az extension add -n application-insights || true

          az account show
          az acr login --name $(acrName)

          BACKEND_IMAGE="$(acrName).azurecr.io/$(backendImageName):$(imageTag)"
          APPINSIGHTS_CONNECTION_STRING=$(az monitor app-insights component show \
            --app $(appInsightsName) \
            --resource-group $(resourceGroupName) \
            --query connectionString \
            -o tsv)

          ACR_USER=$(az acr credential show --name $(acrName) --query username -o tsv)
          ACR_PASS=$(az acr credential show --name $(acrName) --query "passwords[0].value" -o tsv)

          if az containerapp show --name $(backendAppName) --resource-group $(resourceGroupName) >/dev/null 2>&1; then
            az containerapp update \
              --name $(backendAppName) \
              --resource-group $(resourceGroupName) \
              --image ${BACKEND_IMAGE} \
              --set-env-vars APPLICATIONINSIGHTS_CONNECTION_STRING="${APPINSIGHTS_CONNECTION_STRING}" SERVICE_NAME="backend"
          else
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
              --registry-password ${ACR_PASS} \
              --env-vars APPLICATIONINSIGHTS_CONNECTION_STRING="${APPINSIGHTS_CONNECTION_STRING}" SERVICE_NAME="backend"
          fi

- stage: DeployFrontend
  displayName: Deploy frontend to ACA with App Insights
  dependsOn: DeployBackend
  condition: succeeded()
  jobs:
  - job: DeployFrontendJob
    displayName: Deploy frontend container app
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
          az extension add -n application-insights || true

          az account show
          az acr login --name $(acrName)

          FRONTEND_IMAGE="$(acrName).azurecr.io/$(frontendImageName):$(imageTag)"
          APPINSIGHTS_CONNECTION_STRING=$(az monitor app-insights component show \
            --app $(appInsightsName) \
            --resource-group $(resourceGroupName) \
            --query connectionString \
            -o tsv)

          ACR_USER=$(az acr credential show --name $(acrName) --query username -o tsv)
          ACR_PASS=$(az acr credential show --name $(acrName) --query "passwords[0].value" -o tsv)

          BE_FQDN=$(az containerapp show \
            --name $(backendAppName) \
            --resource-group $(resourceGroupName) \
            --query properties.configuration.ingress.fqdn \
            -o tsv)

          BACKEND_URL="https://${BE_FQDN}"

          if az containerapp show --name $(frontendAppName) --resource-group $(resourceGroupName) >/dev/null 2>&1; then
            az containerapp update \
              --name $(frontendAppName) \
              --resource-group $(resourceGroupName) \
              --image ${FRONTEND_IMAGE} \
              --set-env-vars APPLICATIONINSIGHTS_CONNECTION_STRING="${APPINSIGHTS_CONNECTION_STRING}" SERVICE_NAME="frontend" BACKEND_URL="${BACKEND_URL}"
          else
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
              --env-vars APPLICATIONINSIGHTS_CONNECTION_STRING="${APPINSIGHTS_CONNECTION_STRING}" SERVICE_NAME="frontend" BACKEND_URL="${BACKEND_URL}"
          fi

- stage: VerifyDeployment
  displayName: Verify FE and BE deployment
  dependsOn: DeployFrontend
  condition: succeeded()
  jobs:
  - job: VerifyJob
    displayName: Generate traffic and inspect revisions
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

          echo "Warm-up"
          sleep 20

          curl --fail --silent --show-error "https://${BE_FQDN}/health"
          echo
          curl --fail --silent --show-error "https://${FE_FQDN}/health"
          echo

          for i in {1..10}; do
            curl --fail --silent --show-error "https://${FE_FQDN}/demo" >/dev/null
          done

          for i in {1..5}; do
            curl --silent --show-error "https://${FE_FQDN}/demo-error" >/dev/null || true
          done
```

L’aggiornamento delle env vars e dell’immagine con `az containerapp update` è supportato ufficialmente, e se la modifica è revision-scope viene generata una nuova revisione. ([Microsoft Learn][11])

---

# 21. Salvataggio e push su GitHub

```bash
git add .
git commit -m "LAB05 - Add Azure Monitor OpenTelemetry and Application Insights"
git push
```

---

# 22. Esecuzione della pipeline in Azure DevOps

Nel portale Azure DevOps:

1. vai in **Pipelines**
2. apri la pipeline collegata al repository GitHub FE/BE
3. verifica che usi il file `/azure-pipelines.yml`
4. esegui la pipeline

Dovresti vedere:

* `ValidateRepository`
* `PublishImages`
* `DeployBackend`
* `DeployFrontend`
* `VerifyDeployment`

---

# 23. Verifiche manuali finali

## 23.1 Recupero FQDN backend

```bash
BE_FQDN=$(az containerapp show \
  --name NOME_BACKEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  --query properties.configuration.ingress.fqdn \
  -o tsv)

echo $BE_FQDN
```

## 23.2 Recupero FQDN frontend

```bash
FE_FQDN=$(az containerapp show \
  --name NOME_FRONTEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  --query properties.configuration.ingress.fqdn \
  -o tsv)

echo $FE_FQDN
```

## 23.3 Test traffico normale

```bash
curl https://$BE_FQDN/health
curl https://$FE_FQDN/health
curl https://$FE_FQDN/demo
```

## 23.4 Test traffico di errore

```bash
curl https://$FE_FQDN/demo-error
```

## 23.5 Generazione traffico ripetuto

```bash
for i in {1..20}; do curl -s https://$FE_FQDN/demo > /dev/null; done
for i in {1..10}; do curl -s https://$FE_FQDN/demo-error > /dev/null || true; done
```

---

# 24. Verifica nel portale Azure

Application Insights espone viste come **Performance**, **Failures**, **Search** e i dettagli end-to-end delle transazioni. Il modello consigliato da Microsoft per partire è proprio: creare la risorsa, configurare la connection string e aggiungere la distro OpenTelemetry. ([Microsoft Learn][1])

## 24.1 Application Insights Overview

* apri la risorsa **Application Insights**
* attendi alcuni minuti dopo il deploy
* verifica che arrivino richieste

## 24.2 Performance

* apri **Performance**
* osserva le operazioni del frontend e del backend

## 24.3 Failures

* genera traffico su `/demo-error`
* apri **Failures**
* verifica la comparsa delle richieste fallite

## 24.4 Search

* apri **Search**
* seleziona una richiesta del frontend
* osserva il dettaglio dell’operazione

## 24.5 Application Map

* apri **Application Map**
* verifica che frontend e backend compaiano come ruoli distinti

---

# 25. Query KQL su Application Insights / Logs

Microsoft documenta il mapping tra tabelle legacy e tabelle workspace-based. In questo laboratorio usiamo le tabelle workspace-based esplicite `AppRequests` e `AppDependencies`. Le query su tabelle Application Insights restano compatibili anche tra risorse classic e workspace-based, ma qui scegliamo i nomi espliciti per chiarezza. ([Microsoft Learn][4])

## 25.1 Richieste frontend recenti

```kql
AppRequests
| where AppRoleName == "frontend"
| order by TimeGenerated desc
| project TimeGenerated, Name, DurationMs, Success, ResultCode, OperationId
```

## 25.2 Richieste backend recenti

```kql
AppRequests
| where AppRoleName == "backend"
| order by TimeGenerated desc
| project TimeGenerated, Name, DurationMs, Success, ResultCode, OperationId
```

## 25.3 Dependency calls del frontend

```kql
AppDependencies
| where AppRoleName == "frontend"
| order by TimeGenerated desc
| project TimeGenerated, Target, DependencyType, DurationMs, Success, ResultCode, OperationId
```

## 25.4 Correlazione request → dependency tramite `OperationId`

```kql
AppDependencies
| where TimeGenerated > ago(1d)
| join kind=inner (
    AppRequests
    | where TimeGenerated > ago(1d)
) on OperationId
| project
    RequestTime = TimeGenerated1,
    RequestName = Name1,
    RequestDurationMs = DurationMs1,
    DependencyTarget = Target,
    DependencyDurationMs = DurationMs,
    Success,
    OperationId
| order by RequestTime desc
```

## 25.5 Richieste fallite

```kql
AppRequests
| where Success == false
| project TimeGenerated, AppRoleName, Name, ResultCode, DurationMs, OperationId
| order by TimeGenerated desc
```

---

# 26. Query KQL sui log di Azure Container Apps

Azure Container Apps invia nel workspace:

* i log console in `ContainerAppConsoleLogs_CL`
* i log di sistema in `ContainerAppSystemLogs_CL` ([Microsoft Learn][8])

## 26.1 Log console frontend

```kql
ContainerAppConsoleLogs_CL
| where ContainerAppName_s == "fe-obs-app"
| project TimeGenerated, ContainerAppName_s, RevisionName_s, ContainerName_s, Log_s
| order by TimeGenerated desc
| take 100
```

## 26.2 Log console backend

```kql
ContainerAppConsoleLogs_CL
| where ContainerAppName_s == "be-obs-app"
| project TimeGenerated, ContainerAppName_s, RevisionName_s, ContainerName_s, Log_s
| order by TimeGenerated desc
| take 100
```

## 26.3 Parse dei log JSON frontend

```kql
ContainerAppConsoleLogs_CL
| where ContainerAppName_s == "fe-obs-app"
| extend payload = parse_json(Log_s)
| project
    TimeGenerated,
    service = tostring(payload.service),
    path = tostring(payload.path),
    method = tostring(payload.method),
    status = toint(payload.status),
    latency_ms = todouble(payload.latency_ms),
    request_id = tostring(payload.request_id)
| order by TimeGenerated desc
```

## 26.4 Parse dei log JSON backend

```kql
ContainerAppConsoleLogs_CL
| where ContainerAppName_s == "be-obs-app"
| extend payload = parse_json(Log_s)
| project
    TimeGenerated,
    service = tostring(payload.service),
    path = tostring(payload.path),
    method = tostring(payload.method),
    status = toint(payload.status),
    latency_ms = todouble(payload.latency_ms),
    request_id = tostring(payload.request_id)
| order by TimeGenerated desc
```

## 26.5 Log di sistema

```kql
ContainerAppSystemLogs_CL
| where ContainerAppName_s in ("fe-obs-app", "be-obs-app")
| order by TimeGenerated desc
| take 100
```

---

# 27. Verifica revisioni e log da CLI

## 27.1 Revisioni backend

```bash
az containerapp revision list \
  --name NOME_BACKEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  -o table
```

## 27.2 Revisioni frontend

```bash
az containerapp revision list \
  --name NOME_FRONTEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  -o table
```

## 27.3 Log console frontend

```bash
az containerapp logs show \
  --name NOME_FRONTEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  --tail 20
```

## 27.4 Log console backend

```bash
az containerapp logs show \
  --name NOME_BACKEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  --tail 20
```

## 27.5 Log di sistema frontend

```bash
az containerapp logs show \
  --name NOME_FRONTEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  --type system \
  --tail 20
```

La CLI `az containerapp logs show` è il comando documentato per leggere i log di Container Apps, inclusi i log di sistema. ([Microsoft Learn][7])

---

# 28. Troubleshooting minimo

## 28.1 Nessuna telemetria in Application Insights

Possibili cause:

* `APPLICATIONINSIGHTS_CONNECTION_STRING` non presente
* package `azure-monitor-opentelemetry` mancante
* pipeline ha deployato un’immagine vecchia
* hai aspettato troppo poco

Verifica le env vars:

```bash
az containerapp show \
  --name NOME_FRONTEND_APP \
  --resource-group NOME_RESOURCE_GROUP \
  --query properties.template.containers[0].env
```

## 28.2 Il frontend risponde ma non vedi dependency verso backend

Possibili cause:

* `BACKEND_URL` errata
* backend non pronto
* telemetria backend non configurata correttamente

Verifica:

* `curl https://$FE_FQDN/demo`
* `curl https://$BE_FQDN/health`

## 28.3 Frontend e backend compaiono come un unico ruolo

Possibile causa:

* `SERVICE_NAME` non differenziato

Soluzione:

* frontend deve ricevere `SERVICE_NAME=frontend`
* backend deve ricevere `SERVICE_NAME=backend`

## 28.4 Non vedi log in Log Analytics

Possibili cause:

* stai interrogando la tabella sbagliata
* non hai generato traffico
* stai guardando troppo presto

Verifica con:

```kql
ContainerAppConsoleLogs_CL
| take 20
```

## 28.5 `az monitor app-insights component` non riconosciuto

Possibile causa:

* estensione `application-insights` non installata

Soluzione:

```bash
az extension add -n application-insights
```

I comandi `az monitor app-insights component ...` appartengono a quell’estensione. ([Microsoft Learn][10])

---

# 29. Evidenze richieste

Crea il file:

```bash
code docs/evidence_lab05.md
```

Inserisci questa struttura:

```md
# Evidence LAB05

## 1. Obiettivo compreso
Spiego con parole mie come cambia la observability del progetto passando dallo stack locale a Azure Monitor/Application Insights/Log Analytics.

## 2. Risorse usate
Indico:
- Resource Group
- Log Analytics Workspace
- Application Insights
- Container Apps Environment
- Frontend Container App
- Backend Container App

## 3. Modifiche al codice
Descrivo:
- modifiche ai requirements
- uso di configure_azure_monitor
- uso di service.name
- log JSON su stdout

## 4. Pipeline aggiornata
Descrivo le modifiche introdotte nel file azure-pipelines.yml.

## 5. Verifiche applicative
Incollo l’output di:
- GET /health backend
- GET /health frontend
- GET /demo frontend
- GET /demo-error frontend

## 6. Application Insights
Descrivo cosa ho osservato in:
- Performance
- Failures
- Search
- Application Map

## 7. Query KQL su Application Insights
Incollo almeno:
- query AppRequests frontend
- query AppDependencies frontend
- query join su OperationId

## 8. Query KQL su Container Apps
Incollo almeno:
- query ContainerAppConsoleLogs_CL frontend
- query ContainerAppConsoleLogs_CL backend
- query parse_json dei log JSON

## 9. Revisioni
Incollo:
- az containerapp revision list frontend
- az containerapp revision list backend

## 10. Problemi incontrati
Descrivo eventuali errori e come li ho risolti.
```

---

# 30. Consegna

Al termine del laboratorio devi consegnare:

* repository GitHub FE/BE aggiornato
* pipeline Azure DevOps aggiornata
* file `docs/evidence_lab05.md`

Salva il lavoro:

```bash
git add .
git commit -m "LAB05 completato - observability Azure con Application Insights e Log Analytics"
git push
```

---

# 31. Conclusione del laboratorio

In questo laboratorio hai chiuso davvero il cerchio del modulo:

* repository GitHub
* pipeline Azure DevOps
* immagini in ACR
* deploy FE/BE su Azure Container Apps
* telemetria applicativa con Application Insights
* query e log con Azure Monitor / Log Analytics

Hai quindi collegato:

* **delivery**
* **deploy**
* **telemetria**
* **query**
* **diagnostica**
* **verifica operativa**

Che, per una volta, è molto meglio che limitarsi a dire “la pipeline è verde quindi sarà tutto perfetto”, una delle superstizioni più longeve dell’IT moderno.

[1]: https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview "Application Insights OpenTelemetry observability overview - Azure Monitor | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/python/api/overview/azure/monitor-opentelemetry-readme?view=azure-python "Azure Monitor Opentelemetry Distro client library for Python | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/azure/azure-monitor/app/create-workspace-resource "Create and configure Application Insights resources - Azure Monitor | Microsoft Learn"
[4]: https://learn.microsoft.com/en-us/azure/azure-monitor/app/data-model-complete "Application Insights telemetry data model - Azure Monitor | Microsoft Learn"
[5]: https://learn.microsoft.com/en-us/azure/azure-monitor/logs/analyze-usage "Analyze usage in a Log Analytics workspace in Azure Monitor - Azure Monitor | Microsoft Learn"
[6]: https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/appdependencies?utm_source=chatgpt.com "Azure Monitor Logs reference - AppDependencies"
[7]: https://learn.microsoft.com/en-us/azure/container-apps/log-monitoring "Monitor logs in Azure Container Apps with Log Analytics | Microsoft Learn"
[8]: https://learn.microsoft.com/en-us/azure/container-apps/log-monitoring?utm_source=chatgpt.com "Monitor logs in Azure Container Apps with Log Analytics"
[9]: https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/containerappconsolelogs "Azure Monitor Logs reference - ContainerAppConsoleLogs - Azure Monitor | Microsoft Learn"
[10]: https://learn.microsoft.com/en-us/cli/azure/monitor/app-insights/component?view=azure-cli-latest "az monitor app-insights component | Microsoft Learn"
[11]: https://learn.microsoft.com/en-us/azure/container-apps/revisions-manage "Manage revisions in Azure Container Apps | Microsoft Learn"
