# LAB12 Standalone - Alerting completo da zero con ACI, ACR, Log Analytics e Alert Rule

## Obiettivo del laboratorio

Questo laboratorio è una versione **standalone completa** del LAB12.

È pensato per chi ha eliminato le risorse Azure dei laboratori precedenti oppure non possiede più il file `app.py`.

Alla fine del laboratorio avrai costruito da zero un ambiente minimo ma completo con:

- nuovo **Resource Group**
- nuovo **Log Analytics Workspace**
- nuova applicazione Python/Flask osservabile
- nuova immagine container
- nuovo **Azure Container Registry (ACR)**
- nuovo deploy su **Azure Container Instances (ACI)**
- verifica dei log in **Log Analytics**
- query KQL per **error_rate**
- **Action Group**
- **Alert Rule** su query KQL
- simulazione errori e verifica dello stato **Fired**

---

## Durata stimata

**4,5 - 5 ore**

---

## Obiettivi didattici

Al termine del laboratorio saprai:

- creare da zero le risorse minime necessarie per osservare un container su Azure
- preparare una semplice app osservabile con log JSON su stdout
- costruire e pubblicare un’immagine container in ACR
- distribuire un container in ACI collegato a Log Analytics
- verificare il formato reale dei log prima di scrivere query KQL
- calcolare `error_rate` con KQL
- creare un Action Group e una Alert Rule
- provocare volutamente un degrado e verificare l’alert

---

## Dove operare

### WSL Ubuntu

Userai WSL per:

- creare i file locali
- usare Azure CLI
- costruire l’immagine Docker
- inviare richieste HTTP all’applicazione
- raccogliere evidenze

### Azure Portal

Userai il portale per:

- controllare le risorse create
- aprire Log Analytics
- creare Action Group
- creare Alert Rule
- verificare Fired alerts

---

## Prerequisiti

Sono richiesti:

- account Azure con sottoscrizione attiva
- Azure CLI autenticata (`az login`)
- Docker Desktop o Docker Engine funzionante
- WSL Ubuntu
- permessi sufficienti per creare RG, ACR, ACI, Log Analytics e alert

---

# PARTE 1 - Architettura del laboratorio

## 1. Architettura logica

```text
app-v3.py
  ↓
Docker image
  ↓
Azure Container Registry
  ↓
Azure Container Instance
  ↓
stdout JSON logs
  ↓
Log Analytics Workspace
  ↓
KQL
  ↓
Alert Rule
  ↓
Action Group
  ↓
Fired
```

---

## 2. Perché questa versione è standalone

Non richiede nessuna risorsa dei lab precedenti.

Ricostruisce tutto ciò che serve per arrivare all’alerting:

- codice applicativo
- immagine container
- registro immagini
- container in esecuzione
- raccolta log
- query KQL
- alerting

---

## 3. Scelte progettuali del laboratorio

Per rendere il laboratorio più robusto:

- i log applicativi sono **JSON strutturati**
- il campo `status` è presente in ogni riga di log applicativo
- la query KQL usa `parse_json(Message)`
- il calcolo dell’`error_rate` viene filtrato sul container group del laboratorio
- l’alert viene creato solo dopo verifica del dataset reale

---

# PARTE 2 - Laboratorio guidato passo-passo

## Step 1 - Crea la cartella locale del laboratorio

In WSL Ubuntu esegui:

```bash
mkdir -p ~/course/lab12-standalone
cd ~/course/lab12-standalone
script -a cmdlog_lab12_standalone.txt
mkdir -p src docs
```

### Che cosa stai facendo

- crei la cartella locale del laboratorio
- avvii la registrazione della sessione terminale
- prepari le sottocartelle per codice ed evidenze

---

## Step 2 - Definisci le variabili di ambiente del laboratorio

Esegui:

```bash
LOCATION="westeurope"
RG="rg-observability-lab12-standalone"
LAW_NAME="law-observability-lab12"
ACR_NAME="obsacr$(openssl rand -hex 3)"
ACI_NAME="obsapp-aci"
IMAGE_NAME="obsapp"
IMAGE_TAG="v3"
PORT="8000"
```

Verifica i valori:

```bash
echo "$LOCATION"
echo "$RG"
echo "$LAW_NAME"
echo "$ACR_NAME"
echo "$ACI_NAME"
```

### Nota importante

Il nome di **ACR** deve essere globale, univoco, tutto minuscolo e composto solo da caratteri alfanumerici.

---

## Step 3 - Accedi ad Azure e verifica la sottoscrizione

Esegui:

```bash
az login
az account show --output table
```

Se necessario, imposta la sottoscrizione corretta:

```bash
az account set --subscription "<SUBSCRIPTION_ID_O_NOME>"
```

---

## Step 4 - Crea il Resource Group

Esegui:

```bash
az group create --name "$RG" --location "$LOCATION"
```

### Checkpoint

Il Resource Group deve risultare creato correttamente.

---

## Step 5 - Crea il Log Analytics Workspace

Esegui:

```bash
az monitor log-analytics workspace create \
  --resource-group "$RG" \
  --workspace-name "$LAW_NAME" \
  --location "$LOCATION"
```

Recupera Workspace ID e Shared Key:

```bash
LAW_ID=$(az monitor log-analytics workspace show \
  --resource-group "$RG" \
  --workspace-name "$LAW_NAME" \
  --query customerId -o tsv)

LAW_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --resource-group "$RG" \
  --workspace-name "$LAW_NAME" \
  --query primarySharedKey -o tsv)


echo "$LAW_ID"
echo "$LAW_KEY" | cut -c1-8
```

### Checkpoint

Devi avere:

- workspace creato
- workspace id disponibile
- shared key disponibile

---

## Step 6 - Crea il file `src/app.py`

Apri il file:

```bash
nano src/app.py
```

Incolla questo contenuto:

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

Salva il file.

---

## Step 7 - Crea `src/requirements.txt`

Apri il file:

```bash
nano src/requirements.txt
```

Incolla:

```text
Flask==3.0.3
Werkzeug==3.0.3
```

Salva il file.

---

## Step 8 - Crea il `Dockerfile`

Apri il file:

```bash
nano src/Dockerfile
```

Incolla:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

ENV PORT=8000
EXPOSE 8000

CMD ["python", "app.py"]
```

Salva il file.

---

## Step 9 - Test locale rapido facoltativo

Entra nella cartella del codice:

```bash
cd ~/course/lab12-standalone/src
```

Crea un ambiente virtuale e installa le dipendenze:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Avvia l’app:

```bash
python app.py
```

In un secondo terminale, esegui:

```bash
curl http://127.0.0.1:8000/
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8000/time
curl http://127.0.0.1:8000/nope
curl http://127.0.0.1:8000/error
curl -X POST http://127.0.0.1:8000/echo -H 'Content-Type: application/json' -d '{"hello":"world"}'
```

Interrompi poi l’app con `Ctrl+C`.

---

## Step 10 - Crea Azure Container Registry

Torna nella root del laboratorio:

```bash
cd ~/course/lab12-standalone
```

Crea ACR:

```bash
az acr create \
  --resource-group "$RG" \
  --name "$ACR_NAME" \
  --sku Basic
```

Abilita l’admin user:

```bash
az acr update --name "$ACR_NAME" --admin-enabled true
```

Recupera informazioni del registry:

```bash
ACR_LOGIN_SERVER=$(az acr show --name "$ACR_NAME" --query loginServer -o tsv)
ACR_USERNAME=$(az acr credential show --name "$ACR_NAME" --query username -o tsv)
ACR_PASSWORD=$(az acr credential show --name "$ACR_NAME" --query 'passwords[0].value' -o tsv)


echo "$ACR_LOGIN_SERVER"
echo "$ACR_USERNAME"
echo "$ACR_PASSWORD" | cut -c1-8
```

### Checkpoint

Devi avere:

- registry creato
- login server disponibile
- username e password disponibili

---

## Step 11 - Login al registry e build dell’immagine

Esegui:

```bash
az acr login --name "$ACR_NAME"
cd ~/course/lab12-standalone/src

docker build -t "$ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG" .
```

### Verifica immagine locale

```bash
docker images | grep "$IMAGE_NAME"
```

---

## Step 12 - Push dell’immagine su ACR

Esegui:

```bash
docker push "$ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG"
```

Verifica il repository:

```bash
az acr repository list --name "$ACR_NAME" --output table
az acr repository show-tags --name "$ACR_NAME" --repository "$IMAGE_NAME" --output table
```

### Checkpoint

Il repository `obsapp` con tag `v3` deve risultare presente in ACR.

---

## Step 13 - Deploy dell’app su Azure Container Instances

Esegui:

```bash
az container create \
  --resource-group "$RG" \
  --name "$ACI_NAME" \
  --image "$ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG" \
  --registry-login-server "$ACR_LOGIN_SERVER" \
  --registry-username "$ACR_USERNAME" \
  --registry-password "$ACR_PASSWORD" \
  --cpu 1 \
  --memory 1.5 \
  --os-type Linux \
  --ip-address Public \
  --ports 8000 \
  --environment-variables PORT=8000 \
  --log-analytics-workspace "$LAW_NAME" \
  --log-analytics-workspace-key "$LAW_KEY"
```

### Checkpoint

La container instance deve risultare creata correttamente.

---

<details>
### RETTIFICA VARIABILE D'AMBIENTE PER LA CREAZIONE DEL CONTAINER:

ESEGUIAMO QUESTO COMANDO PER ESTRAPOLARE IL LAW_ID:
```
LAW_ID=$(az monitor log-analytics workspace show \
  --resource-group "$RG" \
  --workspace-name "$LAW_NAME" \
  --query customerId -o tsv)
```
ESEGUIAMO NUOVAMENTE IL CONTAINER CREATE CON LA VARIABILE NUOVA:

```bash
az container create \
  --resource-group "$RG" \
  --name "$ACI_NAME" \
  --image "$ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG" \
  --registry-login-server "$ACR_LOGIN_SERVER" \
  --registry-username "$ACR_USERNAME" \
  --registry-password "$ACR_PASSWORD" \
  --cpu 1 \
  --memory 1.5 \
  --os-type Linux \
  --ip-address Public \
  --ports 8000 \
  --environment-variables PORT=8000 \
  --log-analytics-workspace "$LAW_ID" \
  --log-analytics-workspace-key "$LAW_KEY"
```
</details>

---

## Step 14 - Recupera IP pubblico e verifica l’applicazione

Esegui:

```bash
ACI_PUBLIC_IP=$(az container show \
  --resource-group "$RG" \
  --name "$ACI_NAME" \
  --query ipAddress.ip -o tsv)

echo "$ACI_PUBLIC_IP"
```

Testa gli endpoint:

```bash
curl "http://$ACI_PUBLIC_IP:8000/"
curl "http://$ACI_PUBLIC_IP:8000/health"
curl "http://$ACI_PUBLIC_IP:8000/time"
curl -X POST "http://$ACI_PUBLIC_IP:8000/echo" -H 'Content-Type: application/json' -d '{"course":"observability"}'
curl "http://$ACI_PUBLIC_IP:8000/error"
curl "http://$ACI_PUBLIC_IP:8000/nope"
```

### Verifica log runtime ACI

```bash
az container logs --resource-group "$RG" --name "$ACI_NAME"
```

Dovresti vedere righe JSON.

---

## Step 15 - Attendi l’ingestione dei log

Attendi alcuni minuti.

I log non compaiono sempre immediatamente in Log Analytics.

Come riferimento operativo per il laboratorio, attendi circa **5-10 minuti** prima di concludere che qualcosa non va.

---

## Step 16 - Verifica il formato reale dei log in Log Analytics

Apri il portale Azure:

- cerca **Log Analytics workspaces**
- apri `law-observability-lab12`
- apri **Logs**

Esegui questa query:

```kql
ContainerInstanceLog_CL
| where ContainerGroup == "obsapp-aci"
| project TimeGenerated, ContainerGroup, Message
| order by TimeGenerated desc
| take 20
```

### Che cosa devi osservare

- deve esistere la tabella `ContainerInstanceLog_CL`
- il campo `Message` deve contenere JSON applicativo
- nelle righe devono comparire campi come `status`, `path`, `request_id`

### Checkpoint critico

Non creare l’alert finché non hai verificato il dataset reale.

---

## Step 17 - Calcola `error_rate` con KQL

Esegui questa query:

```kql
ContainerInstanceLog_CL
| where ContainerGroup == "obsapp-aci"
| extend payload = parse_json(Message)
| extend status = toint(payload.status)
| where isnotnull(status)
| summarize 
    total_requests = count(),
    error_requests = countif(status >= 400),
    error_rate = todouble(countif(status >= 400)) / todouble(count())
```

### Che cosa fa la query

- prende solo i log del container group del laboratorio
- interpreta `Message` come JSON
- estrae il campo `status`
- conta le richieste totali
- conta le richieste con `status >= 400`
- calcola il rapporto `error_rate`

---

<details>
### RETTIFICA QUERY KQL:
Nelle query che non funzionano, si usa ContainerGroup == "obsapp-aci", mentre la colonna corretta da filtrare è ContainerName_s.
Quindi, per far funzionare le query, basta sostituire ContainerGroup con ContainerName_s in tutti i punti in cui viene usata.

---

### Query 1 KQL:
```kql
ContainerInstanceLog_CL
| where ContainerName_s== "obsapp-aci"
| project TimeGenerated, ContainerName_s, Message
| order by TimeGenerated desc
| take 20
```

### Query 2 KQL:
```kql
ContainerInstanceLog_CL
| where ContainerName_s == "obsapp-aci"
| extend payload = parse_json(Message)
| extend status = toint(payload.status)
| where isnotnull(status)
| summarize 
    total_requests = count(),
    error_requests = countif(status >= 400),
    error_rate = todouble(countif(status >= 400)) / todouble(count())
```
</details>

---

## Step 18 - Genera traffico normale ed errori

In WSL Ubuntu esegui prima richieste sane:

```bash
for i in {1..10}; do
  curl -s "http://$ACI_PUBLIC_IP:8000/health" > /dev/null
  curl -s "http://$ACI_PUBLIC_IP:8000/time" > /dev/null
done
```

Poi genera errori:

```bash
for i in {1..40}; do
  curl -s "http://$ACI_PUBLIC_IP:8000/nope" > /dev/null
done
```

Ripeti quindi la query KQL dello step precedente.

### Risultato atteso

L’`error_rate` deve aumentare sensibilmente.

---

## Step 19 - Crea l’Action Group

Nel portale Azure vai in:

- **Monitor**
- **Alerts**
- **Action groups**
- **Create**

Impostazioni consigliate:

### Basics

- Subscription: la tua subscription
- Resource group: `rg-observability-lab12-standalone`
- Region: **Global**
- Action group name: `ag-observability`
- Display name: `ag-obs`

### Notifications

Aggiungi una notifica di tipo:

- **Email/SMS/Push/Voice**
- Email
- inserisci il tuo indirizzo email

Crea quindi il gruppo.

### Checkpoint

L’Action Group deve risultare creato.

---

## Step 20 - Crea la Alert Rule

Nel portale vai in:

- **Monitor**
- **Alerts**
- **Create**
- **Alert rule**

### Scope

Seleziona il workspace:

- `law-observability-lab12`

### Condition

Seleziona:

- **Custom log search**

Usa questa query:

```kql
ContainerInstanceLog_CL
| where ContainerGroup == "obsapp-aci"
| extend payload = parse_json(Message)
| extend status = toint(payload.status)
| where isnotnull(status)
| summarize error_rate = todouble(countif(status >= 400)) / todouble(count())
```

### Parametri consigliati

- Measure: basata sul valore restituito dalla query
- Operator: Greater than
- Threshold value: `0.20`
- Evaluation frequency: `5 minutes`
- Window size: `5 minutes`
- Severity: `2`

### Actions

Collega:

- `ag-observability`

### Details

Nome suggerito:

- `alert-error-rate-obsapp`

Abilita la regola e crea.

### Checkpoint

La regola deve risultare creata e abilitata.

---

## Step 21 - Attendi la valutazione dell’alert

Dopo la generazione degli errori attendi:

- ingestione dei log
- valutazione della finestra temporale
- esecuzione della regola

Nel laboratorio considera normale attendere **5-10 minuti**.

---

## Step 22 - Verifica Fired alerts

Nel portale vai in:

- **Monitor**
- **Alerts**
- **Fired alerts**

oppure:

- **Alert rules**
- apri la regola
- controlla **alert history**

### Checkpoint finale

Devi verificare che l’alert sia andato in stato **Fired** oppure che la cronologia mostri l’attivazione.

---

## Step 23 - Troubleshooting se l’alert non scatta

Controlla in ordine:

1. il workspace corretto è `law-observability-lab12`?
2. la ACI è in stato Running?
3. l’immagine corretta è stata distribuita?
4. il campo `Message` contiene davvero JSON?
5. la query KQL restituisce dati se lanciata manualmente?
6. `ContainerGroup == "obsapp-aci"` corrisponde davvero al nome usato?
7. hai generato abbastanza errori?
8. hai atteso abbastanza per ingestione e valutazione?
9. l’Action Group è collegato?
10. la regola è abilitata?

---

## Step 24 - Crea il file delle evidenze

Crea:

```bash
cd ~/course/lab12-standalone
nano docs/evidence_lab12_standalone.md
```

Incolla questa struttura:

````md
# LAB12 Standalone - Evidence

## 1. Resource Group
- Nome:
- Location:

## 2. Log Analytics Workspace
- Nome:
- Workspace ID:

## 3. Azure Container Registry
- Nome:
- Login server:
- Repository:
- Tag:

## 4. Azure Container Instance
- Nome:
- Public IP:
- Porta esposta:

## 5. Verifica formato log
- Tabella verificata: ContainerInstanceLog_CL
- Campo verificato: Message
- Esempio di riga osservata:
- Il contenuto è JSON valido? SÌ/NO

## 6. Query KQL usata per error_rate
```kql
ContainerInstanceLog_CL
| where ContainerGroup == "obsapp-aci"
| extend payload = parse_json(Message)
| extend status = toint(payload.status)
| where isnotnull(status)
| summarize error_rate = todouble(countif(status >= 400)) / todouble(count())
```

## 7. Simulazione traffico
- Richieste sane generate: 
- Richieste in errore generate:

## 8. Action Group
- Nome:
- Tipo di azione:
- Email usata:

## 9. Alert Rule
- Nome:
- Threshold:
- Evaluation frequency:
- Window size:
- Severity:

## 10. Verifica alert
- Fired: SÌ/NO
- Orario osservato:
- Screenshot allegato: SÌ/NO

## 11. Motivazione soglia
- Perché è stata scelta la soglia 0.20:
- Perché un alert rumoroso è un problema:

## 12. Note finali
- Che cosa ho capito su Log Analytics:
- Che cosa ho capito su Alert Rule:
- Che cosa ho capito su Action Group:
- Che cosa significa Fired:
````

---

## Step 25 - Consegna nel repository Git

Se stai lavorando dentro un repository Git, esegui:

```bash
git add src/app.py src/requirements.txt src/Dockerfile docs/evidence_lab12_standalone.md
git commit -m "[LAB12 standalone] completato"
git push
```

Se non stai lavorando in un repository, conserva comunque:

- `cmdlog_lab12_standalone.txt`
- `docs/evidence_lab12_standalone.md`
- screenshot del portale

---

# PARTE 3 - Checkpoint, criteri di completamento e cleanup

## Checkpoint riassuntivi

### Checkpoint #1

Resource Group e Log Analytics Workspace creati.

### Checkpoint #2

ACR creato e immagine pubblicata.

### Checkpoint #3

ACI creata e applicazione raggiungibile via IP pubblico.

### Checkpoint #4

Log visibili in `ContainerInstanceLog_CL` con `Message` in formato JSON.

### Checkpoint #5

Query KQL per `error_rate` funzionante.

### Checkpoint #6

Action Group creato.

### Checkpoint #7

Alert Rule creata e abilitata.

### Checkpoint #8

Alert verificato in stato **Fired** oppure in **alert history**.

---

## Criteri di completamento

Il laboratorio è completato se:

- l’applicazione è stata ricreata da zero
- l’immagine è stata pubblicata in ACR
- la ACI è in esecuzione
- i log arrivano in Log Analytics
- la query KQL calcola `error_rate`
- l’Action Group esiste
- la Alert Rule esiste ed è attiva
- l’alert è stato verificato
- l’evidence è completa

---

## Cleanup facoltativo a fine laboratorio

Se vuoi eliminare tutte le risorse create:

```bash
az group delete --name "$RG" --yes --no-wait
```

**Nel percorso continuativo non eseguire questo comando se intendi riusare le risorse nei laboratori successivi.**
