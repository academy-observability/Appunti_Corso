# LAB01 - Introduzione a DevOps e Azure DevOps con GitHub e app-v3.py

## Dal codice su GitHub al primo deploy cloud con Azure DevOps

---

# 1. Obiettivo del laboratorio

In questo laboratorio realizzerai il primo flusso DevOps completo del modulo usando:

- **GitHub** come repository del codice
- **Azure DevOps Pipelines** come motore CI/CD
- **Azure Container Registry (ACR)** come registry immagini
- **Azure Container Instances (ACI)** come primo target cloud semplice

Alla fine del laboratorio dovrai essere in grado di:

- pubblicare il codice su GitHub
- collegare GitHub ad Azure DevOps
- creare una pipeline YAML in Azure DevOps che legge da GitHub
- costruire una immagine Docker
- pubblicare l’immagine su ACR
- fare un primo deploy su ACI
- verificare il funzionamento dell’applicazione

---

# 2. Perché questo laboratorio esiste

Dopo il LAB00, in cui hai visto una applicazione distribuita e uno stack di observability in locale, qui facciamo il primo passo DevOps vero e proprio.

Per evitare di introdurre subito troppa complessità, non usiamo ancora il progetto FE/BE completo.  
Usiamo una sola applicazione, derivata da `app-v3.py`, e costruiamo il primo filo completo:

**codice -> repository -> pipeline -> immagine -> registry -> deploy**. :contentReference[oaicite:2]{index=2}

Questo laboratorio ha quindi un obiettivo preciso: far capire bene il flusso, prima di passare a pipeline più articolate e poi al progetto distribuito reale.

---

# Parte 1 - Spiegazione dei termini e degli strumenti

# 3. Flusso architetturale del laboratorio

Il flusso che userai in questo laboratorio è il seguente:

```text
Sviluppatore
   |
   v
GitHub Repository
   |
   v
Azure DevOps Pipeline
   |
   +--> docker build
   +--> push immagine su ACR
   +--> deploy su ACI
   |
   v
Applicazione raggiungibile via HTTP
````

Questa è la prima forma completa di pipeline del modulo.
Il codice vive in **GitHub**, la pipeline gira in **Azure DevOps**, l’immagine viene salvata in **ACR**, e il container viene eseguito in **ACI**. Azure Pipelines supporta repository GitHub come sorgente della pipeline, ACR è il registry usato per archiviare immagini container, e ACI è un servizio serverless per eseguire container senza dover gestire VM o orchestratori più complessi. ([Microsoft Learn][1])

---

# 4. Che ruolo ha GitHub in questo laboratorio

In questo laboratorio **GitHub** è il repository del codice sorgente.

Questo significa che GitHub conterrà:

* file Python dell’applicazione
* `Dockerfile`
* `requirements.txt`
* file YAML della pipeline
* documentazione del laboratorio

GitHub, in questo contesto, **non** è:

* il registry immagini
* il target di deploy
* il motore della pipeline

Azure Pipelines può leggere il repository GitHub, eseguire la build e validare automaticamente commit e pull request. Inoltre, quando si configura la connessione con GitHub, Azure DevOps raccomanda l’uso della **GitHub App** come metodo di autenticazione preferito per le pipeline CI. ([Microsoft Learn][1])

---

# 5. Che ruolo ha Azure DevOps in questo laboratorio

In questo percorso useremo Azure DevOps soprattutto per **Azure Pipelines**.

Azure Pipelines è il servizio che automatizza build, test e deploy. Nel nostro LAB01 si occuperà di:

* leggere il codice dal repository GitHub
* costruire l’immagine Docker
* pubblicarla su ACR
* creare o aggiornare un container in ACI

Per eseguire comandi Azure nella pipeline useremo il task **AzureCLI@2**, che è il task documentato per eseguire script Bash o PowerShell contro Azure tramite una Azure Resource Manager service connection. ([Microsoft Learn][2])

---

# 6. Che cos’è una service connection

Per collegare Azure DevOps a servizi esterni servono delle connessioni di servizio.

In questo laboratorio userai due connessioni logiche:

## 6.1 GitHub connection

Serve a permettere alla pipeline Azure DevOps di accedere al repository GitHub.

Azure DevOps può autenticarsi verso GitHub in più modi, ma la documentazione indica la **GitHub App** come metodo raccomandato per le pipeline CI, perché evita di basarsi sulla tua identità personale e supporta anche GitHub Checks. Per poterla usare servono i permessi corretti sul repository o sull’organizzazione GitHub. ([Microsoft Learn][1])

## 6.2 Azure Resource Manager service connection

Serve a permettere alla pipeline di autenticarsi verso la subscription Azure per eseguire:

* login ad ACR
* push delle immagini
* deploy su ACI

Questa connessione sarà usata dentro il task `AzureCLI@2`. ([Microsoft Learn][2])

---

# 7. Che cos’è Azure Container Registry (ACR)

**Azure Container Registry** è il registry immagini del laboratorio.

Qui finirà l’immagine Docker costruita dalla pipeline.
Il flusso tipico è:

* `docker build`
* `docker tag`
* `docker push`

ACR usa nomi completi del tipo:

```text
nomeregistry.azurecr.io/nome-immagine:tag
```

La documentazione ufficiale mostra proprio questo modello per push e pull delle immagini. Inoltre l’account admin di ACR esiste ma è **disabilitato di default**; Microsoft raccomanda di abilitarlo solo quando necessario. In questo primo laboratorio lo useremo come semplificazione didattica, sapendo che non è il modello migliore per scenari più maturi. ([Microsoft Learn][3])

---

# 8. Che cos’è Azure Container Instances (ACI)

**Azure Container Instances** è il primo target cloud del laboratorio.

Lo usiamo qui perché è un modo semplice e diretto per eseguire un container nel cloud senza introdurre ancora piattaforme più avanzate.

ACI è utile in questo laboratorio perché permette di:

* creare rapidamente un container
* esporre una porta HTTP pubblica
* verificare il funzionamento dell’app
* concentrarsi sul flusso DevOps senza entrare subito nella complessità di un ambiente multi-servizio

Per distribuire immagini private da ACR ad ACI servono credenziali del registry oppure altri metodi di autenticazione più robusti, come un service principal. In questo laboratorio useremo le credenziali del registry per semplificare i passaggi. ([Microsoft Learn][4])

---

# 9. Che cos’è una pipeline in questo laboratorio

In questo laboratorio una pipeline è una sequenza automatizzata di passaggi che fa queste cose:

1. legge il codice da GitHub
2. costruisce l’immagine Docker
3. pubblica l’immagine in ACR
4. distribuisce l’immagine in ACI
5. restituisce un risultato verificabile

La pipeline elimina gran parte del lavoro manuale e rende il processo ripetibile.
È il primo gradino del modulo. Nei laboratori successivi la pipeline diventerà più leggibile, più strutturata e poi capace di gestire più servizi.

---

# 10. Perché usiamo `app-v3.py`

`app-v3.py` è una base didattica molto buona per LAB01 perché:

* ha endpoint semplici
* produce log JSON su stdout
* include `request_id`
* include un campo `status`
* permette test rapidi con `curl`
* sarà riutile anche nei passaggi successivi del modulo observability 

Non è ancora una applicazione distribuita, ed è proprio il suo vantaggio in questo momento: ci permette di concentrare l’attenzione sul flusso DevOps senza introdurre subito frontend e backend separati.

---

# 11. Concetti chiave da ricordare prima della parte pratica

Prima di passare allo step-by-step, devi avere chiaro questo:

* **GitHub** conserva il codice
* **Azure DevOps** esegue la pipeline
* **ACR** conserva l’immagine
* **ACI** esegue il container
* la pipeline collega questi passaggi in modo automatizzato

Se confondi questi ruoli, il laboratorio diventa una sequenza meccanica di comandi. E quello, francamente, sarebbe un ottimo modo per imparare poco.

---

# Parte 2 - Step-by-step guidato

# 12. Prerequisiti del laboratorio

Per eseguire LAB01 devono essere già disponibili:

* un account GitHub con permessi sul repository del laboratorio
* una organizzazione Azure DevOps
* un progetto Azure DevOps già creato
* una subscription Azure corretta
* un Resource Group già esistente
* un Azure Container Registry già esistente
* Azure CLI installata e funzionante
* Docker installato e funzionante
* Git installato
* VS Code o editor equivalente

### Nota pratica importante

Per la parte GitHub, la connessione più consigliata è tramite **GitHub App**. Per configurarla correttamente devi avere i permessi adeguati sul repository o sull’organizzazione GitHub. Se il repository non è tuo, devi essere collaboratore o membro del team corretto. ([Microsoft Learn][1])

---

# 13. Cartella di lavoro locale

Apri il terminale WSL e crea la cartella di lavoro:

```bash
mkdir -p ~/corso_obs/lab01-github-appv3/src
cd ~/corso_obs/lab01-github-appv3
```

Verifica la cartella corrente:

```bash
pwd
ls
```

---

# 14. Creazione dei file applicativi

## 14.1 File `src/app.py`

Crea il file:

```bash
code src/app.py
```

Inserisci questo contenuto:

```python
import json
import os
import time
import uuid
from datetime import datetime, timezone

from flask import Flask, g, jsonify, request
from werkzeug.exceptions import HTTPException

app = Flask(__name__)


def utc_now_iso() -> str:
    return datetime.now(timezone.utc).isoformat()


def write_log(status_code: int, message: str = "request_completed") -> None:
    latency_ms = round((time.perf_counter() - g.start_time) * 1000, 2)

    log_record = {
        "timestamp": utc_now_iso(),
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

    print(json.dumps(log_record), flush=True)


@app.before_request
def before_request() -> None:
    g.start_time = time.perf_counter()
    g.request_id = request.headers.get("X-Request-Id", str(uuid.uuid4()))


@app.after_request
def after_request(response):
    write_log(response.status_code)
    response.headers["X-Request-Id"] = g.request_id
    return response


@app.errorhandler(HTTPException)
def handle_http_exception(exc: HTTPException):
    response = jsonify(
        {
            "error": exc.name.lower().replace(" ", "_"),
            "status": exc.code,
            "path": request.path,
        }
    )
    response.status_code = exc.code
    return response


@app.errorhandler(Exception)
def handle_generic_exception(exc: Exception):
    response = jsonify(
        {
            "error": "internal_server_error",
            "status": 500,
            "path": request.path,
        }
    )
    response.status_code = 500
    return response


@app.get("/")
def home():
    return jsonify(
        {
            "app": "obsapp",
            "version": "v3",
            "status": "running",
            "timestamp": utc_now_iso(),
        }
    ), 200


@app.get("/health")
def health():
    return jsonify({"status": "ok", "timestamp": utc_now_iso()}), 200


@app.get("/time")
def current_time():
    return jsonify({"time": utc_now_iso()}), 200


@app.post("/echo")
def echo():
    payload = request.get_json(silent=True)
    if payload is None:
        return jsonify({"error": "invalid_json", "status": 400}), 400
    return jsonify({"received": payload, "status": 200}), 200


@app.get("/error")
def simulated_error():
    return jsonify({"error": "simulated_error", "status": 500}), 500


if __name__ == "__main__":
    port = int(os.getenv("PORT", "8000"))
    app.run(host="0.0.0.0", port=port)
```

Questo codice è coerente con la versione `app-v3.py` già usata nel progetto. 

---

## 14.2 File `requirements.txt`

Crea il file:

```bash
code requirements.txt
```

Inserisci:

```txt
flask==3.0.0
```

---

## 14.3 File `Dockerfile`

Crea il file:

```bash
code Dockerfile
```

Inserisci:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/app.py /app/app.py

EXPOSE 8000

ENV PORT=8000

CMD ["python", "app.py"]
```

---

## 14.4 File `.dockerignore`

Crea il file:

```bash
code .dockerignore
```

Inserisci:

```txt
.git
.gitignore
__pycache__/
*.pyc
*.pyo
*.pyd
.venv/
venv/
.env
docs/
```

---

# 15. Test locale rapido dell’applicazione

Crea l’ambiente virtuale e installa le dipendenze:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python src/app.py
```

Apri un secondo terminale e verifica:

```bash
curl http://127.0.0.1:8000/
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8000/time
curl http://127.0.0.1:8000/error
curl -X POST http://127.0.0.1:8000/echo -H 'Content-Type: application/json' -d '{"hello":"world"}'
```

Interrompi il server con `CTRL+C`.

---

# 16. Creazione del repository GitHub

Crea su GitHub un nuovo repository, ad esempio:

* `app-v3-devops`

Poi inizializza Git locale e collega il repository remoto:

```bash
cd ~/corso_obs/lab01-github-appv3
git init
git branch -M main
git add .
git commit -m "Initial commit - app-v3 for Azure DevOps with GitHub"
git remote add origin URL-DEL-TUO-REPO-GITHUB
git push -u origin main
```

Verifica che i file siano presenti su GitHub.

---

# 17. Preparazione delle risorse Azure

## 17.1 Login Azure

```bash
az login
az account show
```

## 17.2 Verifica Resource Group

```bash
az group show --name NOME_RESOURCE_GROUP
```

## 17.3 Verifica Azure Container Registry

```bash
az acr show --name NOME_ACR --resource-group NOME_RESOURCE_GROUP
```

## 17.4 Abilitazione admin user ACR per il laboratorio

Per semplicità didattica, in questo laboratorio usiamo l’admin account di ACR.

```bash
az acr update --name NOME_ACR --admin-enabled true
az acr credential show --name NOME_ACR
```

L’admin account di ACR è disabilitato di default e non è la soluzione consigliata per scenari più maturi, ma per questo primo laboratorio semplifica molto il deploy su ACI. ([Microsoft Learn][5])

---

# 18. Configurazione delle connessioni in Azure DevOps

## 18.1 GitHub connection

Nel progetto Azure DevOps:

* vai in **Project settings**
* apri **Service connections**
* scegli **New service connection**
* seleziona **GitHub**
* autorizza l’accesso al repository GitHub del laboratorio

Azure DevOps usa la GitHub App come metodo preferito quando disponibile. Per installarla servono i permessi giusti sul repository o sull’organizzazione. ([Microsoft Learn][1])

## 18.2 Azure Resource Manager service connection

Sempre in **Service connections**:

* scegli **New service connection**
* seleziona **Azure Resource Manager**
* collega la subscription o il Resource Group corretto
* assegna un nome chiaro, ad esempio:

```text
sc-obs-azure-rg
```

Questa connessione verrà usata da `AzureCLI@2`. ([Microsoft Learn][2])

---

# 19. Creazione del file pipeline

Crea nella radice del repository il file:

```bash
code azure-pipelines.yml
```

Inserisci questo contenuto, sostituendo i valori con quelli reali del tuo ambiente:

```yaml
trigger:
  - main

pr: none

pool:
  vmImage: ubuntu-latest

variables:
  azureServiceConnection: 'sc-obs-azure-rg'
  resourceGroupName: 'rg-observability-dev'
  acrName: 'nomeregistry'
  imageName: 'obsapp-v3'
  aciName: 'obsapp-v3-aci'
  aciDnsName: 'obsappv3-NOMEUNIVOCO'
  location: 'westeurope'
  containerPort: '8000'

stages:
- stage: BuildAndDeploy
  displayName: Build, Push and Deploy
  jobs:
  - job: BuildPushDeploy
    displayName: Build image, push to ACR, deploy to ACI
    steps:
    - checkout: self

    - task: AzureCLI@2
      displayName: Build image, push to ACR and deploy to ACI
      inputs:
        azureSubscription: '$(azureServiceConnection)'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          set -e

          echo "Verifica subscription attiva"
          az account show

          echo "Login ad ACR"
          az acr login --name $(acrName)

          IMAGE_TAG=$(Build.BuildId)
          FULL_IMAGE_NAME="$(acrName).azurecr.io/$(imageName):${IMAGE_TAG}"

          echo "Build immagine ${FULL_IMAGE_NAME}"
          docker build -t ${FULL_IMAGE_NAME} .

          echo "Push immagine ${FULL_IMAGE_NAME}"
          docker push ${FULL_IMAGE_NAME}

          echo "Recupero credenziali ACR"
          ACR_USER=$(az acr credential show --name $(acrName) --query username -o tsv)
          ACR_PASS=$(az acr credential show --name $(acrName) --query "passwords[0].value" -o tsv)

          echo "Elimino eventuale container precedente"
          az container delete \
            --resource-group $(resourceGroupName) \
            --name $(aciName) \
            --yes || true

          echo "Creo nuovo container su ACI"
          az container create \
            --resource-group $(resourceGroupName) \
            --name $(aciName) \
            --image ${FULL_IMAGE_NAME} \
            --cpu 1 \
            --memory 1.5 \
            --ports $(containerPort) \
            --ip-address Public \
            --dns-name-label $(aciDnsName) \
            --location $(location) \
            --registry-login-server $(acrName).azurecr.io \
            --registry-username ${ACR_USER} \
            --registry-password ${ACR_PASS} \
            --environment-variables PORT=$(containerPort)

          echo "Recupero FQDN"
          FQDN=$(az container show \
            --resource-group $(resourceGroupName) \
            --name $(aciName) \
            --query ipAddress.fqdn \
            -o tsv)

          echo "Applicazione disponibile su: http://${FQDN}:$(containerPort)"
```

### Perché questa pipeline va bene per LAB01

Questa pipeline fa tutto in un solo stage, che è esattamente il livello di semplicità giusto per il primo laboratorio:

* legge il codice da GitHub
* costruisce l’immagine
* la pubblica in ACR
* distribuisce il container in ACI

Nei laboratori successivi la renderemo più leggibile e meglio organizzata.

---

# 20. Pubblicazione del file pipeline su GitHub

```bash
git add .
git commit -m "Add Azure DevOps pipeline for LAB01"
git push
```

Verifica che `azure-pipelines.yml` sia visibile nel repository GitHub.

---

# 21. Creazione della pipeline in Azure DevOps

Nel portale Azure DevOps:

1. vai in **Pipelines**
2. scegli **Create Pipeline**
3. seleziona **GitHub**
4. autorizza l’accesso se richiesto
5. seleziona il repository GitHub del laboratorio
6. scegli il file `azure-pipelines.yml`
7. salva ed esegui

Azure DevOps supporta questo flusso in modo nativo: la pipeline viene creata scegliendo prima il repository GitHub e poi il file YAML in quel repository. ([Microsoft Learn][1])

---

# 22. Verifica dell’esecuzione pipeline

Durante l’esecuzione della pipeline controlla che compaiano questi passaggi:

* checkout del repository GitHub
* `az account show`
* `az acr login`
* `docker build`
* `docker push`
* `az container create`

Se tutto va bene, il run termina con stato **Succeeded**.

---

# 23. Verifica applicativa finale

## 23.1 Recupero FQDN

Se non hai già letto l’FQDN nei log della pipeline, recuperalo così:

```bash
az container show \
  --resource-group NOME_RESOURCE_GROUP \
  --name NOME_ACI \
  --query ipAddress.fqdn \
  -o tsv
```

## 23.2 Test endpoint

Sostituisci `FQDN-ACI` con il valore reale.

```bash
curl http://FQDN-ACI:8000/
curl http://FQDN-ACI:8000/health
curl http://FQDN-ACI:8000/time
curl -X POST http://FQDN-ACI:8000/echo -H 'Content-Type: application/json' -d '{"lab":"01"}'
curl http://FQDN-ACI:8000/error
```

---

# 24. Verifica dei log del container

Leggi i log del container:

```bash
az container logs \
  --resource-group NOME_RESOURCE_GROUP \
  --name NOME_ACI
```

Cosa devi osservare:

* righe JSON
* campo `request_id`
* campo `status`
* campo `path`
* campo `latency_ms`

Questo è utile perché l’app è già scritta in modo da produrre log strutturati riusabili nei laboratori successivi. 

---

# 25. Troubleshooting minimo

## 25.1 Il repository GitHub non compare in Azure DevOps

Possibili cause:

* GitHub App non installata correttamente
* permessi insufficienti sul repository
* non sei collaboratore del repository
* la connessione GitHub non è stata autorizzata

Azure DevOps richiede permessi adeguati sul repository GitHub e, con GitHub App, l’installazione corretta dell’app stessa nell’account o nell’organizzazione. ([Microsoft Learn][1])

## 25.2 La pipeline fallisce su `az acr login`

Possibili cause:

* nome ACR errato
* Resource Group errato
* permessi insufficienti nella service connection Azure

Verifica:

```bash
az acr show --name NOME_ACR --resource-group NOME_RG
```

## 25.3 La pipeline fallisce su `az acr credential show`

Possibile causa:

* admin account ACR non abilitato

Verifica:

```bash
az acr credential show --name NOME_ACR
```

Se non funziona:

```bash
az acr update --name NOME_ACR --admin-enabled true
```

## 25.4 Il deploy ACI fallisce

Possibili cause:

* DNS label non univoco
* immagine non disponibile nel registry
* credenziali ACR errate

Cambia il valore di:

```text
aciDnsName
```

nella pipeline e rilancia.

## 25.5 L’app è deployata ma non risponde

Possibili cause:

* container ancora in avvio
* porta errata
* errore runtime

Verifica stato e log:

```bash
az container show \
  --resource-group NOME_RG \
  --name NOME_ACI \
  --query instanceView.state

az container logs \
  --resource-group NOME_RG \
  --name NOME_ACI
```

---
**25.6 🔴 NUOVO: La pipeline fallisce con "No hosted parallelism"**
**Cosa significa:**
Error: No hosted parallelism has been purchased

**Causa:**
I nuovi account Azure DevOps non hanno accesso gratuito ai server Microsoft per eseguire le pipeline. È una limitazione di Microsoft, non un errore tuo.

**Possibili Soluzioni**

- Attendere approvazione Microsoft24-48H
- Deploy manuale (vedi 25.6.1)
- MediaAgente self-hosted (vedi 25.6.2)

 **25.6.1 ✅ SOLUZIONE RAPIDA: Deploy manuale**

 Se non vuoi aspettare Microsoft, puoi eseguire manualmente i comandi che la pipeline eseguirebbe.

- Passaggi 1-5
- Passo 1: Login al registry
- az acr login --name TUO_NOME_ACR
- Passo 2: Build immagine
- docker build -t TUO_NOME_ACR.azurecr.io/obsapp-v3:latest .
- Passo 3: Push su ACR
- docker push TUO_NOME_ACR.azurecr.io/obsapp-v3:latest
- Passo 4: Recupera credenziali (copiale, ti serviranno dopo)
- ACR_USER=$(az acr credential show --name TUO_NOME_ACR --query username -o tsv)
- ACR_PASS=$(az acr credential show --name TUO_NOME_ACR --query "passwords[0].value" -o tsv)
- Passo 5: Deploy su ACI (sostituisci i valori)
- az container create \
  - --resource-group TUO_RESOURCE_GROUP \
  - --name obsapp-v3-aci \
  - --image TUO_NOME_ACR.azurecr.io/obsapp-v3:latest \
  - --os-type Linux \
  - --ports 8000 \
  - --ip-address Public \
  - --dns-name-label obsappv3-TUONOME \
  - --location westeurope \
  - --registry-login-server TUO_NOME_ACR.azurecr.io \
  - --registry-username ${ACR_USER} \
  - --registry-password ${ACR_PASS} \
  - --environment-variables PORT=8000

**RISULTATO ATTESO**
Se tutto funziona vedrai:
Application can be accessed at: http://<IP_PUBBLICO>:8000
**Nota importante da esperienza:**
Se ricevi errore **InvalidOsType**, aggiungi --**os-type Linux** al comando az container create (vedi già fatto nell'esempio sopra).

**25.6.2 🚀 SOLUZIONE DEFINITIVA: Agente self-hosted su WSL**
**Personal Access Token - PAT**
- Vai su Azure DevOps
- Personal Access Tokens
- Clicca + New Token
- Nome: scegli un mome . Scadenza: 30 giorni.
- Scopes (Permessi): Seleziona Full Access.
- Clicca Create e COPIA il codice (te lo mostra una sola volta-quindi copialo)

  **installare l'agent su wsl**
- Scarichiamo e configuriamo il programma che eseguirà i lavori.
- Perché serve: È il software che materialmente scarica il codice e crea i container.
- Apri il terminale WSL e scrivi questi comandi uno alla volta:
- Crea la cartella:
- mkdir agente-lab && cd agente-lab
- Estrai i file:
- tar zxvf vsts-agent-linux-x64-3.225.0.tar.gz
- Configura (Segui l'esempio):
- ./config.sh
  **Quando il terminale ti fa le domande, rispondi così:**


- Server URL:  ([quello che vedi nella barra del browser](https://dev.azure.com/SOLOILNOMEDELLATUAORGANITATION)).
- Authentication type: Premi Invio.
- Personal access token: Incolla la chiave creata al punto 1.
- Agent Pool: premi Invio.
- Agent Name: Scrivi mio-agente-wsl.
- Work folder: Premi Invio.


- 
# 26. Evidenze richieste

Crea il file:

```bash
mkdir -p docs
code docs/evidence_lab01.md
```

Inserisci questa struttura:

```md
# Evidence LAB01

## 1. Obiettivo compreso
Spiego con parole mie il flusso:
GitHub -> Azure DevOps -> ACR -> ACI

## 2. Repository GitHub
Indico il nome del repository usato e il branch principale.

## 3. Azure DevOps
Indico:
- nome del progetto Azure DevOps
- nome della pipeline
- tipo di connessione usata verso GitHub
- nome della Azure Resource Manager service connection

## 4. Build e push immagine
Spiego quale immagine è stata costruita e con quale tag.

## 5. Deploy su ACI
Indico:
- Resource Group
- nome del container
- FQDN finale

## 6. Test applicativi
Incollo l’output di:
- GET /
- GET /health
- GET /time
- POST /echo
- GET /error

## 7. Log del container
Incollo alcune righe JSON e commento i campi principali.

## 8. Problemi incontrati
Descrivo eventuali errori e come li ho risolti.
```

---

# 27. Consegna

Al termine del laboratorio devi consegnare:

* repository GitHub completo
* pipeline Azure DevOps funzionante
* file `docs/evidence_lab01.md`

Salva il lavoro:

```bash
git add .
git commit -m "LAB01 completato - primo flusso GitHub Azure DevOps ACR ACI"
git push
```

---

# 28. Conclusione del laboratorio

In questo laboratorio hai costruito il primo flusso DevOps completo del modulo:

* codice su GitHub
* pipeline su Azure DevOps
* immagine su ACR
* deploy su ACI
* verifica finale applicativa

È il primo gradino corretto.

Nel prossimo laboratorio manterrai la stessa app, ma migliorerai la pipeline rendendola più leggibile e più vicina a un flusso CI/CD disciplinato, con separazione tra build, deploy e smoke test.



[1]: https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/github?view=azure-devops "Build GitHub repositories - Azure Pipelines | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/azure-cli-v2?view=azure-pipelines "AzureCLI@2 - Azure CLI v2 task | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-get-started-docker-cli "Push & Pull Container Image using Azure Container Registry - Azure Container Registry | Microsoft Learn"
[4]: https://learn.microsoft.com/en-us/azure/container-instances/container-instances-using-azure-container-registry "Deploy container image from Azure Container Registry using a service principal - Azure Container Instances | Microsoft Learn"
[5]: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication "Azure Container Registry Authentication Options Explained - Azure Container Registry | Microsoft Learn"
