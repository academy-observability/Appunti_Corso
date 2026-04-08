# LAB13 - Workbooks: Dashboard SLI e confronto SQL vs Log

## Obiettivo del laboratorio

In questo laboratorio imparerai a costruire una **dashboard osservabile** in Azure Monitor usando i **Workbooks**.

Nei laboratori precedenti hai già:

- distribuito un’app osservabile su Azure
- raccolto log applicativi dalla ACI
- centralizzato i log in **Log Analytics**
- calcolato SLI con **KQL**
- calcolato SLI anche con **SQL**
- impostato una **Alert Rule** basata su `error_rate`

Adesso devi organizzare questi dati in una vista visuale leggibile, utile per:

- comprendere rapidamente l’andamento del servizio
- confrontare più fonti dati
- costruire una base di monitoraggio più operativa

La traccia ufficiale del Modulo 2 definisce infatti il LAB13 come:

- creazione di un **Workbook**
- visualizzazione di:
  - **total requests**
  - **error rate**
  - **top endpoint**
- confronto tra **SQL (LAB10)** e **Log Analytics (LAB11)** 

---

## Durata stimata

**4 ore** 

---

## Cost Control - Gestione dei costi

La traccia ribadisce che nel percorso continuativo **non devi eliminare il Resource Group fino al LAB14**. L’eventuale cleanup con `az group delete --name rg-observability-lab --yes --no-wait` è previsto solo in modalità standalone. 

Quindi in questo laboratorio:

- non creare nuove risorse non necessarie
- riusa il workspace e il database già creati
- lavora sul workbook come oggetto logico del monitoraggio
- non distruggere l’ambiente

---

## Modalità Percorso Continuativo (Aula - Ufficiale)

**NON eliminare il Resource Group fino al LAB14.** 

---

# PARTE 1 - Concetti fondamentali, teoria e collegamento con Observability

# 1. Perché serve una dashboard

Fino a questo punto hai imparato a ottenere dati in modo molto corretto ma ancora “analitico”:

- query KQL in Log Analytics
- query SQL su `requests_log`
- log applicativi
- alert

Questi strumenti sono essenziali, ma letti uno per volta non sempre permettono una visione immediata.

Una dashboard serve a rispondere velocemente a domande come:

- il traffico sta aumentando o diminuendo?
- l’error rate è stabile o peggiora?
- quali endpoint vengono usati di più?
- i dati che vedo in SQL sono coerenti con quelli nei log?

Questa è esattamente la funzione del workbook richiesto dal LAB13. 

---

# 2. Che cos’è un Workbook in Azure Monitor

Un **Workbook** è uno strumento di Azure Monitor che permette di costruire pagine interattive composte da:

- testo
- sezioni descrittive
- query
- tabelle
- grafici
- timechart
- visualizzazioni multiple

In pratica, un workbook è una dashboard documentata e interrogabile.

Non è soltanto “un grafico carino”.  
È una vista operativa che combina:

- dati
- interpretazione
- organizzazione logica

---

# 3. Perché usare i Workbooks nel percorso Observability

Il LAB13 arriva dopo:

- raccolta dati
- persistenza dati
- interrogazione
- alerting

Quindi il workbook non è un ornamento finale.  
È il punto in cui i segnali osservabili vengono resi leggibili anche per una consultazione rapida.

In termini didattici, il workbook ti insegna che osservabilità non significa solo:

- fare query
- leggere log
- lanciare comandi

ma anche:

- costruire viste comprensibili
- mostrare indicatori rilevanti
- facilitare analisi rapide e confronto tra fonti

---

# 4. Le tre visualizzazioni richieste dal LAB13

La traccia del file è molto chiara: il workbook deve contenere tre sezioni principali basate su KQL. 

## 4.1 Total Requests
Serve a visualizzare il volume di richieste nel tempo.

Query prevista:

```kql
ContainerInstanceLog_CL
| summarize total=count() by bin(TimeGenerated, 5m)
| render timechart
````

---

## 4.2 Error Rate

Serve a visualizzare la qualità del traffico nel tempo, non solo il volume.

Query prevista:

```kql
ContainerInstanceLog_CL
| extend status = toint(parse_json(LogEntry_s).status)
| summarize total=count(), errors=countif(status >= 400) by bin(TimeGenerated, 5m)
| extend error_rate = todouble(errors)/total
| render timechart
```

---

## 4.3 Top Endpoint

Serve a capire quali endpoint sono più utilizzati.

Query prevista:

```kql
ContainerInstanceLog_CL
| extend path = tostring(parse_json(LogEntry_s).path)
| summarize hits=count() by path
| order by hits desc
```

---

# 5. Perché confrontare SQL e Log Analytics

Il file del LAB13 richiede esplicitamente un confronto tra:

* dati da **SQL** (`requests_log`)
* dati da **Log Analytics** (`ContainerInstanceLog_CL`)

Questa parte è molto importante dal punto di vista didattico, perché mostra che due fonti dati apparentemente simili possono differire per vari motivi.

---

# 6. Perché i numeri possono differire tra SQL e Log Analytics

Il file chiede esplicitamente di rispondere nel report a questa domanda:
**Perché i numeri potrebbero differire tra SQL e Log Analytics?**

Le ragioni più comuni sono:

* **finestra temporale diversa**
* **latenza di ingestione** dei log
* **eventi non persistiti** in SQL ma presenti nei log
* **dataset di simulazione** diversi
* **errori o richieste filtrate diversamente**
* **granularità differente** tra log runtime e dati persistiti

Questa differenza non è un difetto.
È una lezione importante: in observability, ogni fonte dati ha il suo punto di vista.

---

# 7. Architettura mentale del LAB13

Il laboratorio può essere letto così:

```text
ACI / applicazione
  ↓
log applicativi
  ↓
Log Analytics Workspace
  ↓
query KQL
  ↓
Workbook Azure Monitor
```

in parallelo:

```text
dati persistiti
  ↓
Azure SQL / requests_log
  ↓
query SQL
  ↓
confronto con dashboard KQL
```

L’obiettivo non è solo “fare tre grafici”, ma costruire un punto di osservazione unico e leggibile del comportamento del servizio.

---

# 8. Che cosa imparerai davvero

Alla fine del laboratorio dovrai aver capito che:

* una dashboard è uno strumento operativo, non decorativo
* i Workbooks organizzano e rendono leggibili i dati osservabili
* KQL e SQL possono raccontare la stessa storia da prospettive diverse
* osservabilità significa anche saper presentare i dati in modo utile
* confronto tra fonti dati = comprensione migliore del sistema

---

# PARTE 2 - Laboratorio guidato passo-passo

## Prerequisiti

Secondo la traccia del modulo, per il LAB13 servono:

* **LAB10 completato (SQL)**
* **LAB11 completato (Log Analytics)**
* **ACI attiva con log ingestiti**

---

## Step 1 - Setup locale in WSL Ubuntu

Esegui:

```bash
mkdir -p ~/course/lab13 && cd ~/course/lab13
script -a cmdlog_lab13.txt
mkdir -p docs
```

Questa è la sequenza riportata nel file del laboratorio.

### Che cosa fanno i comandi

#### `mkdir -p ~/course/lab13 && cd ~/course/lab13`

Crea la cartella del laboratorio ed entra nella cartella.

#### `script -a cmdlog_lab13.txt`

Registra la sessione del terminale nel file `cmdlog_lab13.txt`.

#### `mkdir -p docs`

Crea la cartella per le evidenze.

---

## Step 2 - Apri Azure Monitor e crea un Workbook

Nel portale Azure segui il percorso indicato nella traccia:

* Cerca: **Monitor**
* Menu: **Workbooks**
* **Create**

Quando richiesto, seleziona il workspace:

```text
law-observability
```

Salva il workbook con nome:

```text
wb-observability-dashboard
```

Questi dettagli sono esplicitamente richiesti dal file.

### Checkpoint #1

Il workbook deve risultare **creato e salvato**.

### Evidenza richiesta

Esegui uno screenshot del workbook salvato.

---

## Step 3 - Crea la Sezione A: Total Requests

Nel workbook seleziona:

* **Add query**

Inserisci la query KQL prevista dal file:

```kql
ContainerInstanceLog_CL
| summarize total=count() by bin(TimeGenerated, 5m)
| render timechart
```

### Che cosa fa questa query

* legge la tabella `ContainerInstanceLog_CL`
* raggruppa gli eventi per intervalli di 5 minuti
* conta il numero totale di richieste
* restituisce un **timechart**

### Che cosa osservare

Dovresti vedere un grafico temporale dell’andamento delle richieste.

### Evidenza richiesta

Esegui uno screenshot della sezione **Total Requests**.

---

## Step 4 - Crea la Sezione B: Error Rate

Nel workbook aggiungi una nuova query.

Inserisci la query prevista dalla traccia:

```kql
ContainerInstanceLog_CL
| extend status = toint(parse_json(LogEntry_s).status)
| summarize total=count(), errors=countif(status >= 400) by bin(TimeGenerated, 5m)
| extend error_rate = todouble(errors)/total
| render timechart
```

### Che cosa fa questa query

* estrae `status` dal JSON di `LogEntry_s`
* conta il totale richieste e quelle con errore
* calcola `error_rate`
* visualizza il risultato nel tempo

### Che cosa osservare

Dovresti vedere un grafico temporale dell’error rate.

### Evidenza richiesta

Esegui uno screenshot della sezione **Error Rate**.

---

## Step 5 - Crea la Sezione C: Top Endpoint

Nel workbook aggiungi una terza query.

Usa la query prevista dal file:

```kql
ContainerInstanceLog_CL
| extend path = tostring(parse_json(LogEntry_s).path)
| summarize hits=count() by path
| order by hits desc
```

### Che cosa fa questa query

* estrae `path`
* conta le richieste per endpoint
* ordina per numero di chiamate

### Che cosa osservare

Dovresti ottenere una tabella o una visualizzazione che mostri gli endpoint più usati.

### Checkpoint #2

Il workbook deve contenere **3 visualizzazioni** funzionanti. Questo checkpoint è richiesto dal file.

### Evidenza richiesta

Esegui uno screenshot della sezione **Top Endpoint**.

---

## Step 6 - Verifica coerenza delle tre visualizzazioni

Prima di passare al confronto SQL, controlla che il workbook mostri:

* trend di volume
* trend di error rate
* distribuzione endpoint

### Obiettivo didattico

Capire che queste tre viste raccontano tre aspetti diversi dello stesso sistema:

* **quanto traffico c’è**
* **quanto traffico è sano o problematico**
* **dove si concentra il traffico**

### Evidenza richiesta

Annota nel file finale una breve spiegazione di ciò che mostrano le tre sezioni.

---

## Step 7 - Apri Azure SQL per il confronto

La traccia del LAB13 richiede esplicitamente un confronto con SQL. Segui il percorso:

* Cerca: **SQL databases**
* Apri `obsdb`
* **Query editor (preview)**

### Prerequisito implicito

Devi aver completato correttamente LAB10 e avere la tabella `requests_log` popolata.

---

## Step 8 - Esegui la query SQL di confronto

Usa la query prevista nel file:

```sql
SELECT TOP 5 path, COUNT(*) AS hits
FROM requests_log
GROUP BY path
ORDER BY hits DESC;
```

### Che cosa fa questa query

* legge la tabella `requests_log`
* conta le richieste per endpoint
* restituisce i 5 endpoint più frequenti

### Che cosa osservare

Confronta i risultati con la sezione **Top Endpoint** del workbook.

### Evidenza richiesta

Copia l’output o esegui uno screenshot del Query Editor con il risultato.

---

## Step 9 - Confronta SQL e Log Analytics

Nel report devi rispondere alla domanda richiesta dal file:

> Perché i numeri potrebbero differire tra SQL e Log Analytics?

### Spunti da considerare

Puoi ragionare su aspetti come:

* i log possono includere richieste non persistite in SQL
* SQL può contenere un dataset di simulazione o storico diverso
* le finestre temporali osservate possono non coincidere
* i tempi di ingestione dei log possono introdurre differenze
* i filtri applicati nelle query possono non essere identici

### Evidenza richiesta

Scrivi un confronto ragionato di **3-6 righe**, come richiesto dal file.

---

## Step 10 - Salva il workbook in modo definitivo

Dopo aver verificato le tre sezioni, salva definitivamente il workbook.

### Che cosa devi controllare

Verifica che il nome resti:

```text
wb-observability-dashboard
```

e che il workbook risulti disponibile tra quelli salvati.

### Evidenza richiesta

Esegui uno screenshot della vista finale del workbook con le sezioni visibili.

---

## Step 11 - Crea il file delle evidenze

Il file del modulo richiede esplicitamente:

* screenshot workbook (3 sezioni)
* query SQL + output
* confronto ragionato (3-6 righe)

Crea:

```text
docs/evidence_lab13.md
```

Usa questa struttura:

````md
# LAB13 - Evidence

## 1. Workbook
- Nome workbook: wb-observability-dashboard
- Workspace usato: law-observability

## 2. Sezione A - Total Requests
- Query usata:
```kql
ContainerInstanceLog_CL
| summarize total=count() by bin(TimeGenerated, 5m)
| render timechart
````

* Screenshot allegato: SÌ/NO
* Osservazioni:

## 3. Sezione B - Error Rate

* Query usata:

```kql
ContainerInstanceLog_CL
| extend status = toint(parse_json(LogEntry_s).status)
| summarize total=count(), errors=countif(status >= 400) by bin(TimeGenerated, 5m)
| extend error_rate = todouble(errors)/total
| render timechart
```

* Screenshot allegato: SÌ/NO
* Osservazioni:

## 4. Sezione C - Top Endpoint

* Query usata:

```kql
ContainerInstanceLog_CL
| extend path = tostring(parse_json(LogEntry_s).path)
| summarize hits=count() by path
| order by hits desc
```

* Screenshot allegato: SÌ/NO
* Osservazioni:

## 5. Confronto SQL

### Query SQL

```sql
SELECT TOP 5 path, COUNT(*) AS hits
FROM requests_log
GROUP BY path
ORDER BY hits DESC;
```

### Output

[incollare output o descrivere risultato]

## 6. Confronto ragionato

Perché i numeri potrebbero differire tra SQL e Log Analytics?

[risposta in 3-6 righe]

## 7. Note finali

* Che cosa ho capito sui Workbooks:
* Che cosa ho capito sul confronto tra fonti dati:
* Quale visualizzazione considero più utile e perché:

````

---

## Step 12 - Consegna nel repository Git

La traccia del file prevede questa consegna: 

```bash
git add docs/evidence_lab13.md
git commit -m "[LAB13] Dashboard completata"
git push
````

### Che cosa fanno i comandi

* `git add` aggiunge il file delle evidenze
* `git commit` salva la versione del laboratorio
* `git push` pubblica il risultato sul repository remoto

---

# PARTE 3 - Checkpoint, criteri di completamento e significato professionale

# 1. Checkpoint riassuntivi

## Checkpoint #1

Il workbook `wb-observability-dashboard` è stato creato e salvato.

## Checkpoint #2

Nel workbook sono presenti 3 visualizzazioni funzionanti:

* Total Requests
* Error Rate
* Top Endpoint

## Checkpoint #3

È stato eseguito il confronto con SQL su `requests_log`.

---

# 2. Criteri di completamento

Secondo il file del Modulo 2, il LAB13 è completato se risultano soddisfatti questi criteri:

* Workbook creato e salvato
* 3 visualizzazioni funzionanti
* Confronto SQL vs Log documentato
* Evidence completa

---

# 3. Che cosa stai imparando davvero

Questo laboratorio ti insegna almeno cinque cose importanti.

## 3.1 Una dashboard non è un ornamento

Serve a leggere rapidamente i segnali importanti del sistema.

## 3.2 I Workbooks trasformano query sparse in una vista operativa

Mettono insieme volume, qualità e distribuzione del traffico.

## 3.3 Una singola fonte dati non basta sempre

Confrontare SQL e Log Analytics aiuta a capire il sistema in modo più maturo.

## 3.4 Divergenza tra fonti non significa errore automatico

Spesso significa che ogni fonte racconta un aspetto diverso del comportamento del sistema.

## 3.5 Presentare i dati è parte dell’Observability

Non basta saperli raccogliere. Devi anche saperli rendere leggibili.

---

# 4. Errori comuni da evitare

## Errore 1 - Creare il workbook senza verificare i dati a monte

Se i log non arrivano o le query non sono corrette, il workbook sarà solo una cornice vuota.

## Errore 2 - Confondere visualizzazione e verità assoluta

Il workbook semplifica la lettura, ma non sostituisce l’analisi quando serve dettaglio.

## Errore 3 - Aspettarsi che SQL e Log Analytics diano sempre numeri identici

La traccia richiede proprio di spiegare perché potrebbero differire. Quindi no, non sono per forza una copia speculare.

## Errore 4 - Fare screenshot senza capire cosa mostrano

L’evidence non deve essere solo estetica. Deve dimostrare comprensione.

## Errore 5 - Non dare nomi chiari al workbook

In un ambiente reale, i nomi vaghi rendono inutilizzabili i dashboard condivisi.

---

# 5. Conclusione

Questo laboratorio introduce la **lettura visuale e comparativa** dei segnali osservabili del Modulo 2.

Dopo aver completato il LAB13 devi avere chiaro che:

* i Workbooks servono a trasformare dati e query in una dashboard utile
* puoi visualizzare volume richieste, qualità del traffico e distribuzione endpoint
* il confronto tra SQL e Log Analytics è istruttivo, anche quando i numeri non coincidono perfettamente
* una dashboard ben costruita è uno strumento operativo reale


