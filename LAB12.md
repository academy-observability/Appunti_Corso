
# LAB12 - Alerting: Alert Rule + Action Group su SLI (error_rate)

## Obiettivo del laboratorio

In questo laboratorio imparerai a trasformare i dati osservabili in **monitoraggio proattivo**.

Nei laboratori precedenti hai già:

- distribuito un’applicazione osservabile su Azure
- generato log applicativi
- centralizzato i log in **Log Analytics**
- interrogato i dati con **KQL**
- calcolato indicatori come:
  - total requests
  - error rate
  - top endpoint
  - latenza media

Adesso fai il passo successivo: userai una query KQL come base per creare una **Alert Rule**, collegherai un **Action Group** e verificherai lo stato **Fired**. Questo è esattamente l’obiettivo del LAB12 definito nella traccia del file condiviso. 

---

## Durata stimata

**4 ore**

---

## Cost Control - Gestione dei costi

Il file del modulo specifica che nel percorso continuativo **non va eliminato il Resource Group fino al LAB14**, mentre la cancellazione del Resource Group è prevista solo come modalità standalone facoltativa. 

Quindi in questo laboratorio:

- non creare risorse inutili
- non creare Action Group duplicati senza motivo
- non eliminare il Resource Group
- limita la configurazione a ciò che serve davvero per testare l’alert

---

## Modalità Percorso Continuativo 

**NON eliminare il Resource Group fino al LAB14.** :contentReference[oaicite:4]{index=4}

---

## Modalità Standalone (facoltativa)

Se stai eseguendo il laboratorio in isolamento, la traccia prevede come cleanup facoltativo:

```bash
az group delete --name rg-observability-lab --yes --no-wait
````



Nel percorso aula, però, **non eseguire questo comando adesso**.

---

# PARTE 1 - Concetti fondamentali, teoria e collegamento con Observability

# 1. Perché serve l’alerting

Finché guardi i dati solo manualmente, il tuo flusso è questo:

* apro il portale
* apro Log Analytics
* lancio una query
* verifico se c’è un problema

Questo approccio è utile per imparare, ma non basta per un sistema reale.
Per questo il laboratorio introduce il concetto di **alerting**, cioè la capacità del sistema di verificare automaticamente una condizione e segnalare quando diventa rilevante.

Nel file del modulo, gli obiettivi del LAB12 sono proprio:

* creare un Action Group
* creare una Alert Rule basata su query KQL
* simulare errori
* verificare lo stato **Fired**
* documentare soglia e motivazione, evitando alert rumorosi

---

# 2. Monitoraggio passivo e monitoraggio proattivo

## 2.1 Monitoraggio passivo

Osservi i dati quando decidi tu.

Esempio:

* esegui una query KQL
* scopri che l’error rate è aumentato

## 2.2 Monitoraggio proattivo

Definisci una regola e lasci che sia Azure Monitor a controllare la condizione.

Esempio:

* se `error_rate > 0.20`
* la regola deve scattare
* l’alert va in **Fired**
* viene associata un’azione

Questo è il salto concettuale del LAB12.

---

# 3. Che cos’è una Alert Rule

Una **Alert Rule** è una regola che verifica automaticamente una condizione su un segnale osservabile.

Nel LAB12 il segnale osservabile è il risultato di una **query KQL** su **Log Analytics**. La traccia è molto esplicita: la regola deve essere creata su **Monitor → Alerts → Create → Alert rule**, con scope il Log Analytics Workspace `law-observability` e condition di tipo **Custom log search**.

Una Alert Rule ha quindi almeno questi elementi:

* **Scope**: su quale risorsa o workspace lavora
* **Condition**: quale condizione controlla
* **Threshold**: quale soglia fa scattare l’alert
* **Evaluation frequency**: ogni quanto controlla
* **Window size**: su quale finestra temporale valuta
* **Severity**: gravità operativa
* **Action Group**: azione o notifica associata

---

# 4. Che cos’è un Action Group

Un **Action Group** definisce che cosa deve accadere quando l’alert si attiva.

Nel file condiviso, il LAB12 richiede esplicitamente di creare un Action Group con:

* Resource group: `rg-observability-lab`
* Nome: `ag-observability`
* Action: **Email** tramite Email/SMS/Push/Voice

Quindi:

* la **regola** decide quando scattare
* l’**Action Group** decide come reagire

Questa distinzione è fondamentale.

---

# 5. Perché usare una query KQL come base dell’alert

Questo laboratorio arriva subito dopo LAB11, in cui hai già imparato a centralizzare i log in **Log Analytics** e a interrogarli con **KQL**. Il file infatti mette LAB12 come dipendente da LAB11 completato e da ingestione log correttamente funzionante.

Usare KQL per l’alert ha tre vantaggi didattici:

* riutilizzi una competenza già vista
* definisci la condizione in modo trasparente
* le soglie si basano su dati reali del sistema

---

# 6. La query di `error_rate` prevista nel file

Il file del Modulo 2 propone esplicitamente questa query KQL per la condizione di alert:

```kql
ContainerInstanceLog_CL
| extend status = toint(parse_json(LogEntry_s).status)
| summarize error_rate = todouble(countif(status >= 400)) / count()
```

## Che cosa fa questa query

* legge la tabella `ContainerInstanceLog_CL`
* estrae il campo `status` dal JSON contenuto in `LogEntry_s`
* conta quante righe hanno `status >= 400`
* divide quel numero per il totale
* restituisce quindi l’**error rate**

Questa query è il cuore del laboratorio.

---

# 7. Soglia, frequenza e finestra temporale

Il file suggerisce per il LAB12 queste impostazioni semplici e coerenti:

* **Threshold**: `error_rate > 0.20`
* **Evaluation frequency**: 5 min
* **Window size**: 5 min
* **Severity**: 2 oppure 3

## Perché questa scelta è didatticamente sensata

* una soglia al 20% è abbastanza bassa da poter essere superata con un test pilotato
* 5 minuti di valutazione e finestra mantengono il laboratorio semplice
* severity 2 o 3 permette di classificare il problema come rilevante ma non catastrofico

---

# 8. Perché simulare errori con `/nope`

Il file indica esplicitamente che per generare errori bisogna:

1. recuperare l’IP pubblico della ACI
2. chiamare ripetutamente l’endpoint `/nope`
3. attendere 5-10 minuti per valutazione e ingestione
4. controllare gli alert Fired o la Alert history nel portale

Questo ha senso perché:

* `/nope` produce errori controllati
* l’error rate aumenta
* la query KQL dovrebbe rilevarli
* la regola dovrebbe andare in **Fired**

---

# 9. Che cosa significa “alert rumoroso”

Il file richiede anche di documentare **perché quella soglia** e di evitare alert rumorosi.

Un alert rumoroso è un alert che:

* scatta troppo spesso
* segnala condizioni poco importanti
* disturba più di quanto aiuti
* genera assuefazione

Nel laboratorio non devi solo farlo funzionare.
Devi anche spiegare perché la soglia scelta ha senso.

---

# 10. Architettura mentale del LAB12

```text
ACI / applicazione
  ↓
log centralizzati
  ↓
ContainerInstanceLog_CL
  ↓
query KQL su error_rate
  ↓
Alert Rule
  ↓
Action Group
  ↓
stato Fired
```

Questa catena è esattamente coerente con quanto richiesto dalla traccia del file.

---

# 11. Che cosa imparerai davvero

Alla fine del laboratorio dovrai aver capito che:

* una query KQL può diventare una condizione operativa
* una Alert Rule non è solo un oggetto del portale, ma una decisione sul comportamento atteso del sistema
* un Action Group è una reazione configurata
* lo stato **Fired** dimostra che la regola ha rilevato davvero la condizione
* l’alerting è il ponte tra osservabilità e operatività

---

# PARTE 2 - Laboratorio guidato passo-passo

## Prerequisiti

Secondo il file condiviso, per LAB12 servono:

* **LAB11 completato**
* **Log Analytics e KQL funzionanti**
* **ACI attiva**
* **ingestione log corretta**

---

## Step 1 - Setup locale in WSL Ubuntu

Esegui:

```bash
mkdir -p ~/course/lab12 && cd ~/course/lab12
script -a cmdlog_lab12.txt
mkdir -p docs
```

Questa è esattamente la sequenza indicata nel file del laboratorio.

### Che cosa fanno i comandi

#### `mkdir -p ~/course/lab12 && cd ~/course/lab12`

Crea la cartella del laboratorio ed entra nella cartella.

#### `script -a cmdlog_lab12.txt`

Registra la sessione del terminale nel file `cmdlog_lab12.txt`.

#### `mkdir -p docs`

Crea la cartella per le evidenze.

---

## Step 2 - Verifica che ACI e Log Analytics siano pronti

Prima di creare qualsiasi alert, verifica dal portale Azure che:

* la **ACI** sia attiva
* il **Log Analytics Workspace** esista
* i log siano visibili e interrogabili

Nel file questi prerequisiti sono obbligatori.

### Checkpoint operativo

Devi essere in grado di dire:

* qual è il nome della ACI
* qual è il nome del workspace
* dove si trovano i log

---

## Step 3 - Apri Azure Monitor

Nel portale Azure cerca:

```text
Monitor
```

Apri il servizio.

### Che cosa osservare

Nel menu laterale verifica la presenza di sezioni come:

* Alerts
* Alert rules
* Action groups
* Fired alerts o Alert history

### Evidenza richiesta

Esegui uno screenshot della home di Azure Monitor.

---

## Step 4 - Crea l’Action Group

Nel portale segui il percorso indicato nel file:

* **Monitor**
* **Alerts**
* **Action groups**
* **Create**

### Impostazioni da usare

Secondo la traccia:

* **Resource group**: `rg-observability-lab`
* **Nome**: `ag-observability`
* **Action**: Email/SMS/Push/Voice → **Email (tuo indirizzo)**

### Checkpoint #1

L’Action Group deve risultare creato. Questo checkpoint è esplicitamente richiesto.

### Evidenza richiesta

Esegui uno screenshot dell’Action Group creato.

---

## Step 5 - Avvia la creazione della Alert Rule

Segui il percorso indicato nel file:

* **Monitor**
* **Alerts**
* **Create**
* **Alert rule**

### Scope

Il file richiede di selezionare come scope il **Log Analytics workspace** `law-observability`.

### Che cosa devi capire

Lo scope è fondamentale: se scegli la risorsa sbagliata, l’alert non leggerà i dati corretti.

---

## Step 6 - Imposta la condizione con query KQL

Nella creazione della regola, imposta come **Condition**:

* **Custom log search** (Log query)

Usa la query prevista nel file:

```kql
ContainerInstanceLog_CL
| extend status = toint(parse_json(LogEntry_s).status)
| summarize error_rate = todouble(countif(status >= 400)) / count()
```

### Che cosa devi fare

Inserisci questa query come base della condizione di alert.

### Che cosa misura

La query calcola il rapporto tra:

* richieste con `status >= 400`
* totale richieste

cioè l’**error_rate**.

---

## Step 7 - Imposta soglia e parametri di valutazione

Usa le impostazioni consigliate riportate nel file:

* **Threshold**: `error_rate > 0.20`
* **Evaluation frequency**: 5 min
* **Window size**: 5 min
* **Severity**: 2 oppure 3

### Perché usare questi valori

Sono stati scelti per rendere l’alert:

* facile da testare
* sensibile a un degrado reale
* non eccessivamente complesso

### Evidenza richiesta

Annota questi valori nel file delle evidenze.

---

## Step 8 - Collega l’Action Group

Nella sezione **Actions** della Alert Rule collega:

```text
ag-observability
```

Questa associazione è esplicitamente richiesta dal file.

### Che cosa devi capire

La regola rileva la condizione.
L’Action Group definisce l’azione associata.

### Checkpoint #2

L’alert rule deve risultare **creata e abilitata**, come indicato dalla traccia.

---

## Step 9 - Recupera l’IP pubblico della ACI

Da WSL Ubuntu esegui il comando previsto dal file:

```bash
ACI_PUBLIC_IP=$(az container show --resource-group rg-observability-lab --name obsapp-aci --query ipAddress.ip -o tsv)
echo "$ACI_PUBLIC_IP"
```

### Che cosa fa questo comando

* interroga la ACI nel Resource Group `rg-observability-lab`
* recupera l’IP pubblico
* lo salva nella variabile `ACI_PUBLIC_IP`

### Evidenza richiesta

Copia il comando e il valore ottenuto nel file delle evidenze.

---

## Step 10 - Simula errori applicativi

Il file richiede di generare errori con questo ciclo:

```bash
for i in {1..40}; do
  curl -s "http://$ACI_PUBLIC_IP:8000/nope" > /dev/null
done
```

### Che cosa fa questo comando

* esegue 40 richieste all’endpoint `/nope`
* scarta l’output
* genera un numero elevato di richieste in errore

### Perché è utile

Con 40 chiamate a `/nope` aumenti artificialmente l’error rate e rendi molto più probabile che la soglia `> 0.20` venga superata.

### Evidenza richiesta

Annota nel file delle evidenze che il traffico di errore è stato generato.

---

## Step 11 - Attendi la valutazione dell’alert

Il file indica di attendere:

* **5-10 minuti**
* per consentire **ingestione dei log + valutazione della regola**

### Cosa devi capire

L’alerting non è istantaneo. Dipende da:

* tempi di ingestione log
* evaluation frequency
* window size

Questo è normale.

---

## Step 12 - Verifica gli alert Fired nel portale

Nel portale segui quanto indicato dal file:

* **Monitor**
* **Alerts**
* **Fired alerts**
  oppure
* **Alert history**

### Checkpoint #3

Devi verificare che l’alert sia in stato **Fired**, se la soglia è stata superata. Questo checkpoint è esplicitamente richiesto dal file.

### Che cosa osservare

Annota:

* nome alert
* severity
* stato
* ora di attivazione
* eventuale resource/workspace associato

### Evidenza richiesta

Esegui uno screenshot con l’alert in stato **Fired** oppure con la Alert history visibile.

---

## Step 13 - Se l’alert non è Fired, verifica la catena logica

Se non compare lo stato Fired, controlla in ordine:

1. il workspace giusto è `law-observability`?
2. la query restituisce dati?
3. la ACI è ancora attiva?
4. i log arrivano davvero in `ContainerInstanceLog_CL`?
5. la soglia è coerente con i dati generati?
6. la regola è abilitata?
7. hai atteso abbastanza?

### Obiettivo didattico

Imparare che il troubleshooting degli alert è parte integrante dell’Observability.

### Evidenza richiesta

Annota eventuali correzioni fatte.

---

## Step 14 - Crea il file delle evidenze

Crea il file richiesto nel repository:

```text
docs/evidence_lab12.md
```

Il file condiviso specifica che l’evidenza deve contenere almeno:

* nome Action Group
* configurazione soglia
* screenshot alert fired / alert history
* breve spiegazione: perché quella soglia e cosa significa

Puoi usare questa struttura completa:

````md
# LAB12 - Evidence

## 1. Action Group
- Nome: ag-observability
- Tipo di azione:
- Indirizzo email usato:

## 2. Alert Rule
- Scope:
- Workspace: law-observability
- Query KQL:
```kql
ContainerInstanceLog_CL
| extend status = toint(parse_json(LogEntry_s).status)
| summarize error_rate = todouble(countif(status >= 400)) / count()
````

* Threshold: error_rate > 0.20
* Evaluation frequency: 5 min
* Window size: 5 min
* Severity:

## 3. Simulazione errori

* Comando usato:

```bash
for i in {1..40}; do
  curl -s "http://$ACI_PUBLIC_IP:8000/nope" > /dev/null
done
```

## 4. Verifica alert

* Fired: SÌ/NO
* Orario osservato:
* Screenshot allegato: SÌ/NO

## 5. Motivazione soglia

* Perché è stata scelta questa soglia:
* Perché un alert troppo rumoroso è un problema:

## 6. Note finali

* Che cosa ho capito su Alert Rule:
* Che cosa ho capito su Action Group:
* Che cosa significa Fired:

````

---

## Step 15 - Consegna nel repository Git

Il file prevede questa consegna: 

```bash
git add docs/evidence_lab12.md
git commit -m "[LAB12] Alerting completato"
git push
````

### Che cosa fanno i comandi

* `git add` aggiunge il file delle evidenze
* `git commit` salva la versione locale
* `git push` pubblica sul repository remoto

---

# PARTE 3 - Checkpoint, criteri di completamento e significato professionale

# 1. Checkpoint riassuntivi

## Checkpoint #1

L’**Action Group** `ag-observability` è stato creato.

## Checkpoint #2

La **Alert Rule** è stata creata ed è abilitata.

## Checkpoint #3

Dopo la simulazione errori, l’alert è stato verificato in stato **Fired** oppure nella alert history, se la soglia è stata superata.

---

# 2. Criteri di completamento

Secondo il file del modulo, il LAB12 è completato se risultano soddisfatti questi criteri:

* Action Group creato
* Alert Rule attiva
* Alert Fired verificato
* Evidence completa

---

# 3. Che cosa stai imparando davvero

Questo laboratorio ti insegna almeno cinque cose importanti.

## 3.1 Una query può diventare una regola operativa

Non resta solo uno strumento di analisi manuale.

## 3.2 L’alerting trasforma l’osservabilità in azione

Passi dall’osservazione alla reazione automatizzata.

## 3.3 La soglia conta quanto la query

Una soglia sbagliata produce alert inutili o rumorosi.

## 3.4 Alert Rule e Action Group sono due oggetti distinti

La regola rileva la condizione.
Il gruppo azioni definisce la reazione.

## 3.5 Fired è una prova operativa

Non basta creare la regola. Devi dimostrare che rileva davvero una condizione reale.

---

# 4. Errori comuni da evitare

## Errore 1 - Creare l’alert senza verificare prima i log

Se i log non arrivano, la regola non potrà funzionare bene.

## Errore 2 - Impostare lo scope sbagliato

Il file richiede il workspace `law-observability`. Se sbagli scope, l’alert guarda nel posto sbagliato.

## Errore 3 - Usare una soglia poco testabile

Nel laboratorio la soglia deve poter essere superata con i dati generati.

## Errore 4 - Aspettarsi alert immediati

Il file stesso chiede di attendere 5-10 minuti. Quindi no, non è un interruttore magico.

## Errore 5 - Documentare male la motivazione della soglia

Il laboratorio richiede esplicitamente di spiegare perché quella soglia e cosa significa, evitando alert rumorosi.

---

# 5. Conclusione

Questo laboratorio introduce il **monitoraggio proattivo** nel Modulo 2.

Dopo aver completato il LAB12 devi avere chiaro che:

* una query KQL può diventare una condizione di allerta
* Azure Monitor può valutare in autonomia una regola
* un Action Group definisce la reazione
* lo stato **Fired** dimostra che la condizione è stata davvero rilevata
* alerting e observability non sono separati: l’alerting è uno degli esiti più operativi dell’observability

Questo prepara il **LAB13**, che nel file è definito come laboratorio sui **Workbooks**, con dashboard su total requests, error rate e top endpoint, oltre al confronto tra SQL e Log Analytics.
