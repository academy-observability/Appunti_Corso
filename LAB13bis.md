# LAB13bis - Dashboarding avanzato: KQL, Workbooks e Grafana in Azure

## Obiettivo del laboratorio

In questo laboratorio estenderai il lavoro svolto nel LAB13.

Nel LAB13 hai già:

- costruito un **Workbook** in Azure Monitor
- visualizzato **total requests**, **error rate** e **top endpoint**
- confrontato i dati tra **SQL Database** e **Log Analytics**

In questo LAB13bis andrai oltre.

L’obiettivo non è rifare lo stesso esercizio con un nome diverso, ma approfondire due aspetti che nel LAB13 sono stati affrontati solo in modo introduttivo:

1. **uso più maturo di KQL per dashboard e analisi visuale**
2. **sperimentazione pratica di Grafana in Azure**

Alla fine del laboratorio avrai realizzato:

- un **Workbook avanzato** con visualizzazioni più ricche e più leggibili
- una **dashboard Grafana in Azure** basata sugli stessi dati di Azure Monitor
- un confronto ragionato tra:
  - query eseguite in Log Analytics
  - pannelli di Workbook
  - pannelli di Grafana

---

## Durata stimata

**4 ore**

---

## Cost Control - Gestione dei costi

Questo laboratorio prevede due percorsi.

### Percorso obbligatorio
Usa **Dashboards with Grafana** dentro Azure Monitor.

Questo percorso è adatto al laboratorio perché:

- usa solo dati Azure Monitor
- non richiede una nuova risorsa Grafana dedicata
- è il percorso più semplice e immediato

### Estensione facoltativa
Usa **Azure Managed Grafana**.

Questa estensione è utile per vedere il servizio gestito completo, ma crea una nuova risorsa Azure. Se la esegui:

- usa uno SKU semplice
- non lasciare la risorsa attiva inutilmente
- elimina la risorsa al termine, se non deve essere riutilizzata

---

## Prerequisiti

Per svolgere questo laboratorio servono:

- **LAB13_standalone completato**
- **ACI attiva**
- **Log Analytics Workspace attivo e con dati recenti**
- **accesso al portale Azure**
- **WSL Ubuntu con Azure CLI autenticata**

È consigliato avere disponibili i valori:

- `RG`
- `LAW_NAME`
- `ACI_NAME`
- `SQL_SERVER`
- `SQL_DB`

---

# PARTE 1 - Concetti fondamentali

# 1. Perché serve un LAB13bis

Il LAB13 ti ha insegnato a costruire una dashboard in Azure Monitor, ma in un percorso didattico serio questo non basta.

Una dashboard utile non nasce solo da tre query corrette. Serve anche:

- scegliere la **giusta metrica o dimensione**
- scegliere la **giusta visualizzazione**
- capire che cosa mostra davvero il pannello
- distinguere tra **volume**, **qualità**, **latenza**, **distribuzione**
- confrontare strumenti diversi che interrogano le stesse sorgenti

Questo laboratorio serve proprio a consolidare queste competenze.

---

# 2. Workbooks e Grafana non sono la stessa cosa

## 2.1 Workbooks

I **Workbooks** di Azure Monitor sono molto adatti quando vuoi:

- restare dentro il portale Azure
- costruire una vista operativa documentata
- combinare testo, query e grafici
- lavorare rapidamente con Log Analytics e KQL

## 2.2 Grafana

**Grafana** è particolarmente utile quando vuoi:

- costruire dashboard operative più orientate al monitoraggio continuo
- lavorare con pannelli, layout e filtri più flessibili
- riusare un modello di dashboard molto diffuso nel mondo observability
- combinare più sorgenti dati in un’unica UI

In breve:

- **Workbook** = ottimo per analisi e documentazione tecnica nel portale
- **Grafana** = ottimo per dashboard operative e visualizzazione continua

---

# 3. Le due modalità Grafana in Azure

Nel contesto Azure puoi sperimentare Grafana in due modi.

## 3.1 Dashboards with Grafana

È l’opzione più semplice per questo laboratorio.

Caratteristiche:

- disponibile direttamente nel portale Azure
- nessuna nuova risorsa Grafana da creare
- adatta quando usi solo dati Azure Monitor
- molto utile per fare esperienza pratica con pannelli e dashboard Grafana

## 3.2 Azure Managed Grafana

È il servizio Grafana gestito completo.

Caratteristiche:

- è una risorsa Azure dedicata
- offre l’esperienza Grafana completa
- usa identità gestita o service principal
- è la scelta corretta quando servono funzionalità più evolute

Per questo LAB13bis:

- il **percorso obbligatorio** usa **Dashboards with Grafana**
- l’**estensione facoltativa** usa **Azure Managed Grafana**

---

# 4. Che cosa deve saper fare una buona dashboard osservabile

Una buona dashboard non deve solo “mostrare dati”.

Deve aiutarti a rispondere velocemente a domande operative come:

- quanto traffico sta arrivando?
- quanto di quel traffico è in errore?
- quali endpoint pesano di più?
- dove si concentra la latenza?
- il comportamento osservato è stabile oppure peggiora?

Per questo in questo laboratorio userai query che coprono almeno cinque prospettive:

- **volume richieste**
- **error rate**
- **distribuzione status code**
- **latenza media e P95**
- **top endpoint**

---

# 5. KQL e visualizzazione: il punto chiave

In observability non basta scrivere query corrette.

Bisogna anche capire se la visualizzazione scelta è adatta al tipo di domanda.

Esempi:

- un **timechart** è ottimo per vedere trend temporali
- una **bar chart** è utile per confrontare categorie
- una **tabella** è adatta quando vuoi leggere il dettaglio
- una **stat visualization** è utile per un singolo numero sintetico

Uno degli obiettivi didattici del LAB13bis è proprio imparare questa relazione:

```text
Domanda operativa
  ↓
query corretta
  ↓
visualizzazione corretta
```

---

# 6. Che cosa costruirai nel laboratorio

Nel LAB13bis costruirai due artefatti principali.

## 6.1 Workbook avanzato
Con almeno queste sezioni:

- Total Requests nel tempo
- Error Rate nel tempo
- Distribuzione degli status code
- Latenza media e P95 per endpoint
- Top Endpoint

## 6.2 Dashboard Grafana in Azure
Con almeno questi pannelli:

- Total Requests
- Error Rate
- Top Endpoint

Poi confronterai:

- facilità di costruzione
- leggibilità
- flessibilità
- casi d’uso più adatti

---

# PARTE 2 - Laboratorio guidato passo-passo

## Step 1 - Setup locale in WSL Ubuntu

Esegui:

```bash
mkdir -p ~/course/lab13bis && cd ~/course/lab13bis
script -a cmdlog_lab13bis.txt
mkdir -p docs
```

### Che cosa fanno i comandi

- creano la cartella del laboratorio
- registrano la sessione terminale
- preparano la cartella delle evidenze

---

## Step 2 - Verifica il contesto Azure attivo

Esegui:

```bash
az account show --output table
```

### Evidenza richiesta

Copia l’output nel file evidenze.

---

## Step 3 - Imposta le variabili del laboratorio

Adatta i nomi ai valori reali del tuo ambiente:

```bash
export RG="rg-observability-lab12-standalone"
export LAW_NAME="law-observability-lab12"
export ACI_NAME="obsapp-aci"
```

Verifica:

```bash
echo "$RG"
echo "$LAW_NAME"
echo "$ACI_NAME"
```

---

## Step 4 - Genera traffico controllato per alimentare le dashboard

Recupera l’IP pubblico della ACI:

```bash
ACI_PUBLIC_IP=$(az container show \
  --resource-group "$RG" \
  --name "$ACI_NAME" \
  --query ipAddress.ip -o tsv)

echo "$ACI_PUBLIC_IP"
```

Genera traffico misto:

```bash
for i in {1..20}; do
  curl -s "http://$ACI_PUBLIC_IP:8000/health" > /dev/null
  curl -s "http://$ACI_PUBLIC_IP:8000/time" > /dev/null
  curl -s "http://$ACI_PUBLIC_IP:8000/nope" > /dev/null
  sleep 1
done
```

### Perché lo fai

Così ottieni:

- richieste riuscite
- richieste in errore
- endpoint diversi
- un dataset recente adatto a visualizzazioni temporalmente coerenti

### Evidenza richiesta

Annota il comando usato e l’orario di generazione del traffico.

---

## Step 5 - Apri Log Analytics e verifica il dataset reale

Nel portale Azure:

- apri il workspace `LAW_NAME`
- apri **Logs**

Esegui questa query di verifica:

```kql
ContainerInstanceLog_CL
| project TimeGenerated, Message
| take 20
```

### Che cosa osservare

Verifica che:

- la tabella `ContainerInstanceLog_CL` contenga dati
- il campo `Message` contenga JSON
- siano presenti campi come `path`, `status`, `latency_ms`

### Evidenza richiesta

Copia una riga di esempio del campo `Message` nel file evidenze.

---

## Step 6 - Costruisci e valida le query KQL avanzate

In questo step devi eseguire e validare le query che userai poi sia nel Workbook sia in Grafana.

---

### Query A - Total Requests nel tempo per endpoint

```kql
ContainerInstanceLog_CL
| where TimeGenerated > ago(30m)
| extend payload = parse_json(Message)
| extend path = tostring(payload.path)
| where isnotempty(path)
| summarize total_requests=count() by bin(TimeGenerated, 5m), path
| order by TimeGenerated asc
| render timechart
```

### Che cosa osservare

- andamento del traffico nel tempo
- confronto tra endpoint diversi

---

### Query B - Error Rate nel tempo

```kql
ContainerInstanceLog_CL
| where TimeGenerated > ago(30m)
| extend payload = parse_json(Message)
| extend status = toint(payload.status)
| where isnotnull(status)
| summarize total=count(), errors=countif(status >= 400) by bin(TimeGenerated, 5m)
| extend error_rate = todouble(errors) / todouble(total)
| order by TimeGenerated asc
| render timechart
```

### Che cosa osservare

- rapporto tra traffico totale e traffico in errore
- variazioni dell’error rate nel tempo

---

### Query C - Distribuzione degli status code

```kql
ContainerInstanceLog_CL
| where TimeGenerated > ago(30m)
| extend payload = parse_json(Message)
| extend status = tostring(payload.status)
| where isnotempty(status)
| summarize hits=count() by status
| order by hits desc
```

### Che cosa osservare

- peso relativo di `200`, `404`, `500`, altri eventuali status

---

### Query D - Latenza media e P95 per endpoint

```kql
ContainerInstanceLog_CL
| where TimeGenerated > ago(30m)
| extend payload = parse_json(Message)
| extend path = tostring(payload.path), latency_ms = todouble(payload.latency_ms)
| where isnotempty(path) and isnotnull(latency_ms)
| summarize avg_latency_ms=avg(latency_ms), p95_latency_ms=percentile(latency_ms, 95) by path
| order by p95_latency_ms desc
```

### Che cosa osservare

- differenza tra media e P95
- endpoint che generano maggiore latenza percepita

---

### Query E - Top Endpoint

```kql
ContainerInstanceLog_CL
| where TimeGenerated > ago(30m)
| extend payload = parse_json(Message)
| extend path = tostring(payload.path)
| where isnotempty(path)
| summarize hits=count() by path
| order by hits desc
```

### Che cosa osservare

- endpoint più usati
- distribuzione del traffico applicativo

---

## Step 7 - Crea un Workbook avanzato

Nel portale Azure:

- apri **Monitor**
- apri **Workbooks**
- crea un nuovo workbook oppure duplica quello del LAB13

Salvalo con nome:

```text
wb-observability-advanced
```

### Sezioni minime richieste

Il workbook deve contenere almeno queste 5 sezioni:

1. **Total Requests nel tempo per endpoint**
2. **Error Rate nel tempo**
3. **Distribuzione status code**
4. **Latenza media e P95 per endpoint**
5. **Top Endpoint**

### Indicazioni consigliate

- usa **timechart** per Total Requests ed Error Rate
- usa **bar chart** o **table** per status code e top endpoint
- usa **table** per latenza media e P95
- aggiungi un breve testo descrittivo iniziale che spieghi lo scopo della dashboard

### Checkpoint #1

Il workbook avanzato deve essere salvato e deve contenere 5 sezioni funzionanti.

### Evidenza richiesta

Esegui screenshot delle 5 sezioni oppure uno screenshot panoramico leggibile.

---

## Step 8 - Accedi a Dashboards with Grafana in Azure Monitor

Nel portale Azure:

- cerca **Monitor**
- apri **Dashboards with Grafana**

### Nota importante

Questo è il percorso principale del laboratorio.

Usalo per fare esperienza con dashboard Grafana in Azure senza creare una nuova risorsa Managed Grafana.

---

## Step 9 - Crea una dashboard Grafana

Dentro la UI Grafana nel portale:

- seleziona **New**
- seleziona **New dashboard**
- seleziona **Add visualization**

Quando richiesto, scegli una **supported data source**.
Per questo laboratorio usa:

- **Azure Monitor**

### Nota didattica

L’interfaccia può variare leggermente nel tempo, ma il flusso concettuale resta questo:

```text
Nuovo dashboard
  ↓
nuovo pannello
  ↓
origine dati Azure Monitor
  ↓
query
  ↓
scelta visualizzazione
```

---

## Step 10 - Crea il pannello 1: Total Requests

Nel pannello Grafana:

- usa l’origine dati **Azure Monitor**
- seleziona la modalità **Logs**
- seleziona il workspace corretto
- incolla questa query:

```kql
ContainerInstanceLog_CL
| where TimeGenerated > ago(30m)
| extend payload = parse_json(Message)
| extend path = tostring(payload.path)
| where isnotempty(path)
| summarize total_requests=count() by bin(TimeGenerated, 5m), path
| order by TimeGenerated asc
```

### Visualizzazione consigliata

- **Time series**

### Titolo pannello consigliato

```text
Total Requests by Path
```

---

## Step 11 - Crea il pannello 2: Error Rate

Crea un nuovo pannello con questa query:

```kql
ContainerInstanceLog_CL
| where TimeGenerated > ago(30m)
| extend payload = parse_json(Message)
| extend status = toint(payload.status)
| where isnotnull(status)
| summarize total=count(), errors=countif(status >= 400) by bin(TimeGenerated, 5m)
| extend error_rate = todouble(errors) / todouble(total)
| order by TimeGenerated asc
```

### Visualizzazione consigliata

- **Time series**

### Titolo pannello consigliato

```text
Error Rate
```

---

## Step 12 - Crea il pannello 3: Top Endpoint

Crea un nuovo pannello con questa query:

```kql
ContainerInstanceLog_CL
| where TimeGenerated > ago(30m)
| extend payload = parse_json(Message)
| extend path = tostring(payload.path)
| where isnotempty(path)
| summarize hits=count() by path
| order by hits desc
```

### Visualizzazione consigliata

- **Bar chart** oppure **Table**

### Titolo pannello consigliato

```text
Top Endpoint
```

---

## Step 13 - Salva la dashboard Grafana

Salva la dashboard con nome:

```text
grafana-observability-overview
```

### Checkpoint #2

La dashboard Grafana deve contenere almeno 3 pannelli funzionanti:

- Total Requests
- Error Rate
- Top Endpoint

### Evidenza richiesta

Esegui uno screenshot della dashboard completa.

---

## Step 14 - Confronta Workbook e Grafana

A questo punto hai due dashboard costruite a partire dallo stesso backend osservabile:

- **Workbook avanzato**
- **Dashboard Grafana**

Devi confrontarle in modo tecnico, non estetico.

### Domande guida

1. In quale strumento hai costruito più rapidamente i pannelli?
2. In quale strumento hai trovato più facile interpretare il dato?
3. Dove ti è sembrato più naturale usare KQL?
4. Dove il layout visivo ti è sembrato più orientato al monitoraggio continuo?
5. In quale contesto useresti Workbook e in quale contesto useresti Grafana?

### Evidenza richiesta

Scrivi una risposta ragionata di **8-12 righe**.

---

## Step 15 - Migliora il confronto con SQL del LAB13

Apri il risultato del LAB13 relativo al confronto tra:

- **SQL Database (`requests_log`)**
- **Log Analytics**

Aggiungi ora un terzo punto di vista:

- **Grafana**

### Obiettivo

Capire che Grafana non introduce una nuova fonte dati, ma una **nuova modalità di consultazione** delle stesse sorgenti.

### Domanda da discutere nel report

In che senso Grafana cambia la fruizione del dato, pur non cambiando necessariamente il dato stesso?

### Spunti di risposta

Puoi ragionare su aspetti come:

- Grafana riorganizza la lettura operativa del dato
- la dashboard favorisce monitoraggio visivo continuo
- la stessa query può risultare più o meno leggibile a seconda dello strumento
- un pannello può evidenziare trend che in una tabella sono meno immediati
- uno strumento di visualizzazione non sostituisce la qualità della query sottostante

### Evidenza richiesta

Scrivi un confronto ragionato di **6-10 righe**.

---

## Step 16 - Estensione facoltativa: crea Azure Managed Grafana

Esegui questa estensione solo se:

- hai tempo residuo
- hai i permessi necessari
- il docente decide di includerla

### 16.1 Registra provider ed estensione CLI

```bash
az provider register -n Microsoft.Dashboard --wait
az extension add --name amg
```

### 16.2 Crea il workspace Managed Grafana

```bash
export GRAFANA_NAME="amgobs$(date +%m%d%H%M%S)"

az grafana create \
  --name "$GRAFANA_NAME" \
  --resource-group "$RG"
```

### 16.3 Recupera endpoint

```bash
az grafana show \
  --name "$GRAFANA_NAME" \
  --resource-group "$RG" \
  --query "properties.endpoint" -o tsv
```

Apri l’endpoint nel browser.

---

## Step 17 - Verifica l’origine dati Azure Monitor in Managed Grafana

Dentro Azure Managed Grafana:

- apri **Connections**
- apri **Data sources**
- verifica che **Azure Monitor** sia presente

### Nota tecnica

Per i nuovi workspace Azure Managed Grafana, l’origine dati **Azure Monitor** è fornita come data source integrata.

Se necessario:

- apri il data source Azure Monitor
- configura l’autenticazione tramite **Managed Identity**
- seleziona la subscription corretta
- usa **Save & test**

---

## Step 18 - Troubleshooting dell’estensione Managed Grafana

Se incontri problemi, verifica questi casi comuni.

### Caso 1 - Non vedi la subscription nel data source Azure Monitor

Possibile causa:

- la managed identity del workspace Grafana non ha accesso sufficiente

### Caso 2 - Non vedi il Log Analytics workspace

Possibile causa:

- mancano permessi di lettura sul workspace

### Azione suggerita

Se il docente lo autorizza, assegna alla managed identity di Azure Managed Grafana un ruolo di lettura sul perimetro richiesto.

Esempi tipici:

- **Monitoring Reader** sul subscription scope o sulla risorsa da leggere
- **Reader** sul Log Analytics workspace se il workspace non viene trovato

---

## Step 19 - Crea il file delle evidenze

Crea:

```text
docs/evidence_lab13bis.md
```

Usa questa struttura:

````md
# LAB13bis - Evidence

## 1. Contesto Azure
- Resource Group:
- Log Analytics Workspace:
- ACI:
- Dashboard Grafana usata: Dashboards with Grafana / Azure Managed Grafana

## 2. Traffico generato
- Comando usato:
- Orario approssimativo:

## 3. Query KQL validate

### Query A - Total Requests by Path
[incollare query]

### Query B - Error Rate
[incollare query]

### Query C - Status Distribution
[incollare query]

### Query D - Avg + P95 Latency
[incollare query]

### Query E - Top Endpoint
[incollare query]

## 4. Workbook avanzato
- Nome workbook: wb-observability-advanced
- Numero sezioni:
- Screenshot allegati: SÌ/NO
- Osservazioni:

## 5. Dashboard Grafana
- Nome dashboard: grafana-observability-overview
- Numero pannelli:
- Screenshot allegati: SÌ/NO
- Osservazioni:

## 6. Confronto Workbook vs Grafana
[risposta ragionata 8-12 righe]

## 7. Relazione con il confronto SQL del LAB13
[risposta ragionata 6-10 righe]

## 8. Estensione Managed Grafana (facoltativa)
- Eseguita: SÌ/NO
- Nome workspace Managed Grafana:
- Endpoint:
- Problemi riscontrati:
- Soluzione adottata:

## 9. Note finali
- Che cosa ho capito sul rapporto tra query e visualizzazione:
- Quale strumento considero più utile per analisi tecnica:
- Quale strumento considero più utile per monitoraggio operativo:
````

---

## Step 20 - Consegna nel repository Git

Esegui:

```bash
git add docs/evidence_lab13bis.md
git commit -m "[LAB13bis] Dashboarding avanzato e Grafana completato"
git push
```

---

# PARTE 3 - Checkpoint, criteri di completamento e significato professionale

# 1. Checkpoint riassuntivi

## Checkpoint #1

È stato creato un **Workbook avanzato** con almeno 5 sezioni funzionanti.

## Checkpoint #2

È stata creata una **dashboard Grafana** con almeno 3 pannelli funzionanti.

## Checkpoint #3

È stato svolto un confronto tecnico tra Workbook e Grafana.

## Checkpoint #4

È stata esplicitata la relazione tra:

- sorgente dati
- query
- strumento di visualizzazione

## Checkpoint #5

È stata eseguita, oppure almeno compresa e documentata, l’estensione su Azure Managed Grafana.

---

# 2. Criteri di completamento

Il LAB13bis è completato se:

- le query KQL avanzate sono state eseguite e comprese
- il workbook avanzato è stato costruito
- la dashboard Grafana è stata costruita
- il confronto Workbook vs Grafana è stato documentato
- il file `docs/evidence_lab13bis.md` è completo

---

# 3. Che cosa stai imparando davvero

Questo laboratorio ti insegna almeno sei cose importanti.

## 3.1 Una query corretta non basta da sola

Serve anche una visualizzazione coerente con la domanda operativa.

## 3.2 KQL è il motore, la dashboard è il mezzo

Se la query è debole, anche la dashboard sarà debole.

## 3.3 Workbook e Grafana rispondono a bisogni diversi

Non sono duplicati perfetti.

## 3.4 La visualizzazione cambia il modo in cui interpreti il dato

Lo stesso dato può risultare più o meno leggibile a seconda dello strumento.

## 3.5 Grafana è parte del panorama reale dell’observability

Saperlo usare in Azure è una competenza concreta.

## 3.6 Un buon osservability engineer non si limita a raccogliere dati

Deve anche saperli rendere chiari, leggibili e utili.

---

# 4. Errori comuni da evitare

## Errore 1 - Costruire pannelli senza validare prima la query in Log Analytics

Prima si valida il dato, poi si costruisce la dashboard.

## Errore 2 - Confondere bellezza grafica e valore operativo

Una dashboard “bella” ma poco interpretabile serve a poco.

## Errore 3 - Usare la stessa visualizzazione per tutto

Ogni domanda ha una visualizzazione più adatta.

## Errore 4 - Pensare che Grafana generi un nuovo dato

Grafana visualizza, non inventa il dato.

## Errore 5 - Non considerare time range e finestra temporale

Molte differenze apparenti nascono semplicemente da finestre temporali diverse.

## Errore 6 - Lasciare risorse Managed Grafana attive senza motivo

Se hai creato una risorsa dedicata solo per il laboratorio, fai cleanup.

---

# 5. Cleanup facoltativo dell’estensione Managed Grafana

Se hai creato Azure Managed Grafana e non ti serve più, eliminala:

```bash
az grafana delete \
  --name "$GRAFANA_NAME" \
  --resource-group "$RG" \
  --yes
```

---

# 6. Conclusione

Questo LAB13bis completa il passaggio da una semplice query osservability a una vera capacità di **dashboarding operativo**.

Dopo aver completato il laboratorio devi avere chiaro che:

- una dashboard efficace nasce da query corrette e domande operative chiare
- Workbooks e Grafana hanno sovrapposizioni, ma non sono equivalenti
- Grafana in Azure è uno strumento reale e spendibile nel lavoro di observability
- la visualizzazione non sostituisce l’analisi, ma la rende più efficace
