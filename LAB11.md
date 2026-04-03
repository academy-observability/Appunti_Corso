# LAB11 - Centralizzazione dei log in Log Analytics e query KQL

## Obiettivo del laboratorio

In questo laboratorio porterai l’Observability del Modulo 2 a un livello superiore.

Nel LAB09 hai visto i log direttamente dalla Container Instance.  
Nel LAB10 hai visto come rendere persistenti dati osservabili in Azure SQL.

Nel LAB11 farai un passaggio molto importante: i log non verranno più letti solo “attaccandoti” alla singola risorsa, ma verranno **centralizzati in un Log Analytics Workspace** e interrogati con **KQL**, il linguaggio di query di Azure Monitor Logs.

Questo laboratorio serve a capire una differenza fondamentale:

- leggere i log direttamente dalla singola risorsa è utile
- **centralizzare i log** in un workspace comune è molto più potente

Al termine del laboratorio sarai in grado di:

- creare un **Log Analytics Workspace**
- collegare la Container Instance ai meccanismi di raccolta log Azure
- verificare che i log arrivino nel workspace
- eseguire query KQL di base
- filtrare, cercare, ordinare e aggregare log
- usare KQL per ragionare in chiave Observability

---

## Durata indicativa

**4 ore**

---

## Cost control - gestione dei costi

Anche in questo laboratorio valgono le regole di cost control del modulo:

- usare SKU minimi o opzioni standard
- non creare risorse non necessarie
- evitare workspace o configurazioni duplicate
- non lasciare risorse inutilizzate senza motivo

Nel percorso continuativo **non eliminare il Resource Group** fino al LAB14.

---

# PARTE 1 - Concetti fondamentali, teoria e collegamento con Observability

# 1. Perché centralizzare i log

Nel LAB09 hai recuperato i log della ACI con un comando come:

```bash
az container logs --resource-group "rg-observability-lab" --name "obsapp-aci"
````

Questo approccio è utile, ma ha limiti molto evidenti:

* sei legato alla singola risorsa
* non hai una vista centralizzata
* fare analisi ripetibili è scomodo
* confrontare eventi nel tempo è più difficile
* creare dashboard e alert diventa molto più naturale solo dopo la centralizzazione

Per questo Azure mette a disposizione **Log Analytics Workspace**.

---

# 2. Che cos’è un Log Analytics Workspace

Un **Log Analytics Workspace** è un contenitore centralizzato per dati di log e telemetria raccolti da Azure Monitor.

In modo semplice, puoi immaginarlo come il punto in cui convergono dati provenienti da una o più risorse Azure.

Il workspace serve a:

* raccogliere log da più sorgenti
* conservare dati osservabili
* interrogare i dati con KQL
* costruire alert
* costruire dashboard e workbook
* fare analisi operative più serie

Nel Modulo 2 il workspace sarà il punto di ingresso per l’osservabilità centralizzata.

---

# 3. Che cos’è KQL

**KQL** significa **Kusto Query Language**.

È il linguaggio usato da Azure per interrogare dati in Log Analytics, Azure Monitor Logs, Microsoft Sentinel e altri servizi della stessa famiglia.

KQL non è SQL.

## 3.1 SQL vs KQL

### SQL

Lavora soprattutto su:

* tabelle relazionali
* join relazionali
* dati strutturati classici
* CRUD e query tabellari

### KQL

Lavora molto bene su:

* log
* eventi
* telemetria
* analisi temporali
* filtri rapidi
* aggregazioni
* ricerca e troubleshooting

In pratica:

* **SQL** è ottimo per dati relazionali persistenti
* **KQL** è ottimo per log ed eventi osservabili

---

# 4. Perché KQL è importante per Observability

KQL ti permette di fare domande operative ai log centralizzati, per esempio:

* quali richieste hanno prodotto errori?
* quali endpoint compaiono più spesso?
* quali righe contengono `request_id`?
* in quale intervallo temporale si sono verificati più eventi?
* qual è la distribuzione dei messaggi?

Quindi KQL è uno strumento perfetto per:

* troubleshooting
* analisi log
* costruzione di viste operative
* base per alerting e dashboard

---

# 5. Differenza tra log grezzi e log centralizzati

## 5.1 Log grezzi sulla singola risorsa

Quando leggi i log direttamente dalla ACI:

* sei legato alla risorsa
* hai una vista locale
* fai un’analisi immediata ma limitata

## 5.2 Log centralizzati nel workspace

Quando i log arrivano nel workspace:

* hai una vista centralizzata
* puoi usare query evolute
* puoi confrontare dati nel tempo
* puoi costruire alert e dashboard
* puoi lavorare in modo più professionale

Questo laboratorio introduce proprio questa transizione.

---

# 6. Pipeline mentale del LAB11

Il laboratorio va immaginato così:

```text
ACI / applicazione
  ↓
diagnostic settings / raccolta log
  ↓
Log Analytics Workspace
  ↓
tabelle log
  ↓
KQL
  ↓
analisi osservability
```

Questa è la vera logica del laboratorio:

* la risorsa continua a eseguire il servizio
* i log non restano solo “attaccati” alla risorsa
* vengono centralizzati
* li interroghi con un linguaggio adatto ai log

---

# 7. Che tipo di dati cercherai

Nel LAB11 cercherai soprattutto log legati alla tua ACI e al servizio osservabile.

A seconda della configurazione del workspace e delle diagnostic settings, i dati potrebbero finire in:

* `AzureDiagnostics`
* tabelle dedicate del tipo `ContainerInstanceLog_CL`
* tabelle eventi della Container Instance
* altre tabelle generate dal tipo di raccolta abilitata

Per questo nel laboratorio userai un approccio pratico:

1. prima individui dove stanno arrivando i dati
2. poi scrivi query di esplorazione
3. poi scrivi query mirate

Questo è un ottimo approccio reale. Nessuno serio parte da una query complessa senza prima capire dove sono i dati.

---

# 8. Schema mentale di KQL

KQL legge spesso come una pipeline.

Esempio concettuale:

```kusto
Tabella
| where condizione
| project colonne
| sort by timestamp desc
```

Questo significa:

1. prendo una tabella
2. filtro i record
3. scelgo le colonne da vedere
4. ordino il risultato

È molto leggibile e molto utile per l’analisi log.

---

# 9. Operatori KQL che userai

Nel laboratorio userai soprattutto questi operatori.

## `where`

Filtra i record.

## `project`

Sceglie quali colonne visualizzare.

## `sort by`

Ordina i risultati.

## `take`

Limita il numero di righe.

## `summarize`

Aggrega i dati.

## `count`

Conta le righe.

## `contains`

Cerca testo dentro una colonna stringa.

## `bin()`

Raggruppa valori temporali in finestre, per esempio 5 minuti.

Questi operatori bastano già per fare analisi molto utili.

---

# 10. Che cosa imparerai davvero

Alla fine del laboratorio dovrai aver capito che:

* i log osservabili diventano molto più utili quando sono centralizzati
* KQL è un linguaggio potente e leggibile per analizzare log
* non basta “vedere log”, bisogna saperli interrogare
* questo laboratorio prepara direttamente:

  * **LAB12**, dove userai query/log come base per alerting
  * **LAB13**, dove userai i dati per workbook e dashboard

---

# PARTE 2 - Laboratorio guidato passo-passo

## Prerequisiti

Per il LAB11 ti servono:

* LAB08 completato
* LAB09 completato
* LAB09bis completato
* Resource Group del modulo già esistente
* ACI già creata e funzionante
* endpoint applicativi già testabili
* file `azure_env.md` aggiornato con almeno:

  * `RG`
  * `LOCATION`
  * `ACI_NAME`
  * `ACI_PUBLIC_IP`

---

## Step 1 - Prepara l’ambiente locale

Apri WSL Ubuntu ed esegui:

```bash
mkdir -p ~/course/lab11 && cd ~/course/lab11
script -a cmdlog_lab11.txt
mkdir -p docs
```

### Che cosa fanno questi comandi

#### `mkdir -p ~/course/lab11 && cd ~/course/lab11`

Crea la cartella del laboratorio e vi entra.

#### `script -a cmdlog_lab11.txt`

Registra la sessione del terminale.

#### `mkdir -p docs`

Crea la cartella per le evidenze.

---

## Step 2 - Verifica il contesto Azure attivo

Esegui:

```bash
az account show --output table
```

### Che cosa fa

Mostra subscription, tenant e utente attivi.

### Perché lo fai

Prima di creare il workspace e configurare raccolta log, devi essere sicuro di lavorare nella subscription corretta.

### Evidenza richiesta

Copia l’output nel file evidenze.

---

## Step 3 - Imposta le variabili del laboratorio

Esegui:

```bash
export RG="rg-observability-lab"
export LOCATION="westeurope"
export ACI_NAME="obsapp-aci"
LAW_NAME="law-observability-$RANDOM"
```

### Spiegazione

#### `RG`

Nome del Resource Group del modulo.

#### `LOCATION`

Region coerente con il percorso.

#### `ACI_NAME`

Nome della Container Instance già esistente.

#### `LAW_NAME`

Nome del Log Analytics Workspace.
Uso una parte casuale per evitare collisioni di naming.

### Verifica facoltativa

```bash
echo "$RG"
echo "$LOCATION"
echo "$ACI_NAME"
echo "$LAW_NAME"
```

---

## Step 4 - Crea il Log Analytics Workspace

Esegui:

```bash
az monitor log-analytics workspace create \
  --resource-group "$RG" \
  --workspace-name "$LAW_NAME" \
  --location "$LOCATION"
```

### Che cosa fa questo comando

* `az monitor log-analytics workspace create` crea il workspace
* `--resource-group "$RG"` inserisce la risorsa nel RG del modulo
* `--workspace-name "$LAW_NAME"` assegna il nome del workspace
* `--location "$LOCATION"` usa la region del modulo

### Evidenza richiesta

Copia l’output sintetico del comando nel file evidenze.

---

## Step 5 - Verifica il workspace nel portale

Nel portale Azure cerca:

```text
Log Analytics workspaces
```

Apri il workspace appena creato.

### Che cosa osservare

Controlla:

* nome del workspace
* Resource Group
* region
* subscription
* menu laterale
* presenza della sezione **Logs**

### Evidenza richiesta

Esegui uno screenshot della Overview del workspace.

---

## Step 6 - Recupera l’ID della ACI

Esegui:

```bash
ACI_ID=$(az container show --resource-group "$RG" --name "$ACI_NAME" --query id -o tsv)
echo "$ACI_ID"
```

### Che cosa fa

Recupera l’ID completo della risorsa ACI.

### Perché serve

Ti servirà per configurare le diagnostic settings sulla risorsa corretta.

### Evidenza richiesta

Annota che l’ID della risorsa è stato recuperato.

---

## Step 7 - Recupera l’ID del workspace

Esegui:

```bash
LAW_ID=$(az monitor log-analytics workspace show \
  --resource-group "$RG" \
  --workspace-name "$LAW_NAME" \
  --query id -o tsv)

echo "$LAW_ID"
```

### Che cosa fa

Recupera l’ID completo del workspace.

### Perché serve

Le diagnostic settings devono sapere verso quale workspace inviare i log.

---

## Step 8 - Configura le diagnostic settings sulla ACI

Esegui:

```bash
az monitor diagnostic-settings create \
  --name "diag-aci-to-law" \
  --resource "$ACI_ID" \
  --workspace "$LAW_ID" \
  --logs '[
    {"category":"ContainerInstanceLog","enabled":true},
    {"category":"ContainerEvent","enabled":true}
  ]'
```

### Che cosa fa questo comando

Crea una diagnostic setting sulla Container Instance e invia al workspace almeno:

* log del container
* eventi del container

### Nota importante

A seconda della sottoscrizione, della regione o della versione dei servizi, i nomi esatti delle categorie possono variare. Se il comando restituisce errore sulle categorie, prima elenca le categorie disponibili con:

```bash
az monitor diagnostic-settings categories list --resource "$ACI_ID"
```

e poi usa i nomi mostrati realmente nel tuo ambiente.

### Evidenza richiesta

Copia l’output nel file evidenze.

---

## Step 9 - Verifica le categorie disponibili, se necessario

Se il comando precedente ha avuto problemi oppure vuoi controllare le categorie supportate, esegui:

```bash
az monitor diagnostic-settings categories list --resource "$ACI_ID"
```

### Che cosa fa

Mostra le categorie diagnostiche disponibili per la risorsa ACI.

### Perché è utile

Ti aiuta a non lavorare “a memoria”, ma sulla realtà del tuo ambiente.

### Evidenza richiesta

Copia l’output oppure annota le categorie trovate.

---

## Step 10 - Genera nuove richieste verso la ACI

Per popolare i log, esegui più richieste verso gli endpoint già noti:

```bash
curl -i "http://<ACI_PUBLIC_IP>:8000/health"
curl -i "http://<ACI_PUBLIC_IP>:8000/time"
curl -i "http://<ACI_PUBLIC_IP>:8000/time"
curl -i "http://<ACI_PUBLIC_IP>:8000/nope"
curl -i "http://<ACI_PUBLIC_IP>:8000/health"
```

Sostituisci `<ACI_PUBLIC_IP>` con il valore reale del tuo ambiente.

### Perché lo fai

Perché vuoi produrre nuovi eventi osservabili che possano essere raccolti dal workspace.

### Evidenza richiesta

Copia almeno una parte degli output nel file evidenze.

---

## Step 11 - Attendi qualche minuto per l’ingestione

Aspetta alcuni minuti.

### Perché lo fai

I log non sempre compaiono immediatamente nel workspace.
Tra generazione, raccolta, invio e indicizzazione può servire un po’ di tempo.

### Nota didattica

Questo è un aspetto reale dell’Observability: non tutto è sempre istantaneo.

---

## Step 12 - Apri la sezione Logs del workspace

Nel portale Azure, dentro il tuo Log Analytics Workspace, apri:

```text
Logs
```

### Che cosa osservare

Vedrai l’editor KQL, la lista delle tabelle e l’area risultati.

Questa è la tua console di analisi centralizzata dei log.

### Evidenza richiesta

Esegui uno screenshot della schermata Logs del workspace.

---

**ATTENZIONE**
Prima di andare avanti manda di nuovo i comandi dei seguenti Step:

Step 6 — Recupera l'ID della ACI
Step 7 — Recupera l'ID del Workspace
Step 8 — Collega ACI al Workspace
Step 10 — Genera traffico verso il container

## Step 13 - Identifica le tabelle che contengono i dati

Nel pannello delle tabelle oppure tramite query, prova a identificare dove stanno arrivando i dati.

Puoi eseguire una query iniziale molto semplice:

```kusto
search *
| take 50
```

### Che cosa fa questa query

* `search *` cerca dati disponibili
* `| take 50` limita i risultati alle prime 50 righe

### Che cosa devi osservare

Controlla:

* nomi delle tabelle
* colonne disponibili
* eventuale presenza di riferimenti alla ACI o al container

### Nota importante

A seconda della configurazione, i dati potrebbero apparire in tabelle diverse.

### Evidenza richiesta

Esegui uno screenshot dei primi risultati o annota i nomi delle tabelle osservate.

---

## Step 14 - Esplora la tabella `AzureDiagnostics`, se presente

Se i log arrivano nella tabella `AzureDiagnostics`, esegui:

```kusto
AzureDiagnostics
| take 50
```

### Che cosa osservare

Cerca colonne come:

* `TimeGenerated`
* `Resource`
* `ResourceProvider`
* `Category`
* `Message`
* campi equivalenti ai log del container

### Perché è utile

`AzureDiagnostics` è spesso la tabella classica dove finiscono i dati diagnostici Azure.

### Evidenza richiesta

Copia o fotografa il risultato.

---

## Step 15 - Filtra i log della tua ACI

Se stai usando `AzureDiagnostics`, esegui una query di questo tipo:

```kusto
AzureDiagnostics
| where Resource contains "obsapp-aci"
| sort by TimeGenerated desc
| take 50
```

Se hai usato un nome ACI diverso, sostituiscilo.

### Che cosa fa questa query

* filtra i record riferiti alla tua Container Instance
* ordina dal più recente al meno recente
* limita i risultati alle ultime 50 righe

### Cosa devi capire

Stai già facendo una query osservability utile: stai isolando la sorgente di log di interesse.

### Evidenza richiesta

Copia o fotografa il risultato.

---

## Step 16 - Filtra i log di sola categoria container log

Se nella tabella vedi una colonna `Category`, prova:

```kusto
AzureDiagnostics
| where Resource contains "obsapp-aci"
| where Category contains "Container"
| sort by TimeGenerated desc
| take 50
```

### Che cosa misura

Isola ancora meglio il tipo di log che ti interessa.

### Evidenza richiesta

Salva il risultato nel file evidenze.

---

## Step 17 - Cerca righe che contengono `request` o `status`

Se nella tabella esiste una colonna messaggio, per esempio `Message`, prova:

```kusto
AzureDiagnostics
| where Resource contains "obsapp-aci"
| where Message contains "request"
   or Message contains "status"
| project TimeGenerated, Resource, Category, Message
| sort by TimeGenerated desc
```

### Che cosa fa

Ti aiuta a cercare righe applicative più interessanti.

### Nota importante

Il nome della colonna messaggio può cambiare. Se non è `Message`, usa il nome reale visto nel tuo ambiente.

### Evidenza richiesta

Copia un estratto utile.

---

## Step 18 - Conta quante righe log hai ricevuto

Esegui:

```kusto
AzureDiagnostics
| where Resource contains "obsapp-aci"
| summarize total_logs = count()
```

### Che cosa misura

Conta quante righe di log sono arrivate per la ACI.

### Perché è utile

È la forma più semplice di aggregazione osservability su log centralizzati.

### Evidenza richiesta

Copia il risultato.

---

## Step 19 - Raggruppa i log nel tempo

Esegui:

```kusto
AzureDiagnostics
| where Resource contains "obsapp-aci"
| summarize total_logs = count() by bin(TimeGenerated, 5m)
| sort by TimeGenerated asc
```

### Che cosa fa

Raggruppa il numero di log per finestre di 5 minuti.

### Perché è utile

Ti introduce all’analisi temporale, fondamentale in Observability.

### Evidenza richiesta

Copia il risultato o fotografa il grafico se il portale lo propone.

---

## Step 20 - Se i dati non sono in `AzureDiagnostics`, esplora la tabella reale

Se i dati sono in una tabella diversa, adatta la query.

Per esempio, se trovi una tabella dedicata simile a `ContainerInstanceLog_CL`, puoi usare:

```kusto
ContainerInstanceLog_CL
| take 50
```

Poi:

```kusto
ContainerInstanceLog_CL
| sort by TimeGenerated desc
| take 50
```

e infine query mirate su colonne reali trovate nella tabella.

### Che cosa devi capire

Nel lavoro reale, prima si osserva lo schema dei dati e poi si scrivono query più mirate.

### Evidenza richiesta

Annota chiaramente quale tabella hai usato davvero nel tuo ambiente.

---

## Step 21 - Crea una query KQL utile di troubleshooting

Costruisci una query che mostri:

* timestamp
* risorsa
* categoria
* messaggio

Per esempio, se usi `AzureDiagnostics`:

```kusto
AzureDiagnostics
| where Resource contains "obsapp-aci"
| project TimeGenerated, Resource, Category, Message
| sort by TimeGenerated desc
| take 20
```

### Che cosa fa

È una query molto utile per troubleshooting rapido.

### Evidenza richiesta

Copia la query e i risultati nel file evidenze.

---

## Step 22 - Aggiorna il file ambiente del modulo

Apri `docs/azure_env.md` e aggiungi:

```markdown
LOG_ANALYTICS_WORKSPACE=<nome effettivo del workspace>
```

### Perché lo fai

Perché dal LAB11 in poi il workspace diventa una risorsa di riferimento del modulo.

---

## Step 23 - Crea il file delle evidenze

Crea:

```text
docs/evidence_lab11.md
```

Usa una struttura come questa:

```md
# LAB11 - Evidence

## Log Analytics Workspace
- Nome workspace:
- Resource Group:
- Regione:

## Diagnostic settings
- Risorsa sorgente:
- Nome diagnostic setting:
- Categorie abilitate:

## Test applicativi
- Endpoint usati:
- Note:

## Tabelle osservate nel workspace
- Tabelle trovate:
- Tabella usata davvero per il laboratorio:

## Query KQL eseguite

### Query esplorativa iniziale
[incollare query e output]

### Query su AzureDiagnostics o tabella reale
[incollare query e output]

### Query filtrata per ACI
[incollare query e output]

### Query con count()
[incollare query e output]

### Query con summarize by bin(TimeGenerated, 5m)
[incollare query e output]

### Query di troubleshooting
[incollare query e output]

## Sintesi finale
- Che cosa ho capito sulla centralizzazione dei log:
- Differenza tra log locali della risorsa e log nel workspace:
- Che cosa mi permette di fare KQL:
- Quali difficoltà ho incontrato:
```

---

## Step 24 - Verifica la struttura locale del laboratorio

Esegui:

```bash
ls -R
```

### Che cosa devi vedere

Almeno:

```text
docs/
cmdlog_lab11.txt
```

e dentro `docs`:

```text
evidence_lab11.md
```

---

## Step 25 - Consegna nel repository Git

Esegui:

```bash
git add docs/azure_env.md docs/evidence_lab11.md
git commit -m "[LAB11] Log Analytics e KQL completato"
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

Il Log Analytics Workspace esiste ed è visibile nel portale.

## Checkpoint #2

La ACI è collegata a una diagnostic setting che invia log al workspace.

## Checkpoint #3

Sono state generate nuove richieste verso la ACI per popolare i log.

## Checkpoint #4

Nel workspace sono state identificate una o più tabelle utili.

## Checkpoint #5

Sono state eseguite query KQL reali sui dati arrivati.

## Checkpoint #6

`azure_env.md` è stato aggiornato con il nome del workspace.

---

# 2. Criteri di completamento

Il LAB11 è da considerarsi completato se:

* il workspace è stato creato
* la raccolta log dalla ACI è stata configurata
* i log sono arrivati nel workspace
* sono state eseguite query KQL di esplorazione e filtraggio
* il file `docs/evidence_lab11.md` è completo

---

# 3. Che cosa stai imparando davvero

Questo laboratorio ti insegna almeno cinque cose decisive.

## 3.1 I log diventano molto più utili quando sono centralizzati

Non sei più costretto a interrogare ogni risorsa singolarmente.

## 3.2 KQL è uno strumento fondamentale per Observability su Azure

Ti permette di filtrare, cercare, aggregare e analizzare log in modo naturale.

## 3.3 Prima si esplora lo schema, poi si analizza

Nei sistemi reali devi prima capire dove stanno i dati e in quale forma arrivano.

## 3.4 L’analisi log è una pipeline

Cerchi, filtri, selezioni colonne, ordini, conti, raggruppi nel tempo.

## 3.5 Questo laboratorio prepara direttamente il LAB12

Per fare alert sui log, devi prima saperli centralizzare e interrogare.

---

# 4. Errori comuni da evitare

## Errore 1 - Aspettarsi i log immediatamente

L’ingestione può richiedere tempo.

## Errore 2 - Dare per scontato il nome della tabella

Controlla sempre dove sono arrivati davvero i dati.

## Errore 3 - Usare query troppo complesse troppo presto

Prima fai query esplorative semplici, poi raffinile.

## Errore 4 - Confondere AzureDiagnostics e log applicativi puri

`AzureDiagnostics` è un contenitore diagnostico Azure, non sempre un log applicativo “pulito”.

## Errore 5 - Pensare che KQL sia solo teoria

Nel mondo Azure KQL è uno strumento operativo quotidiano.

---

# 5. Conclusione

Questo laboratorio introduce la **centralizzazione dei log** e l’uso di **KQL** come strumento chiave di Observability su Azure.

Dopo aver completato il LAB11 devi avere chiaro che:

* i log osservabili diventano molto più potenti quando convergono in un workspace
* KQL è il linguaggio giusto per interrogare questi dati
* la centralizzazione è il passaggio necessario per arrivare ad alerting e dashboard seri
* il lavoro osservability non consiste nel “guardare qualche log”, ma nel costruire un sistema interrogabile e analizzabile

Questo è il ponte naturale verso il **LAB12**, dove userai proprio questi dati centralizzati per configurare **alert rules** e iniziare il monitoraggio proattivo.

```

Nel prossimo passaggio procedo con **LAB12 completo**.
```
