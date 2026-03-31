# LAB – Visualizzazione Observability con Grafana

## Obiettivo del laboratorio
In questo laboratorio introdurremo **Grafana** per visualizzare le metriche raccolte da Prometheus.

Al termine del laboratorio sarete in grado di:

- avviare Grafana
- collegarlo a Prometheus
- creare una dashboard
- visualizzare metriche in tempo reale
- analizzare il comportamento del sistema
- individuare un aumento degli errori

---

# Prerequisiti

Questo laboratorio richiede che siano già attivi:

- applicazione Python osservabile
- Prometheus
- generatore di traffico

Se lo stack non è attivo, eseguire:

```bash
docker compose up -d
```

---

# Architettura del laboratorio

```
Load Generator
     |
     v
Python Application
     |
     v
Prometheus
     |
     v
Grafana
```

Grafana verrà utilizzato per visualizzare le metriche raccolte da Prometheus.

---

# Step 1 – Accesso a Grafana

Aprire il browser e collegarsi a:

```
http://localhost:3000
```

Credenziali iniziali:

```
username: admin
password: admin
```

Al primo accesso Grafana richiederà di cambiare password.

---

# Step 2 – Aggiungere Prometheus come Data Source

Nel menu sinistro:

- cliccare **Connections**
- cliccare **Data Sources**
- cliccare **Add data source**
- selezionare **Prometheus**

Configurare il campo URL:

```
http://prometheus:9090
```

Cliccare su:

```
Save & Test
```

Se la configurazione è corretta apparirà il messaggio:

```
Data source is working
```

---

# Step 3 – Creazione della dashboard

Nel menu sinistro:

- cliccare **Dashboards**
- cliccare **New**
- cliccare **New Dashboard**
- cliccare **Add visualization**

Selezionare il data source:

```
Prometheus
```

---

# Step 4 – Pannello Request Rate

Inserire la query:

```
rate(http_requests_total[1m])
```

Impostare il titolo:

```
Request Rate
```

Cliccare **Apply**.

Questo pannello mostra il numero di richieste al secondo.

---

# Step 5 – Pannello Error Rate

Aggiungere una nuova visualizzazione.

Inserire la query:

```
sum(rate(http_requests_total{status="500"}[1m]))
```

Titolo pannello:

```
Error Rate
```

Cliccare **Apply**.

Questo pannello mostra il numero di errori nel tempo.

---

# Step 6 – Pannello Requests by Endpoint

Aggiungere una nuova visualizzazione.

Query:

```
sum by (endpoint) (rate(http_requests_total[1m]))
```

Titolo:

```
Requests by Endpoint
```

Questo pannello mostra il traffico per endpoint.

---

# Step 7 – Pannello Total Requests

Aggiungere una nuova visualizzazione.

Query:

```
sum(http_requests_total)
```

Titolo:

```
Total Requests
```

Questo pannello mostra il numero totale di richieste.

---

# Step 8 – Salvare la dashboard

Cliccare su **Save dashboard**.

Inserire nome:

```
SRE Demo Dashboard
```

---

# Step 9 – Avviare il traffico

Nel terminale eseguire:

```bash
python3 load.py
```

Osservare la dashboard Grafana.

I grafici inizieranno ad aggiornarsi in tempo reale.

---

# Step 10 – Simulare aumento errori

Aprire il file:

```
loadtest/load.py
```

Modificare la lista endpoint:

```
ENDPOINTS = ["/", "/error", "/error", "/error", "/slow"]
```

Salvare e riavviare il generatore:

```bash
python3 load.py
```

---

# Osservazione dei risultati

Osservare i pannelli:

- Request Rate
- Error Rate
- Requests by Endpoint
- Total Requests

Si noterà:

- aumento error rate
- incremento chiamate /error
- modifica comportamento sistema

---

# Concetti osservati

Questo laboratorio introduce:

- visualizzazione metriche
- dashboard osservability
- correlazione traffico-errori
- analisi grafica incidenti

---

# Output attesi

Al termine del laboratorio produrre:

- screenshot dashboard Grafana
- descrizione metriche osservate
- analisi aumento errori
- interpretazione comportamento sistema

---

# Conclusione

Grafana permette di trasformare le metriche raccolte da Prometheus in una vista operativa del sistema.

Questo laboratorio completa lo stack base di observability:

- applicazione osservabile
- raccolta metriche con Prometheus
- visualizzazione con Grafana

Da questo punto è possibile introdurre alerting e monitoring avanzato.

