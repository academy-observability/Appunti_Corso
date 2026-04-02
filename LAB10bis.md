# LAB10bis - Estensione didattica: dalla richiesta singola alla trace con più span

## Obiettivo del laboratorio

In questa estensione didattica del LAB10 farai un passo concettuale molto importante: distinguerai in modo chiaro tra:

- **richiesta singola osservata come evento**
- **trace composta da più span**

Nel LAB10 hai già lavorato con una tabella `requests_log` utile per:

- persistenza di eventi osservabili
- query SLI
- calcolo di error rate
- calcolo di latenza media
- analisi dei top endpoint

Quella tabella è corretta per introdurre la persistenza osservabile, ma non rappresenta ancora in modo fedele una **trace distribuita**.

Questa estensione serve proprio a colmare quel punto, introducendo una seconda tabella che modella una trace come insieme di più span correlati.

---

## Durata indicativa

**1 ora e 30 minuti - 2 ore**

---

## Quando svolgerlo

Questo laboratorio va svolto:

- **dopo LAB10**
- prima o parallelamente alla parte teorica più avanzata su tracing distribuito

Può essere usato come:

- estensione facoltativa
- approfondimento per partecipanti più rapidi
- chiarimento tecnico per evitare confusione tra `correlation_id`, `trace_id` e `span_id`

---

# PARTE 1 - Concetti fondamentali

# 1. Il limite del modello `requests_log`

Nel LAB10 hai usato una tabella simile a questa:

- `correlation_id`
- `path`
- `status`
- `duration_ms`
- `created_at`

Questa tabella è molto utile per fare query di tipo:

- total requests
- error rate
- top endpoint
- latenza media
- ricerca di una richiesta per identificatore

Per esempio, una query come:

```sql
SELECT *
FROM requests_log
WHERE correlation_id = 'req-004';
````

restituisce normalmente **una sola riga**.

Questo è coerente con il modello dati attuale, perché lì il `correlation_id` si comporta come identificatore di una singola richiesta o evento reso persistente.

---

# 2. Perché questo non è ancora tracing vero

Una **trace** non è normalmente una singola riga.

Una trace rappresenta il percorso di una richiesta attraverso uno o più componenti del sistema.

In uno scenario reale, una richiesta può attraversare:

* API gateway
* applicazione web
* servizio interno
* database
* cache
* servizio esterno

Ognuno di questi passaggi può generare uno **span**.

Quindi:

* una **trace** = insieme di più span
* ogni **span** = una singola operazione o segmento del percorso

---

# 3. Trace, span e correlation: distinzione corretta

## 3.1 Trace ID

Il **trace_id** identifica l’intera traccia di una richiesta distribuita.

Tutti gli span appartenenti alla stessa trace condividono lo stesso `trace_id`.

---

## 3.2 Span ID

Lo **span_id** identifica uno specifico segmento della trace.

Ogni span ha il proprio identificatore.

---

## 3.3 Parent Span ID

Il **parent_span_id** permette di collegare uno span allo span che lo ha generato.

Serve a ricostruire la gerarchia o la catena di chiamate.

---

## 3.4 Correlation ID

Nel linguaggio pratico e in molti sistemi reali, il termine `correlation_id` viene usato in modi diversi:

* a volte coincide con un request id
* a volte coincide con un trace id
* a volte è solo un identificatore generico di correlazione

Per evitare ambiguità didattica, in questo laboratorio userai i termini più precisi:

* `trace_id`
* `span_id`
* `parent_span_id`

---

# 4. Schema mentale corretto

## Modello semplice del LAB10

```text
una richiesta
  ↓
una riga in requests_log
```

## Modello esteso del LAB10bis

```text
una richiesta utente
  ↓
una trace
  ├── span 1: ingresso richiesta
  ├── span 2: logica applicativa
  ├── span 3: accesso DB
  └── span 4: costruzione risposta
```

Quindi:

```text
1 trace_id
+ più span_id
= trace distribuita
```

---

# 5. Esempio concettuale

Immagina una richiesta a:

```text
/time
```

Il sistema potrebbe produrre una trace così:

| trace_id  | span_id  | parent_span_id | service_name | operation_name   | status | duration_ms |
| --------- | -------- | -------------- | ------------ | ---------------- | ------ | ----------- |
| trace-004 | span-001 | NULL           | gateway      | incoming request | 200    | 5           |
| trace-004 | span-002 | span-001       | obsapp       | handle /time     | 200    | 40          |
| trace-004 | span-003 | span-002       | sql-db       | select request   | 200    | 18          |
| trace-004 | span-004 | span-002       | obsapp       | build response   | 200    | 6           |

Qui è chiaro che:

* la trace è una sola: `trace-004`
* gli span sono multipli
* i parent collegano i pezzi della catena
* puoi ricostruire il percorso della richiesta

Questo è molto più corretto dal punto di vista del tracing.

---

# 6. Perché questa estensione è utile nel percorso Observability

Questa estensione è utile perché evita una confusione molto comune:

* **logging strutturato** non è automaticamente **distributed tracing**
* un `request_id` non equivale sempre a una trace completa
* una riga di tabella può descrivere un evento, ma non necessariamente tutta la storia della richiesta

Con questa estensione i partecipanti capiscono che:

* `requests_log` serve bene per gli SLI
* una tabella `trace_spans` serve meglio per ragionare sulla trace

---

# 7. Cosa farai nel laboratorio

In questa estensione farai queste attività:

1. creerai una tabella `trace_spans`
2. inserirai dati di esempio relativi a più trace
3. interrogherai una trace singola
4. ordinerai gli span
5. vedrai la relazione parent-child
6. confronterai `requests_log` e `trace_spans`
7. capirai quando usare un modello e quando usare l’altro

---

# PARTE 2 - Laboratorio guidato passo-passo

## Prerequisiti

Per svolgere questa estensione ti servono:

* LAB10 completato
* accesso al database `obsdb`
* accesso a Query Editor oppure a un client SQL compatibile
* tabella `requests_log` già esistente

---

## Step 1 - Apri il database `obsdb`

Nel portale Azure:

1. apri il tuo SQL Server
2. apri il database `obsdb`
3. entra in **Query editor**

Autenticati con le credenziali amministrative SQL del laboratorio.

---

## Step 2 - Riprendi il modello semplice del LAB10

Esegui questa query:

```sql
SELECT *
FROM requests_log
WHERE correlation_id = 'req-004';
```

### Che cosa osservare

Con i dati del LAB10, questa query restituisce normalmente una sola riga.

### Che cosa devi capire

Questo modello rappresenta bene una **richiesta/evento reso persistente**, ma non ancora una trace composta da più span.

### Evidenza richiesta

Annota il risultato e scrivi una frase di commento:

* quante righe sono state restituite
* perché non si tratta ancora di tracing distribuito

---

## Step 3 - Crea la tabella estesa `trace_spans`

Esegui questa query:

```sql
CREATE TABLE trace_spans (
    id INT IDENTITY(1,1) PRIMARY KEY,
    trace_id NVARCHAR(100) NOT NULL,
    span_id NVARCHAR(100) NOT NULL,
    parent_span_id NVARCHAR(100) NULL,
    service_name NVARCHAR(100) NOT NULL,
    operation_name NVARCHAR(255) NOT NULL,
    path NVARCHAR(255) NULL,
    status INT NOT NULL,
    duration_ms INT NOT NULL,
    created_at DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
```

### Che cosa fa questa query

Crea una tabella progettata per rappresentare una trace tramite più span.

### Significato dei campi

* `trace_id` = identificatore comune dell’intera trace
* `span_id` = identificatore del singolo span
* `parent_span_id` = span padre
* `service_name` = componente che ha prodotto lo span
* `operation_name` = operazione eseguita
* `path` = endpoint, se applicabile
* `status` = esito dell’operazione
* `duration_ms` = durata dello span
* `created_at` = timestamp

### Evidenza richiesta

Salva screenshot o annota l’avvenuta creazione della tabella.

---

## Step 4 - Inserisci dati di esempio

Esegui questa query:

```sql
INSERT INTO trace_spans (trace_id, span_id, parent_span_id, service_name, operation_name, path, status, duration_ms)
VALUES
('trace-001', 'span-001', NULL,       'gateway', 'incoming request', '/health', 200, 4),
('trace-001', 'span-002', 'span-001', 'obsapp',  'handle /health',   '/health', 200, 12),

('trace-002', 'span-003', NULL,       'gateway', 'incoming request', '/time',   200, 5),
('trace-002', 'span-004', 'span-003', 'obsapp',  'handle /time',     '/time',   200, 40),
('trace-002', 'span-005', 'span-004', 'sql-db',  'select time data', '/time',   200, 18),
('trace-002', 'span-006', 'span-004', 'obsapp',  'build response',   '/time',   200, 6),

('trace-003', 'span-007', NULL,       'gateway', 'incoming request', '/nope',   404, 3),
('trace-003', 'span-008', 'span-007', 'obsapp',  'handle /nope',     '/nope',   404, 8),

('trace-004', 'span-009', NULL,       'gateway', 'incoming request', '/time',   200, 6),
('trace-004', 'span-010', 'span-009', 'obsapp',  'handle /time',     '/time',   200, 55),
('trace-004', 'span-011', 'span-010', 'sql-db',  'select request',   '/time',   200, 22),
('trace-004', 'span-012', 'span-010', 'obsapp',  'serialize result', '/time',   200, 7);
```

### Che cosa rappresentano questi dati

Hai ora:

* più trace
* ogni trace con uno o più span
* relazioni padre-figlio
* servizi diversi
* operazioni diverse
* path e durata

### Evidenza richiesta

Annota che il dataset è stato inserito correttamente.

---

## Step 5 - Visualizza tutti gli span

Esegui:

```sql
SELECT *
FROM trace_spans;
```

### Che cosa osservare

Verifica la presenza di:

* più `trace_id`
* più `span_id`
* `parent_span_id`
* servizi diversi
* durate diverse

### Evidenza richiesta

Copia un estratto dell’output o esegui uno screenshot.

---

## Step 6 - Interroga una trace singola

Esegui:

```sql
SELECT *
FROM trace_spans
WHERE trace_id = 'trace-004'
ORDER BY id;
```

### Che cosa osservare

Questa volta la query restituisce **più righe**, non una sola.

### Cosa devi capire

Adesso stai vedendo correttamente una trace composta da più span.

### Evidenza richiesta

Copia il risultato nel file evidenze.

---

## Step 7 - Visualizza solo gli elementi essenziali della trace

Esegui:

```sql
SELECT
    trace_id,
    span_id,
    parent_span_id,
    service_name,
    operation_name,
    status,
    duration_ms
FROM trace_spans
WHERE trace_id = 'trace-004'
ORDER BY id;
```

### Che cosa osservare

Questa query ti mostra la struttura logica della trace in modo più leggibile.

### Evidenza richiesta

Copia il risultato nel file evidenze.

---

## Step 8 - Calcola la durata totale della trace

Esegui:

```sql
SELECT
    trace_id,
    SUM(duration_ms) AS total_trace_duration_ms
FROM trace_spans
WHERE trace_id = 'trace-004'
GROUP BY trace_id;
```

### Che cosa misura

Somma la durata di tutti gli span della trace.

### Nota didattica importante

In sistemi reali la durata totale percepita della trace non sempre coincide con la semplice somma di tutti gli span, perché alcuni span possono essere paralleli o sovrapposti.

In questo laboratorio, però, la somma è utile come modello introduttivo.

### Evidenza richiesta

Copia il risultato nel file evidenze.

---

## Step 9 - Conta quanti span appartengono a ogni trace

Esegui:

```sql
SELECT
    trace_id,
    COUNT(*) AS span_count
FROM trace_spans
GROUP BY trace_id
ORDER BY span_count DESC;
```

### Che cosa misura

Mostra quante operazioni compongono ogni trace.

### Perché è utile

Aiuta a capire che una trace può essere semplice o più articolata.

### Evidenza richiesta

Copia il risultato nel file evidenze.

---

## Step 10 - Individua gli span radice

Esegui:

```sql
SELECT
    trace_id,
    span_id,
    service_name,
    operation_name
FROM trace_spans
WHERE parent_span_id IS NULL;
```

### Che cosa misura

Mostra gli span iniziali, cioè quelli senza padre.

### Cosa devi capire

Gli span radice rappresentano spesso il punto di ingresso della richiesta nel sistema.

### Evidenza richiesta

Copia il risultato nel file evidenze.

---

## Step 11 - Individua gli span più lenti

Esegui:

```sql
SELECT
    trace_id,
    span_id,
    service_name,
    operation_name,
    duration_ms
FROM trace_spans
ORDER BY duration_ms DESC;
```

### Che cosa misura

Mostra gli span ordinati per durata decrescente.

### Perché è utile

In un sistema reale, gli span più lenti sono spesso i primi indizi da analizzare.

### Evidenza richiesta

Copia il risultato nel file evidenze.

---

## Step 12 - Analizza il contributo dei servizi

Esegui:

```sql
SELECT
    service_name,
    COUNT(*) AS total_spans,
    AVG(CAST(duration_ms AS FLOAT)) AS avg_span_duration_ms
FROM trace_spans
GROUP BY service_name
ORDER BY avg_span_duration_ms DESC;
```

### Che cosa misura

Mostra:

* quanti span genera ogni servizio
* la durata media degli span per servizio

### Perché è utile

Ti fa capire come diversi componenti contribuiscono alla trace complessiva.

### Evidenza richiesta

Copia il risultato nel file evidenze.

---

## Step 13 - Confronta `requests_log` e `trace_spans`

Scrivi una breve nota nel file evidenze rispondendo a queste domande:

1. Che cosa rappresenta una riga di `requests_log`?
2. Che cosa rappresenta una riga di `trace_spans`?
3. Quando è più utile usare `requests_log`?
4. Quando è più utile usare `trace_spans`?

### Risposta attesa a livello concettuale

* `requests_log` rappresenta bene eventi o richieste da usare per SLI semplici
* `trace_spans` rappresenta meglio il percorso distribuito di una richiesta

---

## Step 14 - Crea il file delle evidenze

Crea:

```text
docs/evidence_lab10bis_trace.md
```

Usa questa struttura:

````md
# LAB10bis - Evidence

## 1. Modello semplice
### Query su requests_log
```sql
SELECT *
FROM requests_log
WHERE correlation_id = 'req-004';
````

* Numero righe restituite:
* Commento:

## 2. Tabella trace_spans

* Creata correttamente: SÌ/NO

## 3. Dataset inserito

* Inserito correttamente: SÌ/NO

## 4. Query eseguite

### Tutti gli span

[incollare estratto]

### Trace-004

[incollare risultato]

### Struttura essenziale della trace

[incollare risultato]

### Durata totale trace

[incollare risultato]

### Numero span per trace

[incollare risultato]

### Span radice

[incollare risultato]

### Span più lenti

[incollare risultato]

### Analisi per servizio

[incollare risultato]

## 5. Sintesi finale

* Differenza tra correlation_id e trace_id:
* Differenza tra richiesta singola e trace:
* Quando usare requests_log:
* Quando usare trace_spans:
* Che cosa ho capito sugli span:

````

---

## Step 15 - Verifica la struttura locale del laboratorio

In WSL esegui:

```bash
ls -R
````

Verifica di avere almeno:

```text
docs/
cmdlog_lab10bis.txt
```

e dentro `docs`:

```text
evidence_lab10bis_trace.md
```

---

## Step 16 - Consegna nel repository Git

Esegui:

```bash
git add docs/evidence_lab10bis_trace.md
git commit -m "[LAB10bis] Estensione trace con span completata"
git push
```

---

# PARTE 3 - Checkpoint e criteri di completamento

## Checkpoint #1

Hai verificato che la query su `requests_log` con `correlation_id = 'req-004'` restituisce una singola riga.

## Checkpoint #2

Hai creato la tabella `trace_spans`.

## Checkpoint #3

Hai inserito dati di esempio con più trace e più span.

## Checkpoint #4

Hai interrogato una trace singola e ottenuto più righe.

## Checkpoint #5

Hai identificato span radice, span lenti e distribuzione per servizio.

## Checkpoint #6

Hai chiarito la differenza tra modello semplice e modello trace/span.

---

# Criteri di completamento

Il laboratorio è completato se:

* la tabella `trace_spans` esiste
* i dati di esempio sono stati inseriti
* le query principali sono state eseguite
* il file `docs/evidence_lab10bis_trace.md` è completo
* il partecipante sa spiegare con parole proprie la differenza tra:

  * richiesta singola
  * trace
  * span

---

# Errori comuni da evitare

## Errore 1 - Pensare che correlation_id e trace_id siano sempre la stessa cosa

Non è sempre vero. Dipende dal modello adottato.

## Errore 2 - Credere che una trace sia una singola riga

Una trace è normalmente composta da più span.

## Errore 3 - Pensare che tutti gli span siano indipendenti

Gli span hanno relazioni padre-figlio.

## Errore 4 - Usare `requests_log` per spiegare il tracing distribuito senza precisazioni

`requests_log` è utile, ma non basta da solo per modellare una trace reale.

## Errore 5 - Sommare sempre le durate come se fosse la durata reale percepita

In sistemi reali alcuni span possono sovrapporsi. Qui la somma è solo un’approssimazione didattica.

---

# Conclusione

Questa estensione didattica ti ha permesso di fare un passaggio concettuale molto importante:

* da una richiesta singola resa persistente come evento
* a una trace composta da più span correlati

Dopo questo laboratorio devi avere chiaro che:

* `requests_log` è ottimo per SLI semplici e analisi per richiesta
* `trace_spans` è più adatto per spiegare e analizzare il tracing
* `trace_id` identifica l’intera trace
* `span_id` identifica il singolo segmento
* `parent_span_id` permette di ricostruire la catena delle operazioni

Questo chiarimento rende molto più solida la comprensione dei concetti di Observability e prepara meglio eventuali approfondimenti futuri su OpenTelemetry e distributed tracing reale.

```
```
