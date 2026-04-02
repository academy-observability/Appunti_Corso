# Sequenza corretta del **Modulo 2**:

```text
LAB08    Azure Foundations orientato all’Observability
LAB09    ACR + ACI: deploy del container osservabile
LAB09bis Monitoraggio delle risorse Azure dal portale
LAB10    Azure SQL: persistenza dati osservabili e query SLI
LAB11    Log Analytics + KQL
LAB12    Alerting
LAB13    Workbooks / Dashboard
LAB14    Revisione architettura + cleanup
```
# LAB10 - Azure SQL: persistenza dei dati osservabili e query SLI

## Obiettivo del laboratorio

In questo laboratorio farai un passo fondamentale nel percorso Observability su Azure: i segnali applicativi non verranno più solo letti dai log del container, ma saranno (in genere con un processo batch schedulato ad intervalli regolari) **resi persistenti in un database relazionale** e interrogati con query SQL.

La traccia del modulo prevede infatti di:

- creare un **Azure SQL Server**
- creare il database **obsdb**
- creare una tabella **requests_log**
- memorizzare campi osservabili come:
  - `correlation_id`
  - `status`
  - `duration_ms`
- eseguire query per calcolare indicatori come:
  - total requests
  - error rate
  - top endpoint
  - latenza media 

Questo laboratorio è molto importante perché segna il passaggio da:

- osservazione “grezza” del comportamento applicativo
a
- analisi strutturata e interrogabile dei dati osservabili

---

## Durata indicativa

**4 ore**

---

## Cost control - gestione dei costi

Nel LAB10 devi mantenere un comportamento prudente sui costi, come già richiesto per l’intero modulo:

- usare SKU minimi
- evitare configurazioni Premium
- usare opzioni semplici e adeguate a un laboratorio didattico 

### Modalità percorso continuativo
Non eliminare il Resource Group fino al LAB14. Il database e il server SQL potranno essere utili anche nelle verifiche successive. 

---

# PARTE 1 - Concetti fondamentali, teoria e collegamento con Observability

# 1. Perché serve la persistenza dei dati osservabili

Nel LAB09 hai già ottenuto segnali osservabili dal servizio in esecuzione su ACI:

- richieste HTTP
- endpoint chiamati
- status code
- `request_id`
- `duration_ms`
- log applicativi

Questi segnali però, se li guardi solo nei log runtime, hanno diversi limiti:

- sono poco strutturati
- non sono sempre facili da aggregare
- non sono il modo migliore per fare analisi ripetibili
- diventano difficili da interrogare se vuoi statistiche o indicatori

Per questo nel LAB10 introduci la **persistenza relazionale**.

In pratica:

- prendi eventi significativi
- li salvi in tabella
- li interroghi in modo strutturato
- ricavi indicatori osservabili più stabili

---

# 2. Che cosa significa “persistenza” in questo contesto

Persistenza significa che i dati non restano solo:

- nella memoria dell’app
- nello stdout del container
- in un log temporaneo

ma vengono salvati in una struttura duratura, interrogabile e riutilizzabile.

Nel LAB10 la persistenza avviene in **Azure SQL Database**.

Questo cambia la qualità del lavoro osservability, perché ti permette di passare da:

> “vedo delle righe di log”

a

> “posso misurare il comportamento del servizio con query ripetibili”

---

# 3. Perché usare Azure SQL in un percorso Observability

A prima vista qualcuno potrebbe pensare:

> “Ma Observability non doveva essere log, metriche e alert?”

Sì, ma nel mondo reale i segnali osservabili possono essere anche:

- dati di audit
- tabelle di richieste
- eventi resi persistenti
- record di esecuzione
- storici di transazioni
- dati funzionali da cui ricavare indicatori operativi

Azure SQL qui non serve per “fare database” in astratto.  
Serve per mostrare che:

- un sistema osservabile può registrare eventi in forma strutturata
- da questi eventi si possono ricavare indicatori di servizio
- SQL è uno strumento potente per analizzare qualità e comportamento del sistema

---

# 4. Che cos’è un SLI

La traccia del LAB10 richiede esplicitamente query per indicatori come total requests, error rate, top endpoint e latenza media. Questi sono esempi di **Service Level Indicators**, o **SLI**. 

## 4.1 Definizione semplice
Un **SLI** è una misura osservabile della qualità o del comportamento di un servizio.

Esempi:

- percentuale di richieste riuscite
- latenza media
- numero di errori
- endpoint più chiamati

---

## 4.2 Perché gli SLI sono utili
Perché ti permettono di uscire dal livello aneddotico:

- “mi sembra lento”
- “forse ci sono errori”
- “credo che /time venga chiamato spesso”

e di passare a misure verificabili:

- error rate = 12%
- latenza media = 83 ms
- endpoint più invocato = `/health`

---

# 5. Dati osservabili che userai nella tabella `requests_log`

La tabella che userai contiene campi che derivano direttamente dalla logica di osservabilità applicativa del modulo.

I campi più importanti sono:

- `correlation_id`
- `path`
- `status`
- `duration_ms`
- `created_at` o campo temporale equivalente

## 5.1 `correlation_id`
Identifica una singola richiesta o un singolo evento.

Serve a:

- distinguere una richiesta dall’altra
- collegare dati resi persistenti e log applicativi
- supportare troubleshooting e ricostruzione dei casi

---

## 5.2 `path`
Indica l’endpoint richiesto.

Serve a capire:

- quale funzione dell’app è stata usata
- quali endpoint sono più frequenti
- dove si concentrano errori o latenze

---

## 5.3 `status`
Indica l’esito della richiesta, ad esempio:

- `200`
- `404`
- `500`

Serve per:

- calcolare error rate
- distinguere successi e fallimenti
- capire la qualità del servizio

---

## 5.4 `duration_ms`
Indica la durata della richiesta in millisecondi.

Serve per:

- misurare latenza
- confrontare endpoint
- individuare degradi di performance

---

## 5.5 Timestamp
Il campo temporale serve per:

- sapere quando è avvenuto l’evento
- filtrare finestre temporali
- costruire trend
- aggregare nel tempo

---

# 6. Perché query SQL e non solo lettura tabellare

La vera competenza che stai costruendo qui non è “saper vedere una tabella”.

È saper porre domande operative al sistema.

Esempi:

- quante richieste ci sono state?
- quante sono andate in errore?
- qual è la latenza media?
- quale endpoint è stato più usato?

Queste domande diventano query SQL.

In pratica SQL, in questo contesto, è un linguaggio per fare **analisi osservability sui dati resi persistenti**.

---

# 7. Pipeline mentale del LAB10

Il LAB10 può essere letto così:

```text
richieste applicative
  ↓
eventi osservabili
  ↓
persistenza in Azure SQL
  ↓
tabella requests_log
  ↓
query SQL
  ↓
SLI
````

Oppure in forma più concreta:

```text
ACI / applicazione
  ↓
requests_log
  ├── correlation_id
  ├── path
  ├── status
  ├── duration_ms
  └── created_at
  ↓
query
  ├── total requests
  ├── error rate
  ├── top endpoint
  └── avg latency
```

Questa è esattamente la logica espressa dalla traccia del modulo.

---

# 8. Che cosa osserverai davvero in questo laboratorio

Nel LAB10 osservi almeno tre livelli distinti:

## Livello 1 - Stato risorsa Azure

Verifichi che esistano:

* SQL Server
* database `obsdb`

## Livello 2 - Struttura dati osservabile

Verifichi la presenza della tabella `requests_log` e dei campi chiave.

## Livello 3 - Indicatori di servizio

Usi query SQL per misurare:

* volume richieste
* errori
* latenza
* distribuzione endpoint

Questa è observability applicativa persistita.

---

# 9. Dove opererai

Nel LAB10 userai principalmente:

* **Azure Portal** per creare SQL Server e database
* **Azure Cloud Shell o WSL** per eseguire comandi `az`
* **Azure Portal Query Editor** oppure client SQL compatibile per eseguire query
* repository locale per evidenze e file ambiente

---

# 10. Che cosa imparerai davvero

Alla fine del laboratorio dovrai aver capito che:

* i segnali osservabili possono essere resi persistenti in forma strutturata
* SQL è uno strumento molto utile per l’analisi operativa
* gli SLI possono nascere anche da query su dati applicativi
* `correlation_id`, `status`, `duration_ms` non sono solo dettagli di log, ma colonne fondamentali per fare analisi

---

# PARTE 2 - Laboratorio guidato passo-passo

## Prerequisiti

Per il LAB10 ti servono:

* LAB08 completato
* LAB09 completato
* Resource Group del modulo già esistente
* accesso al portale Azure
* accesso a Cloud Shell o WSL con Azure CLI autenticata
* file `azure_env.md` aggiornato

---

## Step 1 - Prepara l’ambiente locale

Apri WSL Ubuntu ed esegui:

```bash
mkdir -p ~/corso_obs/NOME-REPOSITORY/labslab10 && cd ~/corso_obs/NOME-REPOSITORY/labslab10
script -a cmdlog_lab10.txt
mkdir -p docs
```

### Che cosa fanno questi comandi

#### `mkdir -p ~/corso_obs/NOME-REPOSITORY/labslab10 && cd ~/corso_obs/NOME-REPOSITORY/labslab10`

Crea la cartella del laboratorio e vi entra.

#### `script -a cmdlog_lab10.txt`

Registra la sessione terminale.

#### `mkdir -p docs`

Crea la cartella per le evidenze.

---

## Step 2 - Verifica il contesto Azure attivo

Esegui:

```bash
az account show --output table
```

### Che cosa fa questo comando

Mostra il contesto Azure attivo: subscription, tenant, utente e stato.

### Perché lo fai

Prima di creare risorse SQL devi essere sicuro di operare nella subscription corretta.

### Evidenza richiesta

Copia l’output nel file delle evidenze.

---

## Step 3 - Imposta le variabili del laboratorio

Esegui:

```bash
export RG="rg-observability-lab"
export LOCATION="westeurope"
SQL_SERVER="obssql$RANDOM"
SQL_DB="obsdb"
```

### Spiegazione

#### `RG`

Nome del Resource Group persistente del modulo.

#### `LOCATION`

Region coerente con il modulo.

#### `SQL_SERVER="obssql$RANDOM"`

Genera un nome univoco per il server SQL.
Il nome deve essere unico a livello Azure, quindi la parte casuale serve proprio a evitare collisioni.

#### `SQL_DB="obsdb"`

Nome del database, come richiesto dalla traccia.

### Verifica facoltativa

```bash
echo "$RG"
echo "$LOCATION"
echo "$SQL_SERVER"
echo "$SQL_DB"
```

---

## Step 4 - Crea Azure SQL Server

Esegui il comando seguente, sostituendo `<ADMIN_USER>` e `<PASSWORD_COMPLESSA>` con valori reali:

```bash
az sql server create \
  --name "$SQL_SERVER" \
  --resource-group "$RG" \
  --location "$LOCATION" \
  --admin-user <ADMIN_USER> \
  --admin-password <PASSWORD_COMPLESSA>
```

### Che cosa fa questo comando

* `az sql server create` crea un server logico Azure SQL
* `--name "$SQL_SERVER"` usa il nome generato
* `--resource-group "$RG"` lo inserisce nel RG del modulo
* `--location "$LOCATION"` usa la region del modulo
* `--admin-user` definisce l’utente amministrativo SQL
* `--admin-password` definisce la password amministrativa

### Attenzione importante

Usa una password conforme ai requisiti di Azure SQL.
Annota username e password in modo sicuro. Non incollarli nelle evidenze condivise.

### Evidenza richiesta

Copia l’output sintetico del comando nel file delle evidenze.

---

## Step 5 - Consenti l’accesso dal tuo IP o dai servizi Azure

Per poter interrogare il database, devi verificare le regole firewall.

Puoi aggiungere una regola aperta ai servizi Azure:

```bash
az sql server firewall-rule create \
  --resource-group "$RG" \
  --server "$SQL_SERVER" \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

### Che cosa fa questo comando

Crea una regola firewall che consente accesso da servizi Azure.

### Perché lo fai

Senza regola firewall, rischi di non poter usare Query Editor o altri strumenti.

### Nota didattica

In ambienti reali di produzione si lavora con politiche molto più restrittive. Qui stai facendo un laboratorio, non aprendo Fort Knox per hobby.

### Evidenza richiesta

Copia l’output nel file delle evidenze.

---

## Step 6 - Crea il database `obsdb`

Esegui:

```bash
az sql db create \
  --resource-group "$RG" \
  --server "$SQL_SERVER" \
  --name "$SQL_DB" \
  --service-objective Basic
```

### Che cosa fa questo comando

* `az sql db create` crea un database SQL
* `--resource-group "$RG"` specifica il gruppo di risorse
* `--server "$SQL_SERVER"` indica il server logico
* `--name "$SQL_DB"` usa il nome `obsdb`
* `--service-objective Basic` usa un tier semplice e coerente con cost control

### Evidenza richiesta

Copia l’output nel file delle evidenze.

---

## Step 7 - Verifica le risorse SQL nel portale

Nel portale Azure cerca:

```text
SQL servers
```

Apri il server appena creato.

Poi apri il database `obsdb`.

### Che cosa osservare

Nel server:

* nome
* Resource Group
* region
* admin login
* firewall rules o networking

Nel database:

* nome
* server associato
* stato
* tier

### Evidenza richiesta

Esegui uno screenshot della Overview del server SQL e uno del database `obsdb`.

---

## Step 8 - Apri Query Editor oppure usa un client SQL

Nel portale, dentro il database `obsdb`, cerca:

```text
Query editor
```

Se disponibile, autenticati con:

* utente admin SQL
* password admin SQL

Se Query Editor non è disponibile nel tuo ambiente, puoi usare un client compatibile esterno.
Per il laboratorio didattico, però, il Query Editor del portale è la strada più lineare.

---

## Step 9 - Crea la tabella `requests_log`

Esegui questa query SQL:

```sql
CREATE TABLE requests_log (
    id INT IDENTITY(1,1) PRIMARY KEY,
    correlation_id NVARCHAR(100) NOT NULL,
    path NVARCHAR(255) NOT NULL,
    status INT NOT NULL,
    duration_ms INT NOT NULL,
    created_at DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
```

### Che cosa fa questa query

Crea una tabella con:

* chiave primaria `id`
* identificatore di correlazione
* endpoint richiesto
* status HTTP
* durata in millisecondi
* timestamp automatico UTC

### Perché questa struttura è importante

È la base minima per analizzare il comportamento del servizio in modo osservabile.

### Evidenza richiesta

Esegui uno screenshot del Query Editor con la query eseguita correttamente oppure copia il testo e annota il successo dell’operazione.

---

## Step 10 - Inserisci dati di esempio osservabili

Esegui questa query:

```sql
INSERT INTO requests_log (correlation_id, path, status, duration_ms)
VALUES
('req-001', '/health', 200, 12),
('req-002', '/time',   200, 45),
('req-003', '/time',   200, 80),
('req-004', '/nope',   404, 7),
('req-005', '/time',   200, 120),
('req-006', '/health', 200, 15),
('req-007', '/time',   500, 300),
('req-008', '/health', 200, 10),
('req-009', '/nope',   404, 9),
('req-010', '/time',   200, 60);
```

### Che cosa fa questa query

Inserisce un piccolo dataset didattico nella tabella.

### Perché è utile

Se non hai ancora integrazione automatica app → SQL, questo dataset ti permette comunque di esercitare subito le query SLI richieste dalla traccia.

### Evidenza richiesta

Annota che i record sono stati inseriti correttamente.

---

## Step 11 - Visualizza tutti i record

Esegui:

```sql
SELECT * FROM requests_log;
```

### Che cosa osservare

Controlla che siano presenti righe con:

* `correlation_id`
* `path`
* `status`
* `duration_ms`
* `created_at`

### Evidenza richiesta

Copia il risultato o esegui uno screenshot.

---

## Step 12 - Calcola il numero totale di richieste

Esegui:

```sql
SELECT COUNT(*) AS total_requests
FROM requests_log;
```

### Che cosa misura

Questo è il primo SLI elementare: il volume totale delle richieste registrate.

### Evidenza richiesta

Copia il risultato nel file evidenze.

---

## Step 13 - Calcola l’error rate

Esegui:

```sql
SELECT
    COUNT(*) AS total_requests,
    SUM(CASE WHEN status >= 400 THEN 1 ELSE 0 END) AS error_requests,
    CAST(
        100.0 * SUM(CASE WHEN status >= 400 THEN 1 ELSE 0 END) / COUNT(*)
        AS DECIMAL(5,2)
    ) AS error_rate_percent
FROM requests_log;
```

### Che cosa misura

Questa query calcola:

* totale richieste
* richieste in errore
* percentuale di errore

### Perché è importante

L’error rate è uno degli SLI più classici e utili.

### Evidenza richiesta

Copia il risultato nel file evidenze.

---

## Step 14 - Trova i top endpoint

Esegui:

```sql
SELECT
    path,
    COUNT(*) AS total_calls
FROM requests_log
GROUP BY path
ORDER BY total_calls DESC;
```

### Che cosa misura

Mostra quali endpoint sono stati chiamati più spesso.

### Perché è utile

Ti aiuta a capire il profilo di traffico dell’applicazione.

### Evidenza richiesta

Copia il risultato nel file evidenze.

---

## Step 15 - Calcola la latenza media

Esegui:

```sql
SELECT
    AVG(CAST(duration_ms AS FLOAT)) AS avg_latency_ms
FROM requests_log;
```

### Che cosa misura

Restituisce la latenza media delle richieste in millisecondi.

### Perché è importante

La latenza è uno degli indicatori più importanti della qualità percepita del servizio.

### Evidenza richiesta

Copia il risultato nel file evidenze.

---

## Step 16 - Calcola la latenza media per endpoint

Esegui:

```sql
SELECT
    path,
    AVG(CAST(duration_ms AS FLOAT)) AS avg_latency_ms
FROM requests_log
GROUP BY path
ORDER BY avg_latency_ms DESC;
```

### Che cosa misura

Ti mostra quali endpoint risultano mediamente più lenti.

### Perché è utile

Permette un’analisi più fine del comportamento del servizio.

### Evidenza richiesta

Copia il risultato nel file evidenze.

---

## Step 17 - Cerca una richiesta specifica tramite `correlation_id`

Esegui:

```sql
SELECT *
FROM requests_log
WHERE correlation_id = 'req-004';
```

### Che cosa misura

Questa query ti fa vedere come usare il `correlation_id` per recuperare una singola richiesta.

### Perché è importante

È il collegamento concettuale tra:

* evento singolo
* log applicativo
* analisi della persistenza

### Evidenza richiesta

Copia il risultato nel file evidenze.

---

## Step 18 - Aggiorna il file ambiente del modulo

Apri `docs/azure_env.md` e aggiungi o aggiorna:

```markdown
SQL_SERVER=<nome effettivo del server SQL>
SQL_DB=obsdb
```

La traccia del modulo richiede che il file ambiente venga popolato progressivamente anche con i riferimenti SQL.

### Perché lo fai

Perché nei laboratori successivi ti servirà ricordare con precisione il nome del server e del database.

---

## Step 19 - Crea il file delle evidenze

Crea:

```text
docs/evidence_lab10.md
```

Usa questa struttura:

```md
# LAB10 - Evidence

## Azure SQL
- SQL_SERVER:
- SQL_DB: obsdb
- Resource Group:
- Regione:

## Query eseguite

### Creazione tabella requests_log
[incollare query o screenshot]

### Inserimento dati di esempio
[annotare esecuzione]

### SELECT * FROM requests_log
[incollare output o screenshot]

### Total requests
[incollare risultato]

### Error rate
[incollare risultato]

### Top endpoint
[incollare risultato]

### Average latency
[incollare risultato]

### Average latency by path
[incollare risultato]

### Query per correlation_id
[incollare risultato]

## Sintesi finale
- Che cosa ho capito sulla persistenza dei dati osservabili:
- Che cosa misura un SLI:
- Perché SQL è utile nell’Observability:
- Differenza tra log runtime e dati resi persistenti:
```

---

## Step 20 - Verifica la struttura locale del laboratorio

Esegui:

```bash
ls -R
```

### Che cosa devi vedere

Almeno:

```text
docs/
cmdlog_lab10.txt
```

e dentro `docs`:

```text
evidence_lab10.md
```

---

## Step 21 - Consegna nel repository Git

Esegui:

```bash
git add docs/azure_env.md docs/evidence_lab10.md
git commit -m "[LAB10] Azure SQL e query SLI completato"
git push
```

### Che cosa fanno i comandi

* `git add` aggiunge i file aggiornati
* `git commit` salva la nuova versione
* `git push` invia tutto al repository remoto

---

# PARTE 3 - Checkpoint, criteri di completamento e significato professionale

# 1. Checkpoint riassuntivi

## Checkpoint #1

Il server Azure SQL esiste ed è visibile nel portale.

## Checkpoint #2

Il database `obsdb` esiste ed è associato al server corretto.

## Checkpoint #3

La tabella `requests_log` è stata creata correttamente.

## Checkpoint #4

Sono presenti record con:

* `correlation_id`
* `path`
* `status`
* `duration_ms`

## Checkpoint #5

Sono state eseguite query SLI su:

* total requests
* error rate
* top endpoint
* latenza media

## Checkpoint #6

`azure_env.md` è stato aggiornato con:

* `SQL_SERVER`
* `SQL_DB`

---

# 2. Criteri di completamento

Il LAB10 è da considerarsi completato se:

* il server SQL è stato creato
* il database `obsdb` è stato creato
* la tabella `requests_log` esiste
* i dati di esempio sono stati inseriti
* le query SLI sono state eseguite
* il file `docs/evidence_lab10.md` è completo

Questi obiettivi sono coerenti con la traccia del modulo, che richiede proprio la creazione di SQL Server, database, tabella osservabile e query per indicatori di servizio.

---

# 3. Che cosa stai imparando davvero

Questo laboratorio ti insegna almeno cinque cose importanti.

## 3.1 I segnali osservabili possono diventare dati strutturati

Non tutto resta log grezzo. Alcuni eventi vanno resi persistenti in forma interrogabile.

## 3.2 SQL può essere uno strumento di osservabilità

Quando usi SQL per misurare errori, latenza e volumi, stai facendo analisi osservability.

## 3.3 Gli SLI si costruiscono a partire dai dati

Error rate e avg latency non sono opinioni. Sono misure.

## 3.4 Il correlation_id è fondamentale

Permette di recuperare eventi singoli e di collegare analisi aggregate e casi specifici.

## 3.5 Questo laboratorio prepara i successivi

Le logiche imparate qui ti aiuteranno moltissimo quando passerai a:

* centralizzazione log in Log Analytics
* query KQL
* alert rule
* workbook

---

# 4. Errori comuni da evitare

## Errore 1 - Pensare che SQL qui sia “fuori tema”

No. Qui SQL serve per rendere durevole e analizzare dati osservabili.

## Errore 2 - Confondere log applicativo e tabella osservabile

I log raccontano eventi grezzi.
La tabella osservabile struttura i dati per analisi ripetibili.

## Errore 3 - Non proteggere le credenziali SQL

Non inserire username e password reali nelle evidenze condivise.

## Errore 4 - Creare il server ma dimenticare firewall o accesso

Se non gestisci l’accesso, Query Editor o client SQL potrebbero non funzionare.

## Errore 5 - Fare query senza capire che cosa misurano

Il punto non è “eseguire SQL”. Il punto è sapere quale domanda osservability stai ponendo.

---

# 5. Conclusione

Questo laboratorio introduce la **persistenza strutturata dei dati osservabili** e il calcolo dei primi **SLI via SQL**.

Dopo aver completato il LAB10 devi avere chiaro che:

* i segnali osservabili possono essere salvati in una tabella
* campi come `correlation_id`, `status` e `duration_ms` sono fondamentali
* SQL permette di ricavare indicatori di servizio in modo preciso
* la persistenza rende l’analisi più stabile, riusabile e misurabile

Questo è il ponte perfetto verso il **LAB11**, in cui la logica si sposterà dalla persistenza relazionale alla **centralizzazione dei log in Log Analytics** e all’uso di **KQL** per interrogazioni osservability su scala più naturale per Azure. La traccia del modulo prevede infatti nel LAB11 il passaggio a Log Analytics Workspace e query KQL sui log centralizzati.


