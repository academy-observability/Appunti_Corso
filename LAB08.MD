# LAB08 - Azure Foundations orientato all’Observability

## Obiettivo del laboratorio

In questo laboratorio rientri nel focus principale del percorso: **Observability**.

Hai già visto i fondamenti Azure introduttivi. Adesso devi costruire il primo vero ponte tra:

- conoscenza di base della piattaforma Azure
- organizzazione dell’ambiente cloud del modulo
- osservabilità della piattaforma
- preparazione delle risorse che verranno riutilizzate nei laboratori successivi

Questo LAB08 ha una funzione molto importante nel Modulo 2: non serve a creare un’applicazione complessa, ma a costruire il **contesto operativo stabile** su cui lavorerai fino al LAB14. In particolare, la traccia ufficiale del modulo richiede che tu sappia usare Azure Portal e Cloud Shell, comprendere Subscription, Resource Group, Region e risorsa, creare e verificare un Resource Group via CLI, definire una convenzione di naming riutilizzabile e osservare Activity Log come prima forma di osservabilità della piattaforma. Inoltre, nel percorso continuativo non devi eliminare il Resource Group fino al LAB14.

---

## Durata indicativa

**4 ore**

---

## Cost control - gestione dei costi

Azure è un ambiente a consumo. Ogni risorsa attiva può generare costi.  
Nel Modulo 2 devi mantenere un comportamento disciplinato fin dall’inizio.

Regole operative:

- verifica sempre il livello tariffario o SKU
- scegli, quando disponibile, opzioni minime o economiche
- evita SKU Premium se non richiesti
- evita ridondanza geografica non necessaria
- evita VM di dimensione elevata

La traccia del laboratorio lo rende esplicito: per questo percorso vanno privilegiate opzioni come Free tier, Basic/B1/F1, Standard_LRS per storage e SKU Basic per ACR. 

---

## Modalità del percorso

### Modalità percorso continuativo
Questo LAB fa parte di un percorso di più giorni.  
Le risorse create qui verranno riutilizzate nei LAB successivi.  
Per questo motivo **non devi eliminare il Resource Group fino al termine del LAB14**. 

### Modalità standalone
Se esegui il laboratorio in modo isolato, alla fine puoi eliminare il Resource Group. La traccia ufficiale propone questo comando: :contentReference[oaicite:3]{index=3}

```bash
az group delete --name rg-observability-lab --yes --no-wait
````

Nel percorso aula, però, **non usarlo adesso**.

---

# PARTE 1 - Concetti fondamentali e collegamento con Observability

# 1. Perché questo laboratorio esiste

Dopo i laboratori introduttivi Azure, potresti essere tentato di pensare:

> “Ho già visto tenant, subscription, Resource Group e Cloud Shell. Perché rifarlo?”

Perché qui non stai più facendo semplicemente “Azure basics”.
Qui stai costruendo il **fondamento osservabile** del Modulo 2.

Nei laboratori successivi userai:

* Resource Group condiviso
* Azure Container Registry
* Azure Container Instances
* Azure SQL
* Log Analytics
* Alert rules
* Workbook

Se in questo punto del percorso non chiarisci bene:

* dove lavori
* come si chiama l’ambiente
* come lo referenzi in modo coerente
* dove osservi la piattaforma
* quali evidenze raccogli

allora nei LAB09-14 inizieranno confusione, errori di naming, risorse duplicate, query sbagliate e la solita nebbia operativa che trasforma tutto in copia-incolla rituale.

---

# 2. Il vero focus Observability del LAB08

Questo laboratorio introduce una distinzione essenziale:

## 2.1 Osservabilità della piattaforma

Qui osservi eventi che riguardano **Azure come piattaforma**.

Esempi:

* creazione di un Resource Group
* aggiornamento di una risorsa
* cancellazione di una risorsa
* operazioni amministrative

Questa osservabilità passa soprattutto da strumenti come:

* Azure Portal
* Activity Log
* stato delle risorse
* metadati e configurazioni

Nel LAB08 la traccia ufficiale chiede esplicitamente di osservare gli eventi nell’**Activity Log** come forma di osservabilità “della piattaforma”. 

---

## 2.2 Osservabilità dell’applicazione

Qui osservi invece il comportamento di un’applicazione in esecuzione.

Esempi:

* richieste HTTP
* codici di risposta
* durata delle richieste
* correlation/request id
* log applicativi
* errori runtime

Questa parte entrerà soprattutto nei LAB09-13.

---

## 2.3 Perché la distinzione è importante

Se non distingui questi due livelli, succede questo errore classico:

* guardi Activity Log aspettandoti di vedere i dettagli applicativi
* guardi i log applicativi aspettandoti di vedere le operazioni amministrative Azure

Non funziona così.

In forma semplice:

```text
Observability della piattaforma
= cosa accade alle risorse Azure

Observability dell’applicazione
= cosa accade dentro il servizio o il container che gira su Azure
```

---

# 3. Subscription, Resource Group, Region e risorsa nel contesto del Modulo 2

La traccia del LAB08 richiede di saper capire Subscription, Resource Group, Region e risorsa. 
Qui però non basta ripetere la definizione da esame AZ-900. Devi capirla nel contesto del modulo.

## 3.1 Subscription

È il contenitore amministrativo entro cui lavorerai.

Nel modulo significa:

* lì verranno create le risorse del percorso
* lì verranno registrati i costi
* lì esisterà il tuo perimetro operativo

---

## 3.2 Resource Group

È il contenitore logico delle risorse del Modulo 2.

Nel LAB08 il Resource Group non è un semplice esercizio.
È il **contenitore persistente** che porterai avanti fino al LAB14. La traccia ufficiale indica infatti che il Resource Group creato qui viene riutilizzato nei laboratori successivi e non va eliminato fino alla fine del percorso. 

Quindi il Resource Group, in questo contesto, rappresenta:

* il perimetro del progetto
* il contenitore di quasi tutte le risorse del modulo
* il punto principale di filtro per osservare attività e oggetti

---

## 3.3 Region

La region è la collocazione geografica o logica della risorsa Azure.

Nel laboratorio viene proposta, per esempio, `westeurope`. La traccia chiede esplicitamente di scegliere una region, ad esempio `westeurope`, e usarla nella creazione del Resource Group. 

Nel contesto del modulo, usare la stessa region per più risorse aiuta a:

* mantenere coerenza
* ridurre complessità
* evitare dispersione inutile
* semplificare il ragionamento architetturale

---

## 3.4 Risorsa

Una risorsa è il singolo oggetto Azure concreto.

Nel modulo saranno risorse, per esempio:

* il Resource Group stesso come contenitore logico
* l’ACR del LAB09
* l’ACI del LAB09
* il SQL Server e il DB del LAB10
* il Log Analytics Workspace del LAB11
* alert e workbook nei LAB12-13

Il LAB08 serve a chiarire **dove nasceranno** tutte queste risorse e con quale convenzione verranno chiamate.

---

# 4. Perché il Resource Group è il primo oggetto osservabile del modulo

Nel LAB08 il primo oggetto che crei davvero è il Resource Group.
Sembra banale. Non lo è.

Quando crei il Resource Group ottieni già tre cose utilissime:

1. un contenitore logico persistente
2. una prima risorsa Azure concreta
3. un primo evento osservabile nell’Activity Log

Questo è importante perché introduce un concetto molto reale:

> prima ancora di osservare l’applicazione, osservi il ciclo di vita dell’infrastruttura che la ospiterà

È il primo passo di una mentalità osservability matura.

---

# 5. Activity Log: che cos’è davvero

La traccia del LAB08 richiede di osservare gli eventi su Activity Log e di trovare l’evento di creazione del Resource Group annotando timestamp, operation name e status.

## 5.1 Che cosa registra

Activity Log registra le operazioni amministrative sulla piattaforma Azure.

Per esempio:

* create
* update
* delete
* start
* stop
* assegnazioni e modifiche di configurazione

---

## 5.2 Cosa non registra

Non è il log applicativo del tuo container o della tua app.

Non mostra direttamente:

* payload applicativi
* request id HTTP della tua API
* stack trace del tuo codice
* tempo di risposta di una singola richiesta utente

Quello arriverà nei lab successivi.

---

## 5.3 Perché è importante nel percorso Observability

Perché ti abitua a una domanda professionale fondamentale:

> dove guardo per capire che cosa è successo davvero?

Nel LAB08 la risposta è:

* se il problema riguarda la piattaforma o la gestione della risorsa, guardo **Activity Log**
* se il problema riguarderà il comportamento applicativo, guarderò log e strumenti diversi nei prossimi lab

---

# 6. Naming convention persistente: perché è obbligatoria

La traccia del laboratorio richiede di creare una convenzione di naming riutilizzabile per tutto il Modulo 2 e di salvare un file `docs/azure_env.md` con variabili persistenti come `RG`, `LOCATION` e, nei lab successivi, nomi di ACR, ACI, SQL e Log Analytics Workspace.

Questa non è burocrazia. È disciplina operativa.

## 6.1 Perché serve

Perché nei laboratori successivi dovrai ricordare e riusare:

* nome del Resource Group
* region
* nome ACR
* login server ACR
* nome ACI
* IP pubblico ACI
* nome SQL Server
* nome DB
* nome del Log Analytics Workspace

Senza un file ambiente persistente, inizierai a:

* cercare nomi a memoria
* sbagliare copy-paste
* interrogare la risorsa sbagliata
* perdere tempo
* sporcare il percorso con nomi incoerenti

---

## 6.2 Il principio professionale dietro il file ambiente

Il file `azure_env.md` è una versione didattica molto semplice di un concetto professionale reale:

* inventario minimo delle risorse
* variabili d’ambiente documentate
* contesto condiviso e riusabile

È una piccola abitudine, ma molto sana.

---

# 7. Architettura mentale del LAB08

Il LAB08 può essere letto così:

```text
Utente
  ↓
Azure Portal / Cloud Shell
  ↓
Subscription
  ↓
Resource Group persistente del Modulo 2
  ↓
Activity Log
  ↓
Primo livello di osservabilità della piattaforma
```

E in parallelo:

```text
Repository locale
  ↓
docs/azure_env.md
  ↓
Memoria operativa del Modulo 2
```

---

# 8. Dove opererai

La traccia del laboratorio specifica chiaramente i tre ambienti da usare: Azure Portal per navigazione risorse e Activity Log, Azure Cloud Shell in Bash per i comandi `az`, e WSL Ubuntu solo per creare cartella locale e registrare il log dei comandi. 

Questa distinzione è utile anche didatticamente:

## Azure Portal

Lo usi per:

* navigare
* verificare visivamente
* consultare Activity Log
* confermare che le risorse esistono davvero

## Azure Cloud Shell

Lo usi per:

* eseguire comandi Azure CLI
* creare il Resource Group
* verificare contesto e stato

## WSL Ubuntu

Lo usi per:

* creare cartella lab
* raccogliere evidenze
* registrare i comandi eseguiti
* gestire file `docs/`

---

# 9. Cosa imparerai davvero in questo lab

A fine LAB08 non devi uscire pensando soltanto:

* “so creare un RG”
* “so aprire Cloud Shell”

Devi uscire sapendo che:

* l’ambiente Azure del modulo è stato definito
* esiste un perimetro stabile per i LAB successivi
* hai visto il primo punto di osservazione della piattaforma
* hai iniziato a costruire un inventario persistente delle risorse
* hai capito che Observability non è solo log applicativo, ma anche visibilità sulle operazioni infrastrutturali

---

# PARTE 2 - Laboratorio guidato passo-passo

## Prerequisiti

Per svolgere il laboratorio ti servono:

* account Azure attivo, Free o aziendale
* accesso al portale Azure
* browser aggiornato
* repository GitHub del LAB08 disponibile

Questi prerequisiti sono richiesti esplicitamente dalla traccia del modulo. 

---

## Step 1 - Prepara l’ambiente locale in WSL Ubuntu

Apri WSL Ubuntu ed esegui:

```bash
mkdir -p ~/corso_obs/NOME-REPOSITORY/labs/lab08 && cd ~/corso_obs/NOME-REPOSITORY/labs/lab08
script -a cmdlog_lab08.txt
mkdir -p docs
```

### Che cosa fanno questi comandi

#### `mkdir -p ~/corso_obs/NOME-REPOSITORY/labs/lab08 && cd ~/corso_obs/NOME-REPOSITORY/labs/lab08`

* `mkdir -p` crea la cartella del laboratorio
* `~/corso_obs/NOME-REPOSITORY/labs/lab08` è il percorso scelto
* `&&` esegue il comando successivo solo se il primo è andato bene
* `cd ~/corso_obs/NOME-REPOSITORY/labs/lab08` entra nella cartella del laboratorio

#### `script -a cmdlog_lab08.txt`

Avvia una registrazione della sessione terminale dentro il file `cmdlog_lab08.txt`.

* `script` registra quello che fai nel terminale
* `-a` significa append, cioè aggiungi al file invece di sovrascrivere
* `cmdlog_lab08.txt` è il file che conterrà il log dei comandi

#### `mkdir -p docs`

Crea la cartella `docs`, dove salverai i file di evidenza.

### Perché lo fai

Perché la traccia richiede di usare WSL Ubuntu per creare la cartella locale e registrare il log dei comandi. 

### Checkpoint

Devi avere:

* cartella `~/corso_obs/NOME-REPOSITORY/labs/lab08`
* file `cmdlog_lab08.txt` in registrazione
* cartella `docs`

---

## Step 2 - Apri Azure Portal e verifica subscription e tenant

Apri il browser e vai a:

```text
https://portal.azure.com
```

Nel portale:

1. cerca **Subscriptions**
2. apri la subscription che userai
3. cerca **Microsoft Entra ID**
4. verifica il nome del tenant/directory

Questa sequenza è richiesta direttamente nella traccia del lab. 

### Che cosa devi capire

Devi saper indicare chiaramente:

* quale subscription userai
* quale tenant o directory stai usando
* la differenza tra subscription, Resource Group e singola risorsa

### Checkpoint #1

Devi essere in grado di spiegare con parole tue:

* **Subscription** = contenitore amministrativo
* **Resource Group** = contenitore logico del progetto/lab
* **Risorsa** = oggetto concreto Azure

### Evidenza richiesta

Prendi nota nel file finale di:

* nome subscription
* nome tenant/directory

---

## Step 3 - Avvia Azure Cloud Shell in modalità Bash

Nel portale Azure, in alto, clicca sull’icona **Cloud Shell**.

Se richiesto:

* seleziona **Bash**
* crea lo storage necessario per Cloud Shell con l’opzione standard

La traccia prevede esplicitamente l’uso di Cloud Shell in modalità Bash. 

### Che cosa devi capire

Cloud Shell è il terminale Azure già pronto all’uso nel browser.
Ti permette di eseguire comandi `az` senza installare o configurare subito tutto in locale.

---

## Step 4 - Verifica versione CLI e account attivo

In Cloud Shell esegui:

```bash
az version
az account show
```

Questi due comandi sono quelli richiesti dalla traccia base del laboratorio. 

### Spiegazione dei comandi

#### `az version`

Mostra la versione di Azure CLI e dei componenti installati.

Serve a verificare che lo strumento sia disponibile e funzionante.

#### `az account show`

Mostra le informazioni dell’account e del contesto attivo.

### Che cosa devi osservare

Nel secondo comando verifica soprattutto:

* nome subscription
* tenant
* utente
* eventuale subscription attiva o default

### Checkpoint #2

Nel report finale devi incollare un output parziale di `az account show`, evitando dati sensibili se necessario, come richiesto dalla traccia. 

---

## Step 5 - Imposta le variabili del laboratorio

In Cloud Shell esegui:

```bash
export LOCATION="westeurope"
export RG="rg-observability-lab"
```

La traccia del laboratorio propone esattamente questa impostazione, con `westeurope` come esempio e `rg-observability-lab` come nome del Resource Group.

### Che cosa fa `export`

`export` crea una variabile d’ambiente disponibile nella sessione shell corrente.

#### `LOCATION="westeurope"`

Memorizza la region che userai.

#### `RG="rg-observability-lab"`

Memorizza il nome del Resource Group.

### Perché è utile

Perché così nei comandi successivi puoi usare:

* `$LOCATION`
* `$RG`

senza dover riscrivere ogni volta i valori completi.

### Checkpoint

Esegui, se vuoi verificare:

```bash
echo "$LOCATION"
echo "$RG"
```

Dovresti vedere:

```text
westeurope
rg-observability-lab
```

---

## Step 6 - Crea il Resource Group da Azure CLI

Esegui:

```bash
az group create --name "$RG" --location "$LOCATION"
az group list --output table
```

Questi sono i comandi previsti dalla traccia ufficiale per la creazione e verifica del Resource Group.

### Spiegazione del primo comando

```bash
az group create --name "$RG" --location "$LOCATION"
```

* `az` richiama Azure CLI
* `group` indica che stai lavorando sui Resource Group
* `create` chiede di crearne uno
* `--name "$RG"` usa il nome salvato nella variabile `RG`
* `--location "$LOCATION"` usa la region salvata in `LOCATION`

### Che cosa aspettarti

Il comando restituisce un output in JSON con informazioni simili a:

* nome
* id
* location
* provisioningState
* tags, se presenti

### Spiegazione del secondo comando

```bash
az group list --output table
```

* `list` elenca tutti i Resource Group disponibili nella subscription attiva
* `--output table` mostra il risultato in tabella

### Checkpoint #3

Il Resource Group deve risultare visibile:

* nella CLI
* nel portale Azure

Questo è uno dei checkpoint espliciti della traccia. 

### Evidenza richiesta

Copia nel report:

* output della creazione
* output della lista dei Resource Group

---

## Step 7 - Verifica il Resource Group nel portale Azure

Nel portale cerca:

```text
Resource groups
```

Apri il servizio e controlla che esista:

```text
rg-observability-lab
```

### Che cosa devi osservare

Verifica almeno:

* nome
* region
* subscription
* presenza nell’elenco

### Perché lo fai

Per confermare che la risorsa creata da CLI sia visibile anche dal portale.

Questa doppia verifica è didatticamente utile perché ti abitua a usare CLI e portale in modo complementare.

---

## Step 8 - Consulta Activity Log

Nel portale:

1. cerca **Monitor**
2. nel menu di Monitor apri **Activity log**
3. filtra per Resource Group: `rg-observability-lab`

Questa è esattamente la procedura prevista dalla traccia del laboratorio.

### Che cosa devi cercare

Trova l’evento di creazione del Resource Group.

### Che cosa devi annotare

La traccia richiede di annotare:

* timestamp
* operation name
* status

### Che cosa stai osservando davvero

Qui stai guardando la **traccia amministrativa** di un’operazione sulla piattaforma Azure.

Stai quindi esercitando la prima forma di osservabilità del modulo:

* non osservi ancora l’app
* osservi la piattaforma che ospiterà l’app

### Checkpoint #4

Devi aver individuato almeno un evento significativo relativo alla creazione del Resource Group.

---

## Step 9 - Crea il file ambiente persistente del Modulo 2

Nel repository del laboratorio crea il file:

```text
docs/azure_env.md
```

La traccia del lab richiede esplicitamente questo file e ne propone il contenuto iniziale, con `RG`, `LOCATION` e variabili da popolare nei LAB successivi.

Inserisci questo contenuto:

```markdown
# Azure Environment (Modulo 2)

RG=rg-observability-lab
LOCATION=westeurope

# Popola nei LAB successivi:
# ACR_NAME=
# ACR_LOGIN_SERVER=
# ACI_NAME=
# ACI_PUBLIC_IP=
# SQL_SERVER=
# SQL_DB=obsdb
# LOG_ANALYTICS_WORKSPACE=
```

### Perché è importante

Questo file diventa la tua memoria operativa del Modulo 2.

Nei laboratori successivi andrai ad aggiornare questi campi con i valori reali delle risorse create.

---

## Step 10 - Crea il file delle evidenze del laboratorio

Crea:

```text
docs/evidence_lab08.md
```

La traccia ufficiale richiede questo file e ne fornisce la struttura base.

Puoi usare questa versione dettagliata:

```markdown
# LAB08 - Evidence

## Subscription / Tenant
- Nome subscription:
- Nome tenant/directory:

## Resource Group
- Nome: rg-observability-lab
- Location: westeurope

## Azure CLI
### az account show
[incollare estratto utile]

### az group create
[incollare output sintetico o estratto]

### az group list --output table
[incollare output]

## Activity Log
- Timestamp:
- Operation name:
- Status:

## Note finali
- Che cosa è chiaro:
- Che cosa è nuovo:
- Che differenza ho capito tra osservabilità della piattaforma e osservabilità dell’applicazione:
```

### Perché aggiungo quest’ultima riga

Perché questo è il vero concetto didattico chiave del LAB08.

---

## Step 11 - Verifica i file creati in locale

Da WSL Ubuntu, nella cartella del laboratorio, esegui:

```bash
ls -R
```

### Che cosa fa

Elenca ricorsivamente file e cartelle.

### Che cosa devi vedere

Dovresti avere almeno:

```text
docs/
cmdlog_lab08.txt
```

e dentro `docs`:

```text
azure_env.md
evidence_lab08.md
```

### Evidenza utile

Puoi incollare anche questo output nel tuo file o usarlo come controllo finale.

---

## Step 12 - Effettua la consegna nel repository Git

La traccia ufficiale indica questa consegna:

```bash
git add docs/azure_env.md docs/evidence_lab08.md
git commit -m "[LAB08] Azure foundations completato"
git push
```

### Spiegazione dei comandi

#### `git add docs/azure_env.md docs/evidence_lab08.md`

Aggiunge i due file all’area di staging.

#### `git commit -m "[LAB08] Azure foundations completato"`

Crea un commit con messaggio descrittivo.

#### `git push`

Invia il commit al repository remoto.

### Perché lo fai

Perché il laboratorio non è completo finché le evidenze non sono salvate e consegnate.

---

# PARTE 3 - Checkpoint, criteri di completamento e significato professionale

# 1. Checkpoint riassuntivi

## Checkpoint #1

Sai indicare correttamente:

* Subscription
* Resource Group
* Resource

## Checkpoint #2

Hai eseguito:

```bash
az version
az account show
```

e hai raccolto un estratto utile dell’output. 

## Checkpoint #3

Il Resource Group è visibile:

* in CLI
* nel portale Azure 

## Checkpoint #4

Hai trovato nell’Activity Log l’evento di creazione del Resource Group e annotato:

* timestamp
* operation name
* status

---

# 2. Criteri di completamento

Secondo la traccia del modulo, il LAB08 è completato se risultano soddisfatti questi criteri: Resource Group creato, Activity Log verificato, `docs/azure_env.md` creato e `docs/evidence_lab08.md` completo.

Quindi il laboratorio è da considerarsi concluso solo se hai:

* creato `rg-observability-lab`
* verificato almeno un evento nell’Activity Log
* creato il file `docs/azure_env.md`
* creato e compilato `docs/evidence_lab08.md`
* eseguito commit e push

---

# 3. Che cosa stai imparando davvero

Questo laboratorio ti sta insegnando tre cose professionali molto importanti.

## 3.1 L’ambiente cloud va preparato prima di usarlo

Non si parte subito dal container o dal dashboard.
Prima si crea un perimetro coerente.

## 3.2 L’osservabilità non riguarda solo l’applicazione

Riguarda anche la piattaforma, le operazioni, gli eventi amministrativi, il ciclo di vita delle risorse.

## 3.3 Le risorse vanno documentate

Il file `azure_env.md` è una forma minima di inventario operativo.
Nel mondo reale, chi non documenta le risorse poi passa metà del tempo a cercare nomi, endpoint e configurazioni che avrebbe dovuto annotare subito.

---

# 4. Errori comuni da evitare

## Errore 1 - Eliminare il Resource Group a fine LAB08

Nel percorso continuativo non devi farlo.
La traccia lo vieta esplicitamente fino al LAB14. 

## Errore 2 - Confondere Activity Log con log applicativi

Activity Log mostra operazioni amministrative Azure, non il comportamento HTTP dell’app.

## Errore 3 - Non salvare il contesto operativo

Se non compili `azure_env.md`, nei LAB successivi inizierai a perdere nomi e riferimenti.

## Errore 4 - Usare region o naming incoerenti

Il Modulo 2 deve restare coerente. Cambiare naming o region senza motivo aumenta confusione e rischio di errore.

## Errore 5 - Fare il lab solo dal portale o solo da CLI

Il valore didattico qui sta anche nel vedere la stessa realtà da due prospettive:

* CLI per creare e interrogare
* portale per verificare e osservare

---

# 5. Conclusione

Questo laboratorio chiude la fase “solo introduttiva” e apre davvero il **Modulo 2 Observability su Azure**.

Dopo aver completato il LAB08 devi avere chiaro che:

* il tuo ambiente Azure del modulo è stato inizializzato
* esiste un Resource Group persistente che userai fino al LAB14
* sai osservare almeno il primo livello di attività della piattaforma con Activity Log
* hai creato un file ambiente che diventerà il riferimento dei laboratori successivi
* hai capito la differenza tra osservabilità della piattaforma e osservabilità dell’applicazione

Questo è il punto da cui, nel LAB09, potrai finalmente passare al deploy del **container osservabile** su Azure, con ACR, ACI, endpoint pubblico, log runtime e primi segnali applicativi. La traccia del LAB09 prevede infatti la creazione di un ACR Basic, il push di un’immagine Docker, il deploy su ACI con IP pubblico e la verifica di endpoint e log, inclusi request-id, status e durata.

