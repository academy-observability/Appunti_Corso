# Anomalia funzionamento Jeager

Il LAB14, così com’è scritto adesso, usa il percorso **legacy** basato su `JaegerExporter` verso `6831/udp` con `jaegertracing/all-in-one:latest` e variabili `JAEGER_AGENT_HOST` / `JAEGER_AGENT_PORT`. È esattamente quello che c’è nel file che mi hai condiviso 

Il problema è che oggi la strada documentata e raccomandata da Jaeger/OpenTelemetry è **OTLP**, non il vecchio exporter Jaeger. La documentazione Jaeger mostra `all-in-one` con porte OTLP `4317/4318`, e OpenTelemetry indica esplicitamente che il backend Jaeger riceve OTLP dalla v1.35 e che la migrazione raccomandata è dal Jaeger exporter all’OTLP exporter. Anche l’esempio HotROD ufficiale manda OTLP a `http://jaeger:4318`. ([Jaeger][1])


# Fix consigliato: passare a OTLP/HTTP


## 1) Correggi `obs-stack/docker-compose.obs.yml`

Sostituisci la sezione `jaeger` con questa:

```yaml
jaeger:
  image: jaegertracing/all-in-one:1.76.0
  container_name: jaeger
  environment:
    - COLLECTOR_OTLP_ENABLED=true
  ports:
    - "16686:16686"
    - "4318:4318"
  networks:
    - obs-net
```

### Perché

Jaeger `all-in-one` documenta OTLP su `4318` HTTP e `4317` gRPC; la doc di deployment mostra anche `COLLECTOR_OTLP_ENABLED=true` negli esempi di avvio. ([Jaeger][1])

---

## 2) Correggi i `requirements.txt`

### `app-stack/frontend/requirements.txt`

Usa questo:

```txt
flask==3.0.0
requests==2.31.0
prometheus_client==0.20.0
opentelemetry-api==1.24.0
opentelemetry-sdk==1.24.0
opentelemetry-exporter-otlp==1.24.0
opentelemetry-instrumentation-flask==0.45b0
opentelemetry-instrumentation-requests==0.45b0
```

### `app-stack/backend/requirements.txt`

Usa questo:

```txt
flask==3.0.0
prometheus_client==0.20.0
opentelemetry-api==1.24.0
opentelemetry-sdk==1.24.0
opentelemetry-exporter-otlp==1.24.0
opentelemetry-instrumentation-flask==0.45b0
```

### Cosa cambia

Togli:

```txt
opentelemetry-exporter-jaeger-thrift==1.21.0
```

e metti:

```txt
opentelemetry-exporter-otlp==1.24.0
```

OpenTelemetry Python documenta l’uso degli OTLP exporters e le relative variabili `OTEL_EXPORTER_OTLP_*`. ([OpenTelemetry][2])

---

## 3) Correggi `frontend/app.py`

### Sostituisci questi import

Togli:

```python
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
```

Metti:

```python
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
```

### Sostituisci queste variabili

Togli:

```python
JAEGER_AGENT_HOST = os.getenv("JAEGER_AGENT_HOST", "jaeger")
JAEGER_AGENT_PORT = int(os.getenv("JAEGER_AGENT_PORT", "6831"))
```

Metti:

```python
OTLP_ENDPOINT = os.getenv(
    "OTEL_EXPORTER_OTLP_TRACES_ENDPOINT",
    "http://jaeger:4318/v1/traces"
)
```

### Sostituisci la creazione exporter

Togli:

```python
jaeger_exporter = JaegerExporter(
    agent_host_name=JAEGER_AGENT_HOST,
    agent_port=JAEGER_AGENT_PORT
)

provider.add_span_processor(BatchSpanProcessor(jaeger_exporter))
```

Metti:

```python
otlp_exporter = OTLPSpanExporter(endpoint=OTLP_ENDPOINT)
provider.add_span_processor(BatchSpanProcessor(otlp_exporter))
```

---

## 4) Correggi `backend/app.py`

Stessa identica logica.

### Togli:

```python
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
```

### Metti:

```python
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
```

### Togli:

```python
JAEGER_AGENT_HOST = os.getenv("JAEGER_AGENT_HOST", "jaeger")
JAEGER_AGENT_PORT = int(os.getenv("JAEGER_AGENT_PORT", "6831"))
```

### Metti:

```python
OTLP_ENDPOINT = os.getenv(
    "OTEL_EXPORTER_OTLP_TRACES_ENDPOINT",
    "http://jaeger:4318/v1/traces"
)
```

### Togli:

```python
jaeger_exporter = JaegerExporter(
    agent_host_name=JAEGER_AGENT_HOST,
    agent_port=JAEGER_AGENT_PORT
)

provider.add_span_processor(BatchSpanProcessor(jaeger_exporter))
```

### Metti:

```python
otlp_exporter = OTLPSpanExporter(endpoint=OTLP_ENDPOINT)
provider.add_span_processor(BatchSpanProcessor(otlp_exporter))
```

---

## 5) Correggi `app-stack/docker-compose.app.yml`

### Nel servizio `frontend`, togli:

```yaml
- JAEGER_AGENT_HOST=jaeger
- JAEGER_AGENT_PORT=6831
```

e metti:

```yaml
- OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://jaeger:4318/v1/traces
```

### Nel servizio `backend`, togli:

```yaml
- JAEGER_AGENT_HOST=jaeger
- JAEGER_AGENT_PORT=6831
```

e metti:

```yaml
- OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://jaeger:4318/v1/traces
```

OTLP/HTTP usa endpoint tipo `http://host:4318/v1/traces`, che è esattamente il formato documentato dalla configurazione OTLP. ([OpenTelemetry][3])

---

# 6) Ricostruzione completa

Esegui tutto da zero.

## Ferma gli stack

```bash
cd ~/corso_obs/demo-separated-observability/app-stack
docker compose -f docker-compose.app.yml down

cd ../obs-stack
docker compose -f docker-compose.obs.yml down
```

## Riavvia observability stack

```bash
docker compose -f docker-compose.obs.yml up -d
```

## Riavvia app stack con rebuild

```bash
cd ../app-stack
docker compose -f docker-compose.app.yml up --build -d
```

---

# 7) Verifica finale

## Test applicativo

```bash
curl http://localhost:8000/health
curl http://localhost:8001/health
curl http://localhost:8000/demo
for i in {1..20}; do curl -s http://localhost:8000/demo > /dev/null; done
sleep 5
```

## Test Jaeger via API

```bash
curl -s http://localhost:16686/api/services
```

### Risultato atteso

Adesso dovresti vedere qualcosa di questo tipo:

```json
{"data":["frontend-service","backend-service","jaeger-all-in-one"],"total":3,"limit":0,"offset":0,"errors":null}
```

## Poi apri Jaeger UI

```text
http://localhost:16686
```

e seleziona:

* `frontend-service`
* `all`
* `Find Traces`

---

# Sintesi
** Sono quattro i file corretti** : `docker-compose.obs.yml`, `docker-compose.app.yml`, `frontend/app.py`, backend/app.py`.

[1]: https://www.jaegertracing.io/docs/1.76/deployment/ "Deployment | Jaeger"
[2]: https://opentelemetry.io/docs/languages/python/exporters/?utm_source=chatgpt.com "Exporters"
[3]: https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/?utm_source=chatgpt.com "OTLP Exporter Configuration"
