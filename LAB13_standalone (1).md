# LAB13_standalone - Workbooks: Dashboard SLI + ripristino SQL di supporto

## Obiettivo del laboratorio

Questo laboratorio permette di eseguire **LAB13** anche se i partecipanti hanno eliminato le risorse Azure create in **LAB10** e **LAB10bis**.

L’idea è semplice:

- **LAB12_standalone** resta il prerequisito unico
- il **Log Analytics Workspace**, la **ACI** e l’alerting arrivano dal LAB12_standalone
- in questo laboratorio **ricrei da zero le risorse SQL minime** necessarie per il confronto richiesto da LAB13
- poi completi il workbook con le visualizzazioni richieste e con il confronto **SQL vs Log Analytics**

Quindi questo laboratorio:

1. **non richiede** che LAB10 o LAB10bis siano ancora presenti come risorse Azure
2. **ricostruisce** SQL Server + database `obsdb` + tabella `requests_log`
3. **inserisce** un dataset didattico di esempio
4. **consente** di completare il workbook del LAB13 come previsto dal percorso didattico

---

## Dipendenze

### Obbligatoria

- **LAB12_standalone completato**

### Non obbligatorie

- **LAB10**: non necessario come risorsa ancora esistente
- **LAB10bis**: non necessario come risorsa ancora esistente

---

## Durata stimata

**4 ore e 30 minuti**

---

## Cost Control - Gestione dei costi

In questo laboratorio ricrei solo le risorse minime che servono davvero:

- **1 Azure SQL Server**
- **1 Azure SQL Database Basic**
- **1 tabella `requests_log`**
- facoltativamente **1 tabella `trace_spans`** solo se vuoi riallinearti anche al LAB10bis

Per contenere i costi:

- usa il tier **Basic** per il database
- non creare pool elastici
- non abilitare funzionalità non richieste
- usa lo stesso Resource Group del LAB12_standalone
- a fine modulo valuta il cleanup delle risorse SQL, se non più necessarie

---

# PARTE 1 - Concetti fondamentali e logica didattica del laboratorio

# 1. Perché esiste questa versione standalone

Nel LAB13 originale il workbook confronta:

- dati provenienti da **Log Analytics**
- dati provenienti da **Azure SQL**

Il problema pratico è che alcuni partecipanti hanno cancellato:

- SQL Server
- database `obsdb`
- eventuali tabelle del LAB10 o LAB10bis

Quindi il workbook non può più essere completato nella forma prevista dal percorso.

Questa versione standalone non cambia il significato didattico del LAB13.

Fa una sola cosa sensata:

- **ricostruisce ciò che manca** per poter fare il confronto richiesto

---

# 2. Che cosa viene ripristinato

Per completare il LAB13 servono almeno questi oggetti SQL:

- **server logico Azure SQL**
- **database `obsdb`**
- **tabella `requests_log`**
- **dataset di esempio osservabile**

Questa è la parte indispensabile.

In modo facoltativo puoi anche ricreare:

- **tabella `trace_spans`**

Questa seconda tabella non è necessaria per completare il workbook del LAB13, ma può essere utile per riallineare anche il terreno didattico del LAB10bis.

---

# 3. Che cosa resta invariato rispetto al LAB13

Il cuore del laboratorio non cambia.

Dovrai comunque creare un workbook con almeno queste sezioni:

- **Total Requests**
- **Error Rate**
- **Top Endpoint**
- **Confronto SQL vs Log Analytics**

La differenza è solo che qui **ricrei prima il backend SQL necessario**.

---

# 4. Perché il confronto SQL vs Log Analytics resta didatticamente utile

Anche se il dataset SQL è ricreato, il confronto resta utile perché mostra che:

- una fonte persistita e strutturata può essere interrogata con SQL
- una fonte centralizzata di log può essere interrogata con KQL
- i numeri **non devono per forza coincidere perfettamente**
- in observability le fonti raccontano aspetti diversi dello stesso sistema

Questo è esattamente uno dei punti più importanti del LAB13.

---

# 5. Architettura mentale del laboratorio

```text
LAB12_standalone
  ↓
ACI + log applicativi + Log Analytics
  ↓
query KQL
  ↓
Workbook
```

in parallelo:

```text
LAB13_standalone
  ↓
ricreazione Azure SQL
  ↓
database obsdb
  ↓
tabella requests_log
  ↓
query SQL
  ↓
confronto con Workbook
```

---

# 6. Che cosa imparerai davvero

Alla fine del laboratorio dovrai aver capito che:

- puoi recuperare un laboratorio anche se alcune risorse precedenti sono state eliminate
- Azure SQL può essere ricreato rapidamente con Azure CLI
- una dashboard utile nasce dal confronto tra fonti diverse
- SQL e KQL sono strumenti complementari
- il workbook non è un disegno: è una vista operativa costruita su dati reali o ricostruiti in modo consapevole

---

# PARTE 2 - Laboratorio guidato passo-passo

## Prerequisiti

Per questo laboratorio ti servono:

- **LAB12_standalone completato**
- accesso a **WSL Ubuntu** con Azure CLI autenticata
- accesso al portale Azure
- ACI attiva o almeno disponibile in stato coerente con il LAB12_standalone
- LAW attivo e interrogabile

Si assume che siano già presenti, dal LAB12_standalone:

- Resource Group: `rg-observability-lab12-standalone`
- Log Analytics Workspace: `law-observability-lab12`
- ACI: `obsapp-aci`

Se nel tuo ambiente i nomi sono diversi, sostituiscili nei comandi.

---

## Step 1 - Setup locale in WSL Ubuntu

Esegui:

```bash
mkdir -p ~/course/lab13_standalone && cd ~/course/lab13_standalone
script -a cmdlog_lab13_standalone.txt
mkdir -p docs
```

### Che cosa fanno i comandi

- creano la cartella del laboratorio
- registrano la sessione terminale
- preparano la cartella per le evidenze

---

## Step 2 - Autenticazione Azure e verifica contesto

Esegui:

```bash
az login
az account show --output table
```

### Obiettivo

Verificare che:

- l’utente sia autenticato
- la subscription attiva sia quella corretta

### Evidenza richiesta

Copia nel file evidenze il nome della subscription attiva.

---

## Step 3 - Imposta le variabili del laboratorio

Esegui:

```bash
export RG="rg-observability-lab12-standalone"
export LAW_NAME="law-observability-lab12"
export ACI_NAME="obsapp-aci"
export LOCATION=$(az group show --name "$RG" --query location -o tsv)

export SQL_SERVER="obssql$(date +%m%d%H%M%S)"
export SQL_DB="obsdb"
export SQL_ADMIN="sqladminobs"

read -s -p "Inserisci una password SQL complessa: " SQL_PASSWORD
export SQL_PASSWORD
printf '\n'

echo "$RG"
echo "$LAW_NAME"
echo "$ACI_NAME"
echo "$LOCATION"
echo "$SQL_SERVER"
echo "$SQL_DB"
echo "$SQL_ADMIN"
```

### Che cosa devi capire

- il Resource Group e il LAW vengono riusati dal LAB12_standalone
- il server SQL viene creato con un nome univoco
- il database si chiamerà `obsdb`, coerentemente con il LAB10

---

## Step 4 - Registra il provider Microsoft.Sql se necessario

Esegui:

```bash
az provider show -n Microsoft.Sql --query "{namespace:namespace, state:registrationState}" -o table
az provider register -n Microsoft.Sql --wait
az provider show -n Microsoft.Sql --query "{namespace:namespace, state:registrationState}" -o table
```

### Obiettivo

Verificare che il provider SQL sia registrato nella subscription.

### Evidenza richiesta

Annota lo stato finale del provider.

---

## Step 5 - Crea Azure SQL Server

Esegui:

```bash
az sql server create \
  --name "$SQL_SERVER" \
  --resource-group "$RG" \
  --location "$LOCATION" \
  --admin-user "$SQL_ADMIN" \
  --admin-password "$SQL_PASSWORD"
```

### Che cosa fa questo comando

Crea un server logico Azure SQL dentro il Resource Group del laboratorio.

### Evidenza richiesta

Copia nel file evidenze:

- nome server SQL
- Resource Group
- location

---

## Step 6 - Crea le regole firewall minime

### 6.1 Consenti accesso dai servizi Azure

Esegui:

```bash
az sql server firewall-rule create \
  --resource-group "$RG" \
  --server "$SQL_SERVER" \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

### 6.2 Consenti il tuo IP pubblico per Query Editor

Se vuoi usare il **Query editor (preview)** del portale Azure, devi consentire il tuo IP pubblico.

Hai due possibilità:

#### Opzione A - dal portale

Nel portale Azure:

- apri il server SQL appena creato
- vai su **Networking**
- usa **Add current client IPv4 address** oppure il pulsante equivalente
- salva

#### Opzione B - via CLI, se conosci già il tuo IP pubblico

```bash
read -p "Inserisci il tuo IP pubblico IPv4: " MY_IP

az sql server firewall-rule create \
  --resource-group "$RG" \
  --server "$SQL_SERVER" \
  --name AllowMyClientIP \
  --start-ip-address "$MY_IP" \
  --end-ip-address "$MY_IP"
```

### Evidenza richiesta

Esegui:

```bash
az sql server firewall-rule list \
  --resource-group "$RG" \
  --server "$SQL_SERVER" \
  -o table
```

Copia l’output nel file evidenze.

---

## Step 7 - Crea il database `obsdb`

Esegui:

```bash
az sql db create \
  --resource-group "$RG" \
  --server "$SQL_SERVER" \
  --name "$SQL_DB" \
  --service-objective Basic
```

### Che cosa fa questo comando

Crea il database `obsdb` con tier Basic, sufficiente per il laboratorio.

### Evidenza richiesta

Copia nel file evidenze il risultato sintetico della creazione.

---

## Step 8 - Verifica server e database

Esegui:

```bash
az sql server show \
  --name "$SQL_SERVER" \
  --resource-group "$RG" \
  --query "{name:name, location:location, fqdn:fullyQualifiedDomainName}" \
  -o table

az sql db show \
  --resource-group "$RG" \
  --server "$SQL_SERVER" \
  --name "$SQL_DB" \
  --query "{name:name, status:status, sku:currentSku.name}" \
  -o table
```

### Evidenza richiesta

Copia i due output nel file evidenze.

---

## Step 9 - Apri Query Editor nel portale Azure

Nel portale Azure segui questo percorso:

- **SQL databases**
- apri `obsdb`
- **Query editor (preview)**

Autenticati con:

- username: `SQL_ADMIN`
- password: `SQL_PASSWORD`

### Se non riesci a entrare

Controlla in ordine:

1. il database giusto è `obsdb`
2. il firewall ha la regola per il tuo IP
3. hai atteso almeno qualche minuto dopo la modifica del firewall
4. stai usando le credenziali SQL corrette

---

## Step 10 - Crea la tabella `requests_log`

Nel Query Editor esegui:

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

Ricrea la tabella del LAB10 utile per:

- total requests
- error rate
- top endpoint
- latenza media
- confronto SQL vs Log Analytics

### Evidenza richiesta

Annota che la tabella è stata creata correttamente.

---

## Step 11 - Inserisci il dataset di esempio del LAB10

Esegui:

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

### Obiettivo

Ricostruire il dataset minimo necessario per il confronto SQL del LAB13.

### Evidenza richiesta

Annota che i record sono stati inseriti correttamente.

---

## Step 12 - Verifica il contenuto della tabella `requests_log`

Esegui:

```sql
SELECT *
FROM requests_log;
```

### Che cosa osservare

Verifica che siano presenti colonne e dati coerenti con il modello del LAB10.

### Evidenza richiesta

Esegui uno screenshot o incolla un estratto dell’output.

---

## Step 13 - Query SQL di supporto per il confronto del LAB13

### 13.1 Total Requests

```sql
SELECT COUNT(*) AS total_requests
FROM requests_log;
```

### 13.2 Error Rate

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

### 13.3 Top Endpoint

```sql
SELECT TOP 5
    path,
    COUNT(*) AS hits
FROM requests_log
GROUP BY path
ORDER BY hits DESC;
```

### 13.4 Average Latency

```sql
SELECT
    AVG(CAST(duration_ms AS FLOAT)) AS avg_latency_ms
FROM requests_log;
```

### Evidenza richiesta

Copia i risultati nel file evidenze.

---

## Step 14 - Genera un po’ di traffico verso l’ACI

Per rendere il workbook più leggibile, genera nuovo traffico verso l’app della ACI del LAB12_standalone.

Da WSL esegui:

```bash
ACI_PUBLIC_IP=$(az container show \
  --resource-group "$RG" \
  --name "$ACI_NAME" \
  --query ipAddress.ip -o tsv)

echo "$ACI_PUBLIC_IP"
```

Poi genera traffico misto:

```bash
for i in {1..15}; do
  curl -s "http://$ACI_PUBLIC_IP:8000/health" > /dev/null
  curl -s "http://$ACI_PUBLIC_IP:8000/time" > /dev/null
  curl -s "http://$ACI_PUBLIC_IP:8000/nope" > /dev/null
  curl -s -X POST "http://$ACI_PUBLIC_IP:8000/echo" \
    -H "Content-Type: application/json" \
    -d '{"source":"lab13-standalone"}' > /dev/null
  sleep 1
done
```

### Obiettivo

Produrre dati recenti in Log Analytics per il workbook.

### Nota importante

Attendi alcuni minuti per ingestione e aggiornamento delle query KQL.

---

## Step 15 - Apri Azure Monitor e crea il Workbook

Nel portale Azure segui il percorso:

- **Monitor**
- **Workbooks**
- **Create**

Salva il workbook con nome:

```text
wb-observability-dashboard
```

### Checkpoint #1

Il workbook deve risultare creato e salvato.

### Evidenza richiesta

Esegui uno screenshot del workbook salvato.

---

## Step 16 - Sezione A: Total Requests da Log Analytics

Aggiungi una query al workbook e usa:

```kql
ContainerInstanceLog_CL
| where ContainerGroup == "obsapp-aci"
| summarize total=count() by bin(TimeGenerated, 5m)
| render timechart
```

### Che cosa misura

Mostra il volume totale dei log applicativi nel tempo.

### Evidenza richiesta

Esegui uno screenshot della sezione **Total Requests**.

---

## Step 17 - Sezione B: Error Rate da Log Analytics

Aggiungi una seconda query e usa:

```kql
ContainerInstanceLog_CL
| where ContainerGroup == "obsapp-aci"
| extend payload = parse_json(Message)
| extend status = toint(payload.status)
| where isnotnull(status)
| summarize total=count(), errors=countif(status >= 400) by bin(TimeGenerated, 5m)
| extend error_rate = todouble(errors) / todouble(total)
| render timechart
```

### Che cosa misura

Calcola l’error rate dai log centralizzati nel LAW.

### Evidenza richiesta

Esegui uno screenshot della sezione **Error Rate**.

---

## Step 18 - Sezione C: Top Endpoint da Log Analytics

Aggiungi una terza query e usa:

```kql
ContainerInstanceLog_CL
| where ContainerGroup == "obsapp-aci"
| extend payload = parse_json(Message)
| extend path = tostring(payload.path)
| where isnotempty(path)
| summarize hits=count() by path
| order by hits desc
```

### Che cosa misura

Mostra gli endpoint più chiamati secondo i log centralizzati.

### Checkpoint #2

Il workbook deve contenere tre visualizzazioni funzionanti.

### Evidenza richiesta

Esegui uno screenshot della sezione **Top Endpoint**.

---

## Step 19 - Confronto SQL vs Log Analytics

Adesso esegui il confronto richiesto dal LAB13.

### 19.1 Confronto consigliato

Confronta almeno:

- **Top Endpoint** da SQL
- **Top Endpoint** da KQL nel workbook

Facoltativamente confronta anche:

- total requests
- error rate
- latenza media SQL vs andamento error rate/log volume nel workbook

### 19.2 Domanda da discutere nel report

> Perché i numeri potrebbero differire tra SQL e Log Analytics, e quando questa differenza è normale oppure sospetta?

### Guida al ragionamento

Nel report non limitarti a scrivere che “i dataset sono diversi”.
Devi spiegare **che cosa sta misurando ciascuna fonte**, **in quale finestra temporale**, e **se la differenza osservata è plausibile** oppure merita indagine.

Ragiona almeno su questi punti:

#### A. Stessa realtà osservata, ma non stesso punto di osservazione

- **SQL** contiene un dataset ricreato a scopo didattico e persistito nella tabella `requests_log`
- **Log Analytics** contiene i log runtime realmente inviati dalla ACI dopo il deploy e durante i test
- quindi le due fonti non rappresentano necessariamente **lo stesso insieme di richieste**, anche se parlano dello stesso servizio

#### B. Differenza tra evento persistito e log runtime

- in **SQL** una riga rappresenta un evento già selezionato e salvato in forma strutturata
- in **Log Analytics** una riga rappresenta un log prodotto a runtime dal container e poi ingerito nel workspace
- questo significa che una differenza numerica può dipendere non solo dal volume, ma anche dal **tipo di evidenza raccolta**

#### C. Finestra temporale e recency dei dati

- il dataset SQL può essere fermo al momento della ricreazione del database
- Log Analytics continua invece a ricevere richieste nuove, ad esempio i test eseguiti per verificare workbook e alert
- per confrontare correttamente i numeri devi chiederti: **sto guardando lo stesso intervallo temporale oppure no?**

#### D. Filtri e perimetro del confronto

- confrontare “top endpoint” ha senso solo se il perimetro è comparabile
- per esempio: SQL potrebbe contenere solo 10 righe di esempio, mentre Log Analytics include anche `/health`, `/time`, `/echo`, `/nope` generati in momenti diversi
- quindi il confronto è corretto solo se specifichi **quale sottoinsieme** stai confrontando

#### E. Quando la differenza è normale

Una differenza è normalmente accettabile se:

- Log Analytics contiene traffico più recente del dataset SQL
- i due insiemi di dati sono stati prodotti in momenti diversi
- i log includono richieste di test aggiuntive non replicate in SQL
- il confronto è concettuale e non pretende identità perfetta dei valori

#### F. Quando la differenza diventa sospetta

La differenza va invece investigata se:

- il periodo osservato dovrebbe essere lo stesso ma i numeri divergono molto
- in Log Analytics mancano endpoint che sai di aver generato
- l'`error_rate` nei log è incompatibile con gli status persistiti in SQL
- sospetti problemi di ingestione, query errata, filtro sbagliato o dataset SQL inserito in modo incompleto

### Evidenza richiesta

Scrivi un confronto ragionato di **6-12 righe** che includa:

- che cosa misura SQL
- che cosa misura Log Analytics
- se il confronto che hai fatto è **quantitativo** o solo **concettuale**
- una frase finale del tipo:
  - **"la differenza osservata è fisiologica"**
  oppure
  - **"la differenza osservata richiede verifica"**

---

## Step 20 - Salva il workbook in modo definitivo

Controlla che il workbook:

- sia salvato
- abbia nome `wb-observability-dashboard`
- contenga le tre sezioni richieste

### Evidenza richiesta

Esegui uno screenshot finale dell’intero workbook.

---

## Step 21 - Appendice facoltativa: riallineamento con LAB10bis

Questa parte è **facoltativa**. Serve solo se vuoi ripristinare anche l’estensione concettuale del LAB10bis.

### 21.1 Crea la tabella `trace_spans`

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

### 21.2 Inserisci il dataset di esempio

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

### 21.3 Query facoltativa di verifica

```sql
SELECT *
FROM trace_spans
WHERE trace_id = 'trace-004'
ORDER BY id;
```

Questa appendice non è necessaria per il completamento del workbook, ma riallinea il contesto con LAB10bis.

---

## Step 22 - Crea il file delle evidenze

Crea:

```text
docs/evidence_lab13_standalone.md
```

Usa questa struttura:

````md
# LAB13_standalone - Evidence

## 1. Contesto
- Resource Group:
- LAW:
- ACI:
- SQL Server:
- SQL DB:

## 2. Ripristino SQL
- SQL Server creato: SÌ/NO
- Database obsdb creato: SÌ/NO
- Firewall configurato: SÌ/NO
- Tabella requests_log creata: SÌ/NO
- Dataset di esempio inserito: SÌ/NO

## 3. Query SQL

### Total requests
[incollare risultato]

### Error rate
[incollare risultato]

### Top endpoint
[incollare risultato]

### Average latency
[incollare risultato]

## 4. Workbook
- Nome workbook: wb-observability-dashboard
- Sezione A completata: SÌ/NO
- Sezione B completata: SÌ/NO
- Sezione C completata: SÌ/NO

## 5. Query KQL usate

### Total Requests
```kql
ContainerInstanceLog_CL
| where ContainerGroup == "obsapp-aci"
| summarize total=count() by bin(TimeGenerated, 5m)
| render timechart
```

### Error Rate
```kql
ContainerInstanceLog_CL
| where ContainerGroup == "obsapp-aci"
| extend payload = parse_json(Message)
| extend status = toint(payload.status)
| where isnotnull(status)
| summarize total=count(), errors=countif(status >= 400) by bin(TimeGenerated, 5m)
| extend error_rate = todouble(errors) / todouble(total)
| render timechart
```

### Top Endpoint
```kql
ContainerInstanceLog_CL
| where ContainerGroup == "obsapp-aci"
| extend payload = parse_json(Message)
| extend path = tostring(payload.path)
| where isnotempty(path)
| summarize hits=count() by path
| order by hits desc
```

## 6. Confronto SQL vs Log Analytics
- Differenze osservate:
- Possibili motivazioni:
- Quale fonte considero più adatta a questo tipo di domanda e perché:

## 7. Appendice LAB10bis (facoltativa)
- Tabella trace_spans creata: SÌ/NO
- Dataset trace inserito: SÌ/NO
- Query su trace-004 eseguita: SÌ/NO

## 8. Note finali
- Che cosa ho capito sui Workbooks:
- Che cosa ho capito sul confronto tra SQL e Log Analytics:
- Che cosa ho capito sul ripristino delle risorse mancanti:
````

---

## Step 23 - Consegna nel repository Git

Esegui:

```bash
git add docs/evidence_lab13_standalone.md
git commit -m "[LAB13_standalone] Workbook + ripristino SQL completato"
git push
```

---

# PARTE 3 - Checkpoint, criteri di completamento e significato professionale

# 1. Checkpoint riassuntivi

## Checkpoint #1

Il server SQL e il database `obsdb` sono stati ricreati con successo.

## Checkpoint #2

La tabella `requests_log` è stata ricreata e popolata.

## Checkpoint #3

Il workbook `wb-observability-dashboard` è stato creato e salvato.

## Checkpoint #4

Nel workbook sono presenti tre visualizzazioni funzionanti:

- Total Requests
- Error Rate
- Top Endpoint

## Checkpoint #5

È stato eseguito il confronto tra:

- query SQL su `requests_log`
- query KQL su `ContainerInstanceLog_CL`

---

# 2. Criteri di completamento

Il laboratorio è completato se:

- LAB12_standalone è disponibile come base tecnica
- SQL Server e `obsdb` sono stati ricreati
- `requests_log` è stata ricreata
- il dataset SQL di esempio è presente
- il workbook è stato creato e salvato
- il confronto SQL vs Log Analytics è stato documentato
- `docs/evidence_lab13_standalone.md` è completo

---

# 3. Che cosa stai imparando davvero

Questo laboratorio ti insegna almeno sei cose importanti.

## 3.1 Un ambiente lab può essere ripristinato in modo controllato

La cancellazione di una risorsa non obbliga a buttare via il percorso didattico.

## 3.2 Azure CLI è utile anche per recuperare il contesto perso

Non serve solo per creare da zero. Serve anche per ricostruire ciò che manca.

## 3.3 SQL e KQL restano complementari

Uno lavora bene sulla persistenza strutturata, l’altro sulla centralizzazione dei log.

## 3.4 Un workbook è utile solo se i dati dietro hanno senso

Per questo qui non ti limiti ai grafici: ricostruisci prima le fonti necessarie.

## 3.5 I numeri diversi non significano automaticamente errore

Spesso significano dataset diversi, finestre temporali diverse o pipeline diverse.

## 3.6 Il percorso didattico resta coerente

Questa versione standalone non sostituisce il LAB10 o il LAB10bis come concetti.
Li ripristina quanto basta per poter completare il LAB13 in modo sensato.

---

# 4. Errori comuni da evitare

## Errore 1 - Dimenticare che il workbook usa due fonti diverse

Se ricrei SQL ma non controlli LAW, il confronto resta incompleto.

## Errore 2 - Usare ancora `LogEntry_s` nelle query KQL

Nel contesto del LAB12_standalone la query va allineata al campo `Message`.

## Errore 3 - Non configurare il firewall SQL

Senza regole firewall adeguate, Query Editor potrebbe non funzionare.

## Errore 4 - Confondere dataset statico SQL e dataset runtime del LAW

Qui è normale che i numeri non coincidano perfettamente.

## Errore 5 - Fare il workbook senza generare traffico recente

Se non arrivano dati nuovi, i grafici possono risultare poveri o fuorvianti.

## Errore 6 - Ricreare anche `trace_spans` pensando che sia obbligatoria

Per il workbook del LAB13 non è obbligatoria. È solo una estensione facoltativa.

---

# 5. Conclusione

Questa versione di **LAB13_standalone** ti permette di completare il laboratorio anche se le risorse SQL dei LAB precedenti sono state eliminate.

Dopo aver completato questo laboratorio devi avere chiaro che:

- puoi ripristinare in modo controllato le risorse minime mancanti
- puoi ricostruire `obsdb` e `requests_log` con Azure CLI e Query Editor
- puoi completare il workbook del LAB13 senza perdere il senso didattico del confronto SQL vs Log Analytics
- il confronto tra fonti è utile proprio perché mostra differenze, non solo somiglianze
