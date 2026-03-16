# Spiegazione dell’app Python del laboratorio

Questa applicazione è un **piccolo server HTTP scritto in Python**.
Serve per simulare un servizio web molto semplice, ma abbastanza realistico da permetterci di osservare:

* richieste HTTP in ingresso
* endpoint diversi
* codici di risposta HTTP
* tempi di risposta
* logging strutturato in formato JSON
* identificazione univoca delle richieste tramite `request_id`

In pratica, è una mini-app pensata per fare test, troubleshooting e osservabilità.

---

# 1. Import delle librerie

```python
import json
import os
import time
import uuid
from datetime import datetime
from http.server import BaseHTTPRequestHandler, HTTPServer
```

## Cosa fanno questi import

### `import json`

Serve per lavorare con il formato **JSON**.

Lo useremo per:

* convertire dizionari Python in testo JSON da inviare al client
* leggere JSON ricevuto nel body di una richiesta POST
* scrivere log in formato JSON

---

### `import os`

Serve per interagire con il sistema operativo.

Lo useremo per:

* leggere variabili d’ambiente
* creare cartelle se non esistono

---

### `import time`

Serve per misurare il tempo.

Lo useremo per:

* salvare l’istante iniziale della richiesta
* calcolare la durata della risposta in millisecondi

---

### `import uuid`

Serve a generare identificativi univoci.

Ogni richiesta riceverà un proprio `request_id`, utile per:

* tracciare una richiesta specifica
* collegare risposta HTTP e log
* simulare una pratica comune nei sistemi distribuiti

---

### `from datetime import datetime`

Serve per ottenere data e ora correnti.

Lo useremo per:

* scrivere timestamp nei log
* restituire l’ora UTC nell’endpoint `/time`

---

### `from http.server import BaseHTTPRequestHandler, HTTPServer`

Queste sono classi standard di Python per creare un server HTTP.

* `HTTPServer` = il server vero e proprio
* `BaseHTTPRequestHandler` = la classe da estendere per definire come gestire le richieste

---

# 2. Configurazione iniziale

```python
PORT = int(os.environ.get("PORT", "9000"))
LOG_PATH = os.environ.get("LOG_PATH", "logs/app.log")
```

Qui l’app legge due configurazioni.

---

## `PORT`

```python
PORT = int(os.environ.get("PORT", "9000"))
```

Significa:

* prova a leggere la variabile d’ambiente `PORT`
* se non esiste, usa il valore predefinito `"9000"`
* convertilo in intero con `int(...)`

### Esempio

Se non imposti nulla:

```python
PORT = 9000
```

Quindi il server ascolterà su:

```text
http://localhost:9000
```

Se invece lanci l’app con una variabile d’ambiente:

```bash
PORT=8080 python3 app.py
```

allora il server userà la porta `8080`.

---

## `LOG_PATH`

```python
LOG_PATH = os.environ.get("LOG_PATH", "logs/app.log")
```

Significa:

* prova a leggere la variabile d’ambiente `LOG_PATH`
* se non esiste, usa `logs/app.log`

Quindi, per default, i log saranno scritti nel file:

```text
logs/app.log
```

---

# 3. Funzione `log_line(...)`

```python
def log_line(level, request_id, method, path, status, duration_ms, extra=None):
    record = {
        "ts": datetime.utcnow().isoformat() + "Z",
        "level": level,
        "request_id": request_id,
        "method": method,
        "path": path,
        "status": status,
        "duration_ms": duration_ms,
    }
    if extra:
        record.update(extra)

    os.makedirs(os.path.dirname(LOG_PATH), exist_ok=True)
    with open(LOG_PATH, "a", encoding="utf-8") as f:
        f.write(json.dumps(record) + "\n")
```

Questa funzione serve a **scrivere una riga di log** nel file.

---

## Parametri della funzione

* `level` → livello log, ad esempio `INFO` o `WARN`
* `request_id` → identificativo univoco della richiesta
* `method` → metodo HTTP, ad esempio `GET` o `POST`
* `path` → percorso richiesto, ad esempio `/health`
* `status` → codice HTTP restituito, ad esempio `200`, `404`, `400`
* `duration_ms` → durata della richiesta in millisecondi
* `extra` → eventuali dati aggiuntivi

---

## Creazione del record

```python
record = {
    "ts": datetime.utcnow().isoformat() + "Z",
    "level": level,
    "request_id": request_id,
    "method": method,
    "path": path,
    "status": status,
    "duration_ms": duration_ms,
}
```

Qui viene creato un dizionario Python con le informazioni principali della richiesta.

### Campi importanti

* `ts` = timestamp UTC
* `level` = livello del log
* `request_id` = ID univoco
* `method` = metodo HTTP
* `path` = endpoint richiesto
* `status` = codice di risposta
* `duration_ms` = durata della richiesta

---

## Gestione di dati extra

```python
if extra:
    record.update(extra)
```

Se sono presenti campi aggiuntivi, vengono aggiunti al record.

Esempio: in caso di JSON non valido potremmo aggiungere:

```python
{"error": "bad_json"}
```

---

## Creazione della directory dei log

```python
os.makedirs(os.path.dirname(LOG_PATH), exist_ok=True)
```

Questa riga crea la cartella che contiene il file di log, se non esiste già.

Esempio:

* `LOG_PATH = "logs/app.log"`
* `os.path.dirname(LOG_PATH)` restituisce `"logs"`

Quindi viene creata la directory `logs/`.

`exist_ok=True` evita errori se la cartella esiste già.

---

## Scrittura nel file

```python
with open(LOG_PATH, "a", encoding="utf-8") as f:
    f.write(json.dumps(record) + "\n")
```

Questa parte:

* apre il file in modalità append (`"a"`)
* converte il dizionario in una stringa JSON
* scrive una riga nel file

Ogni richiesta produce una riga di log.

### Esempio di log

```json
{"ts": "2026-03-16T12:00:00.123456Z", "level": "INFO", "request_id": "abc-123", "method": "GET", "path": "/health", "status": 200, "duration_ms": 3}
```

Questo è un **log strutturato**.
È molto utile perché può essere letto facilmente da strumenti automatici.

---

# 4. Classe `Handler`

```python
class Handler(BaseHTTPRequestHandler):
```

Questa classe definisce **come il server risponde alle richieste HTTP**.

Ereditando da `BaseHTTPRequestHandler`, possiamo personalizzare il comportamento per:

* richieste GET
* richieste POST

---

# 5. Metodo `_send_json(...)`

```python
def _send_json(self, status, obj, request_id, start_ts, level="INFO", extra=None):
    payload = json.dumps(obj).encode("utf-8")
    self.send_response(status)
    self.send_header("Content-Type", "application/json")
    self.send_header("X-Request-Id", request_id)
    self.send_header("Content-Length", str(len(payload)))
    self.end_headers()
    self.wfile.write(payload)

    duration_ms = int((time.time() - start_ts) * 1000)
    log_line(level, request_id, self.command, self.path, status, duration_ms, extra=extra)
```

Questo è un metodo di supporto interno.
Serve per evitare di riscrivere ogni volta lo stesso codice.

In pratica fa due cose:

1. invia una risposta JSON al client
2. registra un log della richiesta

---

## `payload = json.dumps(obj).encode("utf-8")`

* `obj` è un dizionario Python
* `json.dumps(obj)` lo trasforma in testo JSON
* `.encode("utf-8")` lo trasforma in bytes, perché la risposta HTTP viene inviata come bytes

---

## `self.send_response(status)`

Invia il codice HTTP.

Esempi:

* `200` = OK
* `404` = Not Found
* `400` = Bad Request

---

## Header HTTP

```python
self.send_header("Content-Type", "application/json")
self.send_header("X-Request-Id", request_id)
self.send_header("Content-Length", str(len(payload)))
```

### `Content-Type`

Dice al client che il contenuto è JSON.

### `X-Request-Id`

Aggiunge un header personalizzato con l’identificativo della richiesta.

Questo è molto utile per correlare:

* ciò che vede il client
* ciò che compare nei log

### `Content-Length`

Indica la lunghezza del payload in byte.

---

## `self.end_headers()`

Segna la fine degli header HTTP.

Dopo questo punto inizia il body della risposta.

---

## `self.wfile.write(payload)`

Scrive il contenuto della risposta HTTP.

---

## Calcolo durata richiesta

```python
duration_ms = int((time.time() - start_ts) * 1000)
```

* `time.time()` dà il timestamp corrente
* `start_ts` è il tempo salvato all’inizio della richiesta
* la differenza è la durata in secondi
* moltiplicando per `1000` otteniamo millisecondi

---

## Scrittura del log

```python
log_line(level, request_id, self.command, self.path, status, duration_ms, extra=extra)
```

Dopo aver risposto al client, l’app salva anche il log della richiesta.

---

# 6. Metodo `do_GET(self)`

```python
def do_GET(self):
    start_ts = time.time()
    request_id = str(uuid.uuid4())
```

Questo metodo viene eseguito automaticamente quando il client invia una richiesta `GET`.

All’inizio:

* salva il timestamp iniziale
* genera un `request_id` univoco

---

## Endpoint `/health`

```python
if self.path == "/health":
    self._send_json(200, {"status": "ok"}, request_id, start_ts)
    return
```

Se il percorso richiesto è `/health`, l’app risponde con:

```json
{"status": "ok"}
```

e codice HTTP `200`.

### A cosa serve `/health`

È un endpoint tipico per i controlli di salute del servizio.

Può essere usato da:

* esseri umani
* script
* monitor
* sistemi di orchestrazione

---

## Endpoint `/time`

```python
if self.path == "/time":
    self._send_json(200, {"utc": datetime.utcnow().isoformat() + "Z"}, request_id, start_ts)
    return
```

Se il percorso è `/time`, l’app risponde con l’ora UTC corrente.

Esempio:

```json
{"utc": "2026-03-16T12:10:00.000000Z"}
```

### A cosa serve

Serve per mostrare una risposta dinamica, non sempre uguale.

---

## Qualsiasi altro percorso GET

```python
self._send_json(404, {"error": "not found", "path": self.path}, request_id, start_ts)
```

Se il path non è `/health` e non è `/time`, l’app restituisce `404`.

Esempio:

richiesta:

```text
GET /abc
```

risposta:

```json
{"error": "not found", "path": "/abc"}
```

---

# 7. Metodo `do_POST(self)`

```python
def do_POST(self):
    start_ts = time.time()
    request_id = str(uuid.uuid4())
```

Questo metodo gestisce le richieste POST.

Anche qui vengono creati:

* timestamp iniziale
* request id univoco

---

## Controllo endpoint `/echo`

```python
if self.path != "/echo":
    self._send_json(404, {"error": "not found", "path": self.path}, request_id, start_ts)
    return
```

La POST è supportata solo su `/echo`.

Se il client fa POST su un altro path, riceve `404`.

---

## Lettura del body della richiesta

```python
try:
    length = int(self.headers.get("Content-Length", "0"))
    raw = self.rfile.read(length).decode("utf-8")
    data = json.loads(raw) if raw else None
```

Questa parte legge il contenuto inviato dal client.

### `Content-Length`

Indica quanti byte leggere.

### `self.rfile.read(length)`

Legge il body della richiesta.

### `.decode("utf-8")`

Converte i bytes in stringa.

### `json.loads(raw)`

Converte il testo JSON in oggetto Python.

---

## Gestione errore JSON non valido

```python
except Exception:
    self._send_json(400, {"error": "bad json"}, request_id, start_ts, level="WARN", extra={"error": "bad_json"})
    return
```

Se il body non contiene un JSON valido:

* risposta HTTP `400`
* body JSON: `{"error": "bad json"}`
* livello log: `WARN`
* aggiunta informazione extra `error=bad_json`

Questo è utile per vedere nei log che l’errore dipende dall’input del client.

---

## Risposta corretta su `/echo`

```python
self._send_json(200, {"echo": data}, request_id, start_ts)
```

Se il JSON è valido, il server lo restituisce indietro.

Esempio:

richiesta:

```json
{"name": "Anna", "role": "student"}
```

risposta:

```json
{"echo": {"name": "Anna", "role": "student"}}
```

---

# 8. Funzione `main()`

```python
def main():
    server = HTTPServer(("0.0.0.0", PORT), Handler)
    print(f"Serving on http://localhost:{PORT}")
    server.serve_forever()
```

Questa funzione avvia il server.

---

## `HTTPServer(("0.0.0.0", PORT), Handler)`

Qui viene creato il server HTTP.

### `0.0.0.0`

Vuol dire: ascolta su tutte le interfacce di rete disponibili.

### `PORT`

È la porta configurata prima.

### `Handler`

È la classe che gestisce le richieste.

---

## `print(...)`

Stampa un messaggio di avvio, ad esempio:

```text
Serving on http://localhost:9000
```

---

## `server.serve_forever()`

Il server entra in un ciclo infinito e rimane in ascolto delle richieste.

---

# 9. Punto di ingresso del programma

```python
if __name__ == "__main__":
    main()
```

Questa è la parte che dice:

* se il file viene eseguito direttamente
* allora chiama `main()`

È il classico punto di ingresso di un programma Python.

---

# 10. Cosa fa l’app nel complesso

Questa applicazione:

* avvia un server HTTP
* espone endpoint GET e POST
* restituisce risposte JSON
* genera un `request_id` per ogni richiesta
* calcola il tempo di risposta
* scrive log strutturati in JSON

---

# 11. Endpoint disponibili

## `GET /health`

Risponde:

```json
{"status": "ok"}
```

Serve per health check.

---

## `GET /time`

Risponde con data e ora UTC.

---

## `POST /echo`

Accetta un body JSON e lo restituisce indietro.

---

## Altri path

Restituiscono `404`.

---

# 12. Perché questa app è utile nel laboratorio

Questa app è didatticamente utile perché ci permette di osservare concetti reali ma semplici:

## Logging strutturato

Ogni richiesta produce un record JSON nel file di log.

## Correlazione

Il `request_id` compare sia nella risposta HTTP sia nei log.

## HTTP status code

Vediamo in pratica `200`, `400`, `404`.

## Misura della latenza

Per ogni richiesta viene salvata la durata in millisecondi.

## Distinzione tra errori

* `404` = endpoint inesistente
* `400` = input JSON non valido

---

# 13. Esempi di richieste da provare

## Health check

```bash
curl http://localhost:9000/health
```

---

## Ora UTC

```bash
curl http://localhost:9000/time
```

---

## Echo corretto

```bash
curl -X POST http://localhost:9000/echo \
  -H "Content-Type: application/json" \
  -d '{"msg":"ciao"}'
```

---

## Echo con JSON errato

```bash
curl -X POST http://localhost:9000/echo \
  -H "Content-Type: application/json" \
  -d '{msg:ciao}'
```

Qui il JSON non è valido e l’app dovrebbe rispondere con `400`.

---

## Path inesistente

```bash
curl http://localhost:9000/qualcosa
```

Qui dovrebbe rispondere con `404`.

---

# 14. Cosa osservare nei log

Nel file `logs/app.log` troverai una riga JSON per ogni richiesta.

Controlla in particolare:

* `ts`
* `level`
* `request_id`
* `method`
* `path`
* `status`
* `duration_ms`

Questi campi sono la base dell’osservabilità applicativa.

---

# 15. Riassunto finale

Questa app non è complessa, ma è molto utile per imparare.

In concreto ci insegna:

* come funziona un piccolo server HTTP
* come si definiscono endpoint GET e POST
* come si inviano risposte JSON
* come si leggono richieste POST
* come si gestiscono errori
* come si registra un log strutturato
* come si misura il tempo di risposta
* come si introduce un `request_id` per tracciare ogni richiesta

È quindi una mini-app ideale per fare esercizi di:

* troubleshooting
* logging
* metriche di base
* osservabilità

Se dovete presentarla in modo sintetico, potete definirla così:

> È un micro-servizio HTTP minimale scritto in Python che espone alcuni endpoint JSON, registra log strutturati per ogni richiesta e permette di fare test pratici di observability, troubleshooting e analisi dei flussi HTTP.
