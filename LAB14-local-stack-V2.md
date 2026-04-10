Questo **LAB14-00-local-stack** è coerente con l’architettura locale che hai già impostato: **app stack separato** da **observability stack**, con `frontend-service` e `backend-service` osservati da **Prometheus**, **Grafana** e **Jaeger** tramite rete Docker condivisa `obs-net`. Serve anche come **ponte didattico** verso il seguito su Azure, dove il mapping concettuale diventa Prometheus → Azure Monitor metrics, Jaeger → Application Insights, Grafana → Azure Managed Grafana.  



---

# LAB14-00-local-stack -- Ponte tra Observability e DevOps

## Applicazione distribuita locale con stack observability separato

## 1. Obiettivo del laboratorio

In questo laboratorio realizzerai e verificherai in locale una piccola applicazione distribuita composta da:

* un **frontend service**
* un **backend service**

L’applicazione sarà osservata da uno stack di observability separato composto da:

* **Prometheus**
* **Grafana**
* **Jaeger**

Alla fine del laboratorio dovrai essere in grado di:

* distinguere tra **sistema osservato** e **sistema osservante**
* avviare due stack Docker separati ma collegati tra loro
* verificare una chiamata distribuita **frontend -> backend**
* leggere **metriche aggregate** in Prometheus
* costruire una **dashboard base** in Grafana
* visualizzare una **trace distribuita** in Jaeger
* spiegare perché questo laboratorio è il ponte corretto verso il successivo uso di Azure Monitor, Application Insights e Log Analytics

---

## 2. Scenario del laboratorio

L’architettura che realizzerai è la seguente:

```text
+-----------------------------+
|         APP STACK           |
|                             |
|  frontend-service :8000     |
|         |                   |
|         v                   |
|  backend-service  :8001     |
+-----------------------------+

              |
              | rete Docker condivisa: obs-net
              v

+-----------------------------+
|    OBSERVABILITY STACK      |
|                             |
|  Prometheus      :9090      |
|  Grafana         :3000      |
|  Jaeger UI       :16686     |
+-----------------------------+
```

### Flussi principali

**Metriche**

* frontend espone `/metrics`
* backend espone `/metrics`
* Prometheus esegue lo scrape

**Trace**

* frontend riceve la richiesta
* frontend chiama backend
* il contesto viene propagato
* backend continua la stessa trace
* Jaeger visualizza la trace distribuita completa

**Dashboard**

* Grafana usa Prometheus come datasource
* Grafana mostra pannelli con richieste e latenza

---

## 3. Perché facciamo questo laboratorio prima di Azure DevOps

Prima di costruire pipeline, fare push su ACR e distribuire nel cloud, devi capire bene:

* che cos’è una **applicazione distribuita**
* che cosa significa **strumentarla**
* che cosa osservano davvero i diversi strumenti
* perché le metriche aggregate e la trace della singola richiesta **non sono la stessa cosa**

Questo laboratorio ti serve per vedere in modo pulito che:

* l’applicazione **produce telemetria**
* lo stack di observability **raccoglie telemetria**
* Prometheus lavora sulle **metriche aggregate**
* Jaeger ricostruisce la **singola trace**
* Grafana presenta una **vista operativa sintetica**

Nel modulo successivo, quando passerai ad Azure, la logica resterà la stessa ma con strumenti gestiti dalla piattaforma. Il ponte concettuale è proprio questo.  

---

## 4. Durata stimata

**4-6 ore**

---

## 5. Prerequisiti

È necessario disporre di:

* Windows 10/11 con **WSL2**
* Ubuntu in WSL
* **Docker Desktop** configurato con motore Linux
* Docker funzionante in WSL
* VS Code o altro editor
* conoscenze minime di:

  * terminale Linux
  * file e cartelle
  * Docker di base
  * concetti introduttivi di observability

Verifica rapida dei prerequisiti:

```bash
docker --version
docker compose version
pwd
ls
```

---

## 6. Struttura del progetto

Crea una cartella di lavoro con questa struttura:

```text
demo-separated-observability/
│
├─ app-stack/
│  ├─ docker-compose.app.yml
│  ├─ frontend/
│  │  ├─ Dockerfile
│  │  ├─ requirements.txt
│  │  └─ app.py
│  └─ backend/
│     ├─ Dockerfile
│     ├─ requirements.txt
│     └─ app.py
│
└─ obs-stack/
   ├─ docker-compose.obs.yml
   ├─ prometheus/
   │  └─ prometheus.yml
   └─ grafana/
      └─ provisioning/
         └─ datasources/
            └─ datasource.yml
```

---

## 7. Parte 1 - Preparazione cartelle

Apri il terminale WSL e crea le cartelle.

```bash
mkdir -p ~/corso_obs/demo-separated-observability/app-stack/frontend
mkdir -p ~/corso_obs/demo-separated-observability/app-stack/backend
mkdir -p ~/corso_obs/demo-separated-observability/obs-stack/prometheus
mkdir -p ~/corso_obs/demo-separated-observability/obs-stack/grafana/provisioning/datasources
```

Entra nella cartella di progetto:

```bash
cd ~/corso_obs/demo-separated-observability
```

Verifica:

```bash
find . -maxdepth 4 -type d | sort
```

---

## 8. Parte 2 - Creazione dei file applicativi

### 8.1 File `app-stack/docker-compose.app.yml`

Crea il file:

```bash
code app-stack/docker-compose.app.yml
```

Inserisci il seguente contenuto:

```yaml
services:
  frontend:
    build: ./frontend
    container_name: frontend-service
    ports:
      - "8000:8000"
    environment:
      - SERVICE_NAME=frontend-service
      - BACKEND_URL=http://backend-service:8001/work
      - JAEGER_AGENT_HOST=jaeger
      - JAEGER_AGENT_PORT=6831
    networks:
      - obs-net

  backend:
    build: ./backend
    container_name: backend-service
    ports:
      - "8001:8001"
    environment:
      - SERVICE_NAME=backend-service
      - JAEGER_AGENT_HOST=jaeger
      - JAEGER_AGENT_PORT=6831
    networks:
      - obs-net

networks:
  obs-net:
    external: true
```

---

### 8.2 File `app-stack/frontend/requirements.txt`

Crea il file:

```bash
code app-stack/frontend/requirements.txt
```

Inserisci:

```txt
flask==3.0.0
requests==2.31.0
prometheus_client==0.20.0
opentelemetry-api==1.24.0
opentelemetry-sdk==1.24.0
opentelemetry-exporter-jaeger-thrift==1.21.0
opentelemetry-instrumentation-flask==0.45b0
opentelemetry-instrumentation-requests==0.45b0
```

### 8.2 bis - Che cos'è il file `requirements.txt`

Nel mondo Python, il file `requirements.txt` contiene l'elenco delle librerie esterne che l'applicazione deve installare per poter funzionare.

Quando nel `Dockerfile` eseguiamo il comando:

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

stiamo dicendo a Python di installare tutte le dipendenze elencate nel file.

Questo approccio ha tre vantaggi pratici:

1. rende l'ambiente riproducibile
2. evita installazioni manuali una per una
3. permette a tutti i partecipanti di usare le stesse versioni delle librerie

In questo laboratorio usiamo versioni esplicite, ad esempio `flask==3.0.0`, per evitare differenze di comportamento tra un ambiente e l'altro.

### 8.2 ter - Spiegazione del file `app-stack/frontend/requirements.txt`

Il file del frontend contiene librerie che servono a quattro scopi principali:

* creare il servizio web
* eseguire chiamate HTTP verso il backend
* esporre metriche per Prometheus
* generare ed esportare trace verso Jaeger

Di seguito trovi il significato di ogni dipendenza.

#### `flask==3.0.0`

**Flask** è il micro-framework web che usiamo per creare il servizio HTTP.

Nel file `frontend/app.py` Flask serve a:

* creare l'applicazione
* definire gli endpoint come `/health`, `/demo`, `/demo-load` e `/metrics`
* restituire risposte JSON al client

Senza Flask il frontend non potrebbe comportarsi da servizio web.

#### `requests==2.31.0`

La libreria **requests** serve per eseguire chiamate HTTP in uscita verso altri servizi.

Nel frontend viene usata perché questo servizio, quando riceve una richiesta su `/demo`, deve chiamare il backend all'URL definito dalla variabile `BACKEND_URL`.

In pratica:

* il browser o `curl` chiamano il frontend
* il frontend usa `requests` per chiamare il backend
* il backend restituisce la sua risposta
* il frontend la incorpora nella risposta finale

Questa libreria è presente solo nel frontend perché è il frontend che effettua la chiamata HTTP verso un altro servizio.

#### `prometheus_client==0.20.0`

Questa libreria serve a esporre **metriche Prometheus**.

Nel laboratorio viene usata per creare:

* un **Counter** per contare le richieste
* un **Histogram** per misurare la latenza
* l'endpoint `/metrics` che Prometheus leggerà periodicamente

Senza questa libreria, Prometheus non avrebbe nulla da raccogliere dal frontend.

#### `opentelemetry-api==1.24.0`

Questa è la parte **API** di OpenTelemetry.

Serve a fornire le interfacce di base per lavorare con il tracing, ad esempio:

* ottenere un tracer
* creare span
* propagare il contesto della trace

Si può pensare all'API come al contratto generale del sistema di tracing.

#### `opentelemetry-sdk==1.24.0`

La **SDK** è l'implementazione concreta dell'API.

Se l'API definisce i concetti, la SDK li rende operativi. In questo laboratorio la SDK serve per:

* creare il `TracerProvider`
* definire la `Resource` con il nome del servizio
* aggiungere lo span processor
* gestire la produzione reale delle trace

#### `opentelemetry-exporter-jaeger-thrift==1.21.0`

Questa libreria serve a **spedire le trace a Jaeger**.

Generare gli span nel codice non basta: bisogna anche esportarli verso un sistema esterno che li raccolga e li visualizzi.

Nel laboratorio quel sistema è Jaeger.

#### `opentelemetry-instrumentation-flask==0.45b0`

Questa libreria abilita l'**instrumentation automatica di Flask**.

Significa che OpenTelemetry intercetta automaticamente le richieste in ingresso all'app Flask e crea span relativi alle chiamate HTTP senza che tu debba costruirli tutti a mano.

In termini didattici:

* il codice manuale crea gli span di business
* l'instrumentation automatica crea gli span tecnici legati al framework web

#### `opentelemetry-instrumentation-requests==0.45b0`

Questa libreria abilita l'**instrumentation automatica della libreria requests**.

Nel frontend è molto importante, perché il frontend non si limita a ricevere richieste: ne genera anche una verso il backend.

Questo aiuta a vedere in Jaeger una trace distribuita che attraversa:

* il frontend
* la chiamata HTTP tra frontend e backend
* il backend

### 8.2 quater - Tabella riassuntiva delle dipendenze del frontend

| Libreria | A cosa serve nel laboratorio |
|---|---|
| Flask | Crea il servizio web e gli endpoint HTTP |
| requests | Esegue la chiamata HTTP dal frontend al backend |
| prometheus_client | Espone metriche leggibili da Prometheus |
| opentelemetry-api | Definisce le API di tracing |
| opentelemetry-sdk | Implementa concretamente il tracing |
| opentelemetry-exporter-jaeger-thrift | Invia le trace a Jaeger |
| opentelemetry-instrumentation-flask | Traccia automaticamente le richieste Flask |
| opentelemetry-instrumentation-requests | Traccia automaticamente le chiamate HTTP in uscita |


---

### 8.3 File `app-stack/backend/requirements.txt`

Crea il file:

```bash
code app-stack/backend/requirements.txt
```

Inserisci:

```txt
flask==3.0.0
prometheus_client==0.20.0
opentelemetry-api==1.24.0
opentelemetry-sdk==1.24.0
opentelemetry-exporter-jaeger-thrift==1.21.0
opentelemetry-instrumentation-flask==0.45b0
```

### 8.3 bis - Spiegazione del file `app-stack/backend/requirements.txt`

Il backend usa un file più corto rispetto al frontend:

* usa Flask per esporre gli endpoint
* usa `prometheus_client` per pubblicare metriche
* usa OpenTelemetry per produrre trace
* usa l'exporter Jaeger per inviarle a Jaeger
* usa l'instrumentation di Flask per tracciare le richieste in ingresso

#### Perché nel backend non c'è `requests`

Nel laboratorio il backend **riceve** richieste dal frontend, ma non effettua a sua volta chiamate HTTP verso un terzo servizio.

Per questo nel suo `requirements.txt` non compare la libreria `requests`.

#### Perché nel backend non c'è `opentelemetry-instrumentation-requests`

Questa dipendenza serve a tracciare le chiamate HTTP in uscita realizzate con la libreria `requests`.

Poiché nel backend fornito dal laboratorio non esistono chiamate HTTP in uscita verso altri servizi, questa dipendenza non è necessaria.

### 8.3 ter - Confronto concettuale tra frontend e backend

Confrontando i due file `requirements.txt` si capisce anche il ruolo dei due servizi.

Il **frontend**:

* riceve richieste HTTP
* chiama il backend
* restituisce una risposta composta al client
* esporta metriche e trace

Il **backend**:

* riceve richieste dal frontend
* esegue la logica di lavoro
* restituisce il risultato
* esporta metriche e trace

Per questo i due file sono simili, ma non identici.

### 8.3 quater - Cosa devi ricordare quando leggi un `requirements.txt`

Quando leggi un file `requirements.txt`, non devi pensarlo come una lista casuale di nomi.

Devi chiederti:

* quale libreria costruisce il servizio web
* quale libreria genera metriche
* quale libreria abilita il tracing
* quale libreria esporta le trace
* quale libreria serve per le chiamate tra microservizi

In questo laboratorio la risposta è:

* **Flask** costruisce il servizio web
* **prometheus_client** espone metriche
* **OpenTelemetry API + SDK** gestiscono il tracing
* **Jaeger exporter** invia le trace a Jaeger
* **requests** permette al frontend di chiamare il backend
* le librerie di **instrumentation** automatizzano parte della raccolta di telemetria


---

### 8.4 File `app-stack/frontend/Dockerfile`

Crea il file:

```bash
code app-stack/frontend/Dockerfile
```

Inserisci:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8000

CMD ["python", "app.py"]
```

---

### 8.5 File `app-stack/backend/Dockerfile`

Crea il file:

```bash
code app-stack/backend/Dockerfile
```

Inserisci:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8001

CMD ["python", "app.py"]
```

---

### 8.6 File `app-stack/frontend/app.py`

Crea il file:

```bash
code app-stack/frontend/app.py
```

Inserisci:

```python
import os
import time
import random
import requests
from flask import Flask, jsonify, Response
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

from opentelemetry import trace
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

SERVICE_NAME = os.getenv("SERVICE_NAME", "frontend-service")
BACKEND_URL = os.getenv("BACKEND_URL", "http://backend-service:8001/work")
JAEGER_AGENT_HOST = os.getenv("JAEGER_AGENT_HOST", "jaeger")
JAEGER_AGENT_PORT = int(os.getenv("JAEGER_AGENT_PORT", "6831"))

app = Flask(__name__)

REQUEST_COUNT = Counter(
    "demo_frontend_requests_total",
    "Total frontend requests",
    ["endpoint", "method", "status"]
)

REQUEST_LATENCY = Histogram(
    "demo_frontend_request_duration_seconds",
    "Frontend request latency",
    ["endpoint"]
)

resource = Resource.create({"service.name": SERVICE_NAME})
provider = TracerProvider(resource=resource)
trace.set_tracer_provider(provider)

jaeger_exporter = JaegerExporter(
    agent_host_name=JAEGER_AGENT_HOST,
    agent_port=JAEGER_AGENT_PORT
)

provider.add_span_processor(BatchSpanProcessor(jaeger_exporter))
tracer = trace.get_tracer(__name__)

FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

@app.route("/health")
def health():
    return jsonify({"status": "ok", "service": SERVICE_NAME})

@app.route("/demo")
def demo():
    start = time.time()
    status = "200"

    with tracer.start_as_current_span("frontend-business-span") as span:
        span.set_attribute("service.layer", "frontend")

        delay = random.uniform(0.05, 0.30)
        time.sleep(delay)
        span.set_attribute("frontend.synthetic_delay_seconds", delay)

        try:
            response = requests.get(BACKEND_URL, timeout=5)
            backend_data = response.json()
        except Exception as exc:
            status = "500"
            REQUEST_COUNT.labels(endpoint="/demo", method="GET", status=status).inc()
            REQUEST_LATENCY.labels(endpoint="/demo").observe(time.time() - start)
            return jsonify({
                "service": SERVICE_NAME,
                "status": "error",
                "message": str(exc)
            }), 500

    REQUEST_COUNT.labels(endpoint="/demo", method="GET", status=status).inc()
    REQUEST_LATENCY.labels(endpoint="/demo").observe(time.time() - start)

    return jsonify({
        "service": SERVICE_NAME,
        "message": "frontend completed",
        "backend_response": backend_data
    })

@app.route("/demo-load")
def demo_load():
    results = []
    for _ in range(5):
        try:
            r = requests.get("http://localhost:8000/demo", timeout=10)
            results.append(r.status_code)
        except Exception:
            results.append(500)
    return jsonify({"generated_requests": len(results), "statuses": results})

@app.route("/metrics")
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

---

### 8.7 File `app-stack/backend/app.py`

Crea il file:

```bash
code app-stack/backend/app.py
```

Inserisci:

```python
import os
import time
import random
from flask import Flask, jsonify, Response
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

from opentelemetry import trace
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor

SERVICE_NAME = os.getenv("SERVICE_NAME", "backend-service")
JAEGER_AGENT_HOST = os.getenv("JAEGER_AGENT_HOST", "jaeger")
JAEGER_AGENT_PORT = int(os.getenv("JAEGER_AGENT_PORT", "6831"))

app = Flask(__name__)

REQUEST_COUNT = Counter(
    "demo_backend_requests_total",
    "Total backend requests",
    ["endpoint", "method", "status"]
)

REQUEST_LATENCY = Histogram(
    "demo_backend_request_duration_seconds",
    "Backend request latency",
    ["endpoint"]
)

resource = Resource.create({"service.name": SERVICE_NAME})
provider = TracerProvider(resource=resource)
trace.set_tracer_provider(provider)

jaeger_exporter = JaegerExporter(
    agent_host_name=JAEGER_AGENT_HOST,
    agent_port=JAEGER_AGENT_PORT
)

provider.add_span_processor(BatchSpanProcessor(jaeger_exporter))
tracer = trace.get_tracer(__name__)

FlaskInstrumentor().instrument_app(app)

@app.route("/health")
def health():
    return jsonify({"status": "ok", "service": SERVICE_NAME})

@app.route("/work")
def work():
    start = time.time()
    status = "200"

    with tracer.start_as_current_span("backend-business-span") as span:
        span.set_attribute("service.layer", "backend")

        delay = random.uniform(0.10, 0.80)
        time.sleep(delay)
        span.set_attribute("backend.synthetic_delay_seconds", delay)

        if delay > 0.65:
            span.set_attribute("backend.slow_call", True)

    REQUEST_COUNT.labels(endpoint="/work", method="GET", status=status).inc()
    REQUEST_LATENCY.labels(endpoint="/work").observe(time.time() - start)

    return jsonify({
        "service": SERVICE_NAME,
        "message": "backend completed",
        "processing_time_seconds": round(delay, 3)
    })

@app.route("/work-error")
def work_error():
    start = time.time()
    status = "500"

    with tracer.start_as_current_span("backend-error-span") as span:
        span.set_attribute("service.layer", "backend")
        span.set_attribute("error.simulated", True)

    REQUEST_COUNT.labels(endpoint="/work-error", method="GET", status=status).inc()
    REQUEST_LATENCY.labels(endpoint="/work-error").observe(time.time() - start)

    return jsonify({
        "service": SERVICE_NAME,
        "message": "simulated backend failure"
    }), 500

@app.route("/metrics")
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8001)
```

---

## 9. Parte 3 - Creazione dei file dello stack observability

### 9.1 File `obs-stack/docker-compose.obs.yml`

Crea il file:

```bash
code obs-stack/docker-compose.obs.yml
```

Inserisci:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    networks:
      - obs-net

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - ./grafana/provisioning/datasources:/etc/grafana/provisioning/datasources:ro
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus
    networks:
      - obs-net

  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger
    ports:
      - "16686:16686"
      - "6831:6831/udp"
    networks:
      - obs-net

networks:
  obs-net:
    external: true
```

---

### 9.2 File `obs-stack/prometheus/prometheus.yml`

Crea il file:

```bash
code obs-stack/prometheus/prometheus.yml
```

Inserisci:

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: "frontend-service"
    metrics_path: /metrics
    static_configs:
      - targets: ["frontend-service:8000"]

  - job_name: "backend-service"
    metrics_path: /metrics
    static_configs:
      - targets: ["backend-service:8001"]
```

---

### 9.3 File `obs-stack/grafana/provisioning/datasources/datasource.yml`

Crea il file:

```bash
code obs-stack/grafana/provisioning/datasources/datasource.yml
```

Inserisci:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
```

---

## 10. Parte 4 - Creazione della rete Docker condivisa

Crea la rete una sola volta:

```bash
docker network create obs-net
```

Verifica che esista:

```bash
docker network ls
```

Dovresti vedere `obs-net`.

---

## 11. Parte 5 - Avvio dei container

### 11.1 Avvio dello stack observability

Entra nella cartella:

```bash
cd ~/corso_obs/demo-separated-observability/obs-stack
```

Avvia i container:

```bash
docker compose -f docker-compose.obs.yml up -d
```

Verifica:

```bash
docker ps
```

Dovresti vedere almeno:

* `prometheus`
* `grafana`
* `jaeger`

---

### 11.2 Avvio dello stack applicativo

Entra nella cartella:

```bash
cd ~/corso_obs/demo-separated-observability/app-stack
```

Costruisci e avvia:

```bash
docker compose -f docker-compose.app.yml up --build -d
```

Verifica:

```bash
docker ps
```

Dovresti vedere anche:

* `frontend-service`
* `backend-service`

---

## 12. Parte 6 - Test di funzionamento applicativo

### 12.1 Health check frontend

```bash
curl http://localhost:8000/health
```

Risultato atteso:

```json
{"status":"ok","service":"frontend-service"}
```

---

### 12.2 Health check backend

```bash
curl http://localhost:8001/health
```

Risultato atteso:

```json
{"status":"ok","service":"backend-service"}
```

---

### 12.3 Chiamata distribuita frontend -> backend

```bash
curl http://localhost:8000/demo
```

Risultato atteso simile a questo:

```json
{
  "service": "frontend-service",
  "message": "frontend completed",
  "backend_response": {
    "service": "backend-service",
    "message": "backend completed",
    "processing_time_seconds": 0.421
  }
}
```

---

### 12.4 Generazione traffico ripetuto

Esegui più chiamate:

```bash
for i in {1..10}; do curl -s http://localhost:8000/demo; echo; done
```

Poi genera ancora più traffico:

```bash
for i in {1..20}; do curl -s http://localhost:8000/demo > /dev/null; done
```

---

## 13. Parte 7 - Verifica delle metriche in Prometheus

Apri il browser su:

```text
http://localhost:9090
```

### Query 1 - richieste frontend totali

```promql
demo_frontend_requests_total
```

### Query 2 - richieste backend totali

```promql
demo_backend_requests_total
```

### Query 3 - rate frontend

```promql
rate(demo_frontend_requests_total[1m])
```

### Query 4 - rate backend

```promql
rate(demo_backend_requests_total[1m])
```

### Query 5 - latenza media frontend

```promql
rate(demo_frontend_request_duration_seconds_sum[1m])
/
rate(demo_frontend_request_duration_seconds_count[1m])
```

### Query 6 - latenza media backend

```promql
rate(demo_backend_request_duration_seconds_sum[1m])
/
rate(demo_backend_request_duration_seconds_count[1m])
```

### Cosa devi osservare

* il numero di richieste cresce dopo i test con `curl`
* frontend e backend hanno entrambi metriche proprie
* la latenza del backend tende a essere più variabile per via del ritardo sintetico inserito nel codice
* Prometheus ti mostra una **vista aggregata**, non il dettaglio della singola richiesta

---

## 14. Parte 8 - Verifica della dashboard in Grafana

Apri il browser su:

```text
http://localhost:3000
```

Credenziali:

* username: `admin`
* password: `admin`

### Passi da eseguire

1. entra in Grafana
2. apri **Connections / Data Sources**
3. verifica che il datasource **Prometheus** sia presente
4. crea una dashboard nuova
5. aggiungi i seguenti pannelli

### Pannello 1 - total frontend requests

```promql
demo_frontend_requests_total
```

### Pannello 2 - total backend requests

```promql
demo_backend_requests_total
```

### Pannello 3 - average frontend latency

```promql
rate(demo_frontend_request_duration_seconds_sum[1m])
/
rate(demo_frontend_request_duration_seconds_count[1m])
```

### Pannello 4 - average backend latency

```promql
rate(demo_backend_request_duration_seconds_sum[1m])
/
rate(demo_backend_request_duration_seconds_count[1m])
```

### Cosa devi osservare

* Grafana non raccoglie i dati da sola
* Grafana **visualizza** i dati interrogando Prometheus
* i pannelli mostrano una vista rapida utile per operazioni e monitoraggio

---

## 15. Parte 9 - Verifica delle trace in Jaeger

Apri il browser su:

```text
http://localhost:16686
```

### Passi da eseguire

1. seleziona il servizio `frontend-service`
2. clicca **Find Traces**
3. apri una trace disponibile
4. analizza la catena completa della richiesta

### Cosa devi osservare

Nella stessa trace troverai tipicamente:

* span HTTP del frontend
* span business del frontend
* chiamata HTTP verso backend
* span HTTP del backend
* span business del backend

### Concetto fondamentale

La correlazione della singola richiesta **non la fa Prometheus**.

La trace distribuita viene ricostruita da Jaeger grazie a:

* generazione della trace
* propagazione del contesto
* continuità tra gli span

Questo è il punto chiave del laboratorio e uno dei pochi momenti in cui bisogna davvero capire, non solo eseguire. 

---

## 16. Parte 10 - Analisi guidata

Rispondi alle seguenti domande in modo sintetico ma preciso.

### 16.1 Sistema osservato e sistema osservante

* Quali container appartengono al **sistema osservato**?
* Quali container appartengono al **sistema osservante**?

---

### 16.2 Metriche e trace

* Che differenza c’è tra una metrica aggregata e una trace distribuita?
* Perché Prometheus non basta per ricostruire una singola richiesta end-to-end?
* Perché Jaeger non sostituisce Prometheus?

---

### 16.3 Ruolo di Grafana

* Qual è il ruolo reale di Grafana in questo stack?
* Perché Grafana non è il datastore delle metriche?

---

### 16.4 Preparazione al cloud

* Se in locale usiamo Prometheus, Jaeger e Grafana, quali servizi Azure svolgeranno un ruolo concettualmente analogo nei laboratori successivi?
* Perché ha senso prima vedere tutto in locale e poi spostarsi in Azure?

---

## 17. Parte 11 - Verifica finale pratica

Esegui questa sequenza:

```bash
docker ps
curl http://localhost:8000/health
curl http://localhost:8001/health
for i in {1..20}; do curl -s http://localhost:8000/demo > /dev/null; done
```

Poi verifica:

* in Prometheus: almeno una metrica di frontend e una di backend
* in Grafana: dashboard con almeno 4 pannelli
* in Jaeger: almeno una trace distribuita completa

---

## 18. Parte 12 - Troubleshooting minimo

### Problema 1 - `docker: command not found`

Possibile causa:

* Docker Desktop non installato
* integrazione WSL non attiva

Verifica:

```bash
docker --version
```

---

### Problema 2 - Grafana non raggiungibile su `localhost:3000`

Possibile causa:

* container non avviato
* porta occupata

Verifica:

```bash
docker ps
docker logs grafana
```

---

### Problema 3 - Prometheus non mostra target attivi

Possibile causa:

* rete Docker non corretta
* nomi container non risolti
* app stack non avviato

Verifica:

```bash
docker network ls
docker ps
docker logs prometheus
```

---

### Problema 4 - Jaeger non mostra trace

Possibile causa:

* nessuna richiesta `/demo` eseguita
* exporter non raggiunge Jaeger
* app stack e obs stack non condividono la stessa rete

Verifica:

```bash
docker ps
docker logs frontend-service
docker logs backend-service
docker logs jaeger
```

---

### Problema 5 - errore nella build delle immagini Python

Possibile causa:

* file `requirements.txt` o `Dockerfile` scritti male
* indentazione o nome file errati

Verifica:

```bash
find app-stack -maxdepth 3 -type f | sort
```

---

## 19. Parte 13 - Evidenze richieste

Crea il file:

```bash
mkdir -p docs
code docs/evidence_lab00.md
```

Inserisci nel file le seguenti sezioni.

### Struttura richiesta per `docs/evidence_lab00.md`

```md
# Evidence LAB00

## 1. Architettura compresa
Descrivo con parole mie la differenza tra sistema osservato e sistema osservante.

## 2. Verifica container
Incollo l'output di:
- docker ps

## 3. Test applicativi
Incollo l'output di:
- curl http://localhost:8000/health
- curl http://localhost:8001/health
- curl http://localhost:8000/demo

## 4. Metriche Prometheus
Descrivo quali query ho eseguito e cosa ho osservato.

## 5. Dashboard Grafana
Inserisco screenshot oppure descrizione dei 4 pannelli creati.

## 6. Trace Jaeger
Descrivo la trace visualizzata e i passaggi frontend -> backend che ho osservato.

## 7. Differenze tra metriche e trace
Spiego perché Prometheus e Jaeger non fanno la stessa cosa.

## 8. Ponte verso Azure
Spiego quali strumenti Azure sostituiranno concettualmente Prometheus, Jaeger e Grafana nei laboratori successivi.
```

---

## 20. Parte 14 - Consegna

Al termine del laboratorio devi consegnare:

* il progetto completo `demo-separated-observability/`
* il file `docs/evidence_lab00.md`

Se stai lavorando in un repository Git del corso, aggiungi e salva il lavoro:

```bash
git add .
git commit -m "LAB00 completato - app distribuita locale con observability stack separato"
git push
```

---

## 21. Risultato atteso finale

Al termine del laboratorio avrai realizzato in locale:

* una **applicazione distribuita** con frontend e backend
* uno **stack observability separato**
* metriche visibili in Prometheus
* dashboard visibili in Grafana
* trace distribuite visibili in Jaeger

Soprattutto, dovrai aver capito che:

* osservare un sistema non significa solo “vedere dei grafici”
* metriche, trace e dashboard hanno ruoli diversi
* una pipeline DevOps ha senso solo se sai **che cosa stai distribuendo** e **come lo osserverai dopo**

---

## 22. Estensione facoltativa

Se completi il laboratorio in anticipo, prova questa estensione.

### Obiettivo

Simulare errori dal backend e osservare l’impatto su metriche e trace.

### Modifica da fare

Nel file `app-stack/frontend/app.py`, dentro la funzione `/demo`, modifica la parte che costruisce la chiamata backend.

Sostituisci il blocco:

```python
response = requests.get(BACKEND_URL, timeout=5)
```

con un comportamento casuale come questo:

```python
target_url = BACKEND_URL
if random.random() < 0.2:
    target_url = "http://backend-service:8001/work-error"

response = requests.get(target_url, timeout=5)
```

### Poi esegui

Ricostruisci e riavvia il frontend:

```bash
cd ~/corso_obs/demo-separated-observability/app-stack
docker compose -f docker-compose.app.yml up --build -d
```

Genera traffico:

```bash
for i in {1..30}; do curl -s http://localhost:8000/demo > /dev/null; done
```

### Osserva

* aumento degli errori nelle metriche
* differenze nelle trace
* relazione tra sintomo aggregato e singola richiesta

---

## 23. Conclusione del laboratorio

In questo laboratorio hai visto la versione locale, pulita e didatticamente corretta di un sistema osservabile:

* applicazione separata
* stack di osservabilità separato
* metriche
* trace
* dashboard

Nei laboratori successivi manterrai la stessa logica, ma sostituirai lo stack locale con servizi Azure gestiti. 


