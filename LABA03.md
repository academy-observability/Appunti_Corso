# LABA03 - Azure Foundations: Storage Account, Blob Storage e gestione base dei file nel cloud

## Obiettivo del laboratorio

In questo laboratorio imparerai a usare uno dei servizi fondamentali di Azure: lo **Storage Account**.

Prima di arrivare a database, log centralizzati, metriche, backup o deployment applicativi, è importante capire come Azure gestisce i dati e l’archiviazione.

In questo laboratorio vedrai:

- che cos’è uno Storage Account
- quali servizi di archiviazione può contenere
- che cos’è Blob Storage
- che cos’è un container blob
- come creare uno Storage Account
- come creare un contenitore per i file
- come caricare un file dal portale
- come eseguire le stesse operazioni da Azure CLI
- come leggere in modo corretto i comandi usati

Al termine del laboratorio sarai in grado di:

- spiegare che cosa sia uno Storage Account
- distinguere almeno a livello introduttivo Blob, File Share, Queue e Table
- creare uno Storage Account
- creare un container blob
- caricare un file nel cloud
- elencare i blob caricati
- comprendere il ruolo dell’archiviazione in Azure

---

## Scenario

In un ambiente cloud i dati devono essere archiviati in modo strutturato.

Un’applicazione può avere bisogno di salvare:

- file
- immagini
- log
- documenti
- backup
- dati temporanei
- contenuti statici

Azure mette a disposizione uno strumento fondamentale per farlo: lo **Storage Account**.

Lo Storage Account non è un singolo file e non è una cartella.  
È il contenitore di alto livello che permette di usare diversi servizi di archiviazione.

In questo laboratorio userai in particolare **Blob Storage**, cioè il servizio pensato per archiviare oggetti e file.

---

## Prerequisiti

Per svolgere il laboratorio occorrono:

- account Azure funzionante
- accesso al portale Azure
- accesso a Cloud Shell
- completamento dei laboratori LABA01 e LABA02
- un Resource Group già disponibile
- connessione Internet stabile

---

## Durata indicativa

**2 ore circa**

---

## Competenze in uscita

Al termine del laboratorio sarai in grado di:

- spiegare il ruolo di uno Storage Account
- descrivere in modo semplice i principali tipi di storage Azure
- creare e configurare uno Storage Account
- creare un blob container
- caricare e visualizzare file nel cloud
- usare comandi Azure CLI di base per lavorare con lo storage

---

# PARTE 1 - Concetti fondamentali

# 1. Perché serve uno Storage Account

In ogni sistema informativo esiste il problema di dove conservare i dati.

Anche un’applicazione molto semplice può aver bisogno di archiviare:

- immagini caricate dagli utenti
- file PDF
- log applicativi
- report
- documenti
- file di configurazione
- backup

Nel cloud questi dati non vengono normalmente salvati “sul disco locale del server” come si farebbe in un ambiente tradizionale molto semplice.

In Azure, una delle soluzioni principali per archiviare dati è lo **Storage Account**.

Lo Storage Account è una risorsa Azure che mette a disposizione servizi di archiviazione scalabili e gestiti.

---

# 2. Che cos’è uno Storage Account

Uno **Storage Account** è il contenitore di alto livello che racchiude uno o più servizi di archiviazione Azure.

Puoi immaginarlo come una “porta di accesso” a diversi modi di salvare dati.

Uno Storage Account può essere usato per:

- archiviare file
- archiviare oggetti
- fornire condivisioni file
- gestire code di messaggi
- gestire archivi chiave-valore semplici

In questo laboratorio lavorerai soprattutto con **Blob Storage**, che è il servizio più immediato da capire per chi inizia.

---

# 3. I principali tipi di storage Azure

## 3.1 Blob Storage

**Blob** significa Binary Large Object.

In pratica, nel contesto Azure, Blob Storage è il servizio usato per archiviare oggetti e file nel cloud.

Esempi tipici:

- immagini
- file di testo
- file JSON
- documenti PDF
- file CSV
- log esportati
- backup applicativi

Il Blob Storage è molto usato perché è semplice, scalabile e adatto a molti scenari.

---

## 3.2 Azure Files

Azure Files permette di creare condivisioni file simili a file share di rete.

Può essere utile quando serve un accesso a file condivisi in stile più tradizionale.

Per questo laboratorio non lo userai direttamente, ma è importante sapere che esiste.

---

## 3.3 Queue Storage

Queue Storage serve a gestire messaggi in coda tra componenti applicativi.

È utile in architetture distribuite, ma non è l’oggetto di questo laboratorio.

---

## 3.4 Table Storage

Table Storage è un archivio NoSQL molto semplice basato su chiavi.

Anche questo non è il focus del laboratorio, ma fa parte dei servizi disponibili in uno Storage Account.

---

# 4. Che cos’è un container blob

Dentro il Blob Storage, i file non vengono messi “direttamente nello storage account” in modo disordinato.

Si usa un livello organizzativo chiamato **container**.

Un **blob container** è un contenitore logico che raccoglie blob, cioè file o oggetti.

La gerarchia semplificata è:

```text
Storage Account
 └── Blob Container
      └── Blob (file/oggetto)
````

Esempio:

```text
stlaba03mario
 └── docs
      └── appunti.txt
```

In questo esempio:

* `stlaba03mario` è lo Storage Account
* `docs` è il container
* `appunti.txt` è il blob

---

# 5. Differenza tra Storage Account, container e file

Questa distinzione deve essere molto chiara.

## Storage Account

È la risorsa Azure di alto livello.

## Container

È il contenitore logico interno usato per organizzare i blob.

## Blob

È il singolo file o oggetto archiviato.

In forma semplice:

* lo Storage Account contiene
* il container organizza
* il blob è il dato concreto

---

# 6. Naming e vincoli sugli Storage Account

I nomi degli Storage Account devono rispettare regole precise.

In generale:

* devono essere univoci a livello globale
* devono usare lettere minuscole e numeri
* non devono contenere spazi
* non devono contenere caratteri speciali non ammessi
* devono avere una lunghezza limitata

Per questo motivo i nomi tipici sono compatti, ad esempio:

```text
stlaba03mario01
```

oppure:

```text
stobsdev001
```

Il nome deve essere unico in Azure.
Questo significa che se un nome è già usato da qualcun altro, dovrai sceglierne uno diverso.

---

# 7. Ridondanza e replica dei dati

Quando si crea uno Storage Account, Azure chiede anche il tipo di ridondanza.

Per beginners è sufficiente capire il concetto generale.

La ridondanza indica **quante copie dei dati vengono mantenute** e in quale ambito geografico o logico.

Un’opzione comune nei laboratori è:

* **LRS** = Locally Redundant Storage

In modo semplice significa che Azure mantiene copie dei dati nello stesso datacenter o nella stessa area locale gestita.

Per i laboratori introduttivi questa scelta è normalmente sufficiente.

---

# 8. Perché Blob Storage è importante anche per Observability

Anche se in questo laboratorio stai ancora lavorando sulle basi Azure, Blob Storage tornerà utile anche in scenari più avanzati.

Può essere usato per:

* esportare log
* salvare file di report
* archiviare evidenze
* ospitare file statici
* conservare output di script
* integrare pipeline e processi automatici

Per questo motivo è importante imparare bene la differenza tra:

* risorsa di archiviazione
* contenitore logico
* singolo oggetto salvato

---

# 9. Cosa farai nella parte pratica

Nella parte pratica eseguirai queste attività:

1. verificherai la subscription attiva
2. verificherai il Resource Group disponibile
3. creerai uno Storage Account dal portale
4. esplorerai le sezioni principali della risorsa
5. creerai un container blob
6. caricherai un file dal portale
7. leggerai le proprietà del file
8. userai Azure CLI per interrogare lo Storage Account
9. proverai a creare un secondo container da CLI
10. osserverai l’Activity Log

L’obiettivo non è solo “fare clic e lanciare comandi”, ma capire cosa rappresentano i vari oggetti creati.

---

# PARTE 2 - Attività pratiche guidate

## Step 1 - Verifica la subscription attiva

Apri Cloud Shell in modalità Bash ed esegui:

```bash
az account show --output table
```

### Che cosa fa questo comando

* `az` richiama Azure CLI
* `account` indica che stai lavorando sulle informazioni dell’account Azure
* `show` chiede di mostrare il contesto attivo
* `--output table` visualizza il risultato in formato tabellare, più leggibile per un principiante

### Che cosa devi osservare

Controlla almeno questi elementi:

* nome subscription
* stato
* tenant ID
* utente
* indicazione della subscription di default

### Perché lo fai

Prima di creare qualsiasi risorsa è sempre necessario controllare in quale subscription stai lavorando.

### Evidenza richiesta

Copia l’output nel file delle evidenze.

---

## Step 2 - Verifica il Resource Group da usare

Se hai già creato il Resource Group nel laboratorio precedente, elenca i Resource Group disponibili:

```bash
az group list --output table
```

### Che cosa fa questo comando

* `group` indica che stai lavorando sui Resource Group
* `list` chiede l’elenco dei gruppi disponibili
* `--output table` mostra i risultati in forma leggibile

### Che cosa devi osservare

Controlla che sia presente il tuo Resource Group principale, ad esempio:

```text
rg-laba02-mario
```

oppure un nome analogo assegnato nel laboratorio precedente.

### Perché lo fai

Lo Storage Account verrà creato dentro un Resource Group.

### Evidenza richiesta

Copia l’output nel file delle evidenze.

---

## Step 3 - Apri il servizio Storage accounts nel portale

Nel portale Azure cerca:

```text
Storage accounts
```

Apri il servizio.

### Che cosa devi osservare

Guarda la pagina iniziale e individua:

* elenco degli Storage Account esistenti, se presenti
* pulsante di creazione
* eventuali colonne come nome, resource group, location, SKU

### Obiettivo dello step

Prendere familiarità con la sezione del portale dedicata all’archiviazione.

### Evidenza richiesta

Esegui uno screenshot della pagina Storage accounts.

---

## Step 4 - Avvia la creazione di uno Storage Account

Dalla pagina **Storage accounts**, clicca su **Create**.

Compila con attenzione i campi principali.

### Impostazioni suggerite

* **Subscription**: seleziona la subscription del laboratorio
* **Resource group**: seleziona il tuo Resource Group
* **Storage account name**: scegli un nome univoco, ad esempio:

```text
stlaba03<tuonome>01
```

Esempio:

```text
stlaba03mario01
```

* **Region**: scegli una regione disponibile, ad esempio `West Europe`
* **Primary service**: Blob Storage o impostazione standard proposta dal portale
* **Performance**: Standard
* **Redundancy**: LRS

### Perché questi valori

Per un laboratorio introduttivo si scelgono impostazioni semplici e generalmente economiche.

### Attenzione

Se il nome non viene accettato, significa spesso che non è univoco.
In questo caso aggiungi numeri o iniziali.

Esempio:

```text
stlaba03mario02
```

### Evidenza richiesta

Esegui uno screenshot della schermata di riepilogo prima della creazione oppure della risorsa appena creata.

---

## Step 5 - Apri lo Storage Account creato

Dopo la creazione, apri lo Storage Account.

### Che cosa devi osservare

Nella schermata principale cerca di individuare:

* nome della risorsa
* Resource Group
* region
* subscription
* tipo di ridondanza
* menu laterale della risorsa

### Obiettivo dello step

Capire che lo Storage Account è una risorsa Azure vera e propria, con proprietà, configurazioni e funzionalità.

### Evidenza richiesta

Annota nel file evidenze:

* nome Storage Account
* Resource Group
* region
* ridondanza

---

## Step 6 - Esplora la sezione Containers

Nel menu dello Storage Account cerca la sezione relativa ai blob.

Apri:

```text
Data storage > Containers
```

### Che cosa devi osservare

Controlla se l’elenco è vuoto oppure se sono presenti container già esistenti.

### Obiettivo dello step

Capire che i file non si caricano direttamente “nello Storage Account” senza struttura, ma vengono organizzati in container.

---

## Step 7 - Crea un blob container dal portale

Clicca su **+ Container**.

Inserisci un nome semplice, ad esempio:

```text
docs
```

oppure:

```text
labfiles
```

Per esempio:

```text
docs
```

Conferma la creazione.

### Che cosa hai fatto

Hai creato un contenitore logico dentro il Blob Storage.

### Differenza importante

Non hai ancora caricato un file.
Hai creato il contenitore che servirà a ospitare uno o più blob.

### Evidenza richiesta

Esegui uno screenshot del container creato.

---

## Step 8 - Prepara un file locale da caricare

Puoi creare un piccolo file di testo nel tuo ambiente Cloud Shell oppure usare un file locale dal tuo PC.

### Opzione A - File locale dal tuo computer

Crea un file chiamato:

```text
appunti_laba03.txt
```

Contenuto di esempio:

```text
Questo è un file di test caricato nel blob storage del laboratorio LABA03.
```

### Opzione B - File creato in Cloud Shell

Se vuoi creare il file da terminale, esegui:

```bash
echo "Questo è un file di test caricato nel blob storage del laboratorio LABA03." > appunti_laba03.txt
```

### Spiegazione del comando

* `echo` stampa il testo indicato
* il simbolo `>` reindirizza l’output in un file
* `appunti_laba03.txt` è il nome del file da creare

### Verifica il file

Esegui:

```bash
ls -l appunti_laba03.txt
```

e poi:

```bash
cat appunti_laba03.txt
```

### Spiegazione

* `ls -l` mostra il file con dettagli
* `cat` visualizza il contenuto del file

### Evidenza richiesta

Copia l’output di `ls -l` e `cat` nel file delle evidenze.

---

## Step 9 - Carica il file nel container dal portale

Apri il container creato, ad esempio `docs`.

Clicca su **Upload** e seleziona il file:

```text
appunti_laba03.txt
```

Completa il caricamento.

### Che cosa hai fatto

Hai caricato un blob dentro il container.

### Che cosa devi osservare

Dopo il caricamento controlla:

* nome del file
* dimensione
* data di upload
* eventuali proprietà del blob

### Evidenza richiesta

Esegui uno screenshot del blob caricato nel container.

---

## Step 10 - Apri il blob e osservane le proprietà

Apri il file caricato.

Controlla, se disponibili:

* nome
* tipo
* URL
* dimensione
* data ultima modifica

### Cosa devi capire

Il blob è il singolo oggetto archiviato nel cloud.
Da questo momento esiste come dato salvato in Azure.

### Evidenza richiesta

Annota nel file evidenze il nome del blob e le principali proprietà visibili.

---

## Step 11 - Recupera la connection string dello Storage Account dal portale

Nel menu dello Storage Account cerca la sezione:

```text
Access keys
```

Visualizza le chiavi e individua la **connection string**.

### Importante

La connection string contiene informazioni sensibili di accesso.
In un laboratorio introduttivo può essere usata per capire come gli strumenti si collegano allo storage, ma non deve essere trattata con leggerezza in ambienti reali.

### Cosa fare

Copia temporaneamente la connection string della chiave principale e usala nel comando del passo successivo.

### Nota didattica importante

Nel file delle evidenze non incollare la connection string completa se il laboratorio viene condiviso con altri.
Puoi scrivere semplicemente che hai recuperato la connection string.

---

## Step 12 - Salva temporaneamente la connection string in una variabile

In Cloud Shell esegui un comando di questo tipo, sostituendo il valore reale:

```bash
export AZURE_STORAGE_CONNECTION_STRING="INCOLLA_QUI_LA_CONNECTION_STRING"
```

### Che cosa fa questo comando

* `export` crea una variabile d’ambiente
* `AZURE_STORAGE_CONNECTION_STRING` è il nome della variabile
* il valore assegnato è la connection string dello Storage Account

### Perché è utile

Molti comandi di Azure CLI relativi allo storage possono usare questa variabile per autenticarsi.

### Attenzione

Non condividere pubblicamente la connection string reale.

---

## Step 13 - Elenca i container con Azure CLI

Esegui:

```bash
az storage container list --output table
```

### Che cosa fa questo comando

* `storage` indica che lavori sui servizi di storage
* `container` specifica che vuoi operare sui container
* `list` richiede l’elenco dei container
* `--output table` mostra il risultato in formato tabellare

### Perché funziona senza riscrivere la connection string nel comando

Perché Azure CLI userà la variabile d’ambiente impostata prima.

### Che cosa devi osservare

Verifica che compaia il container creato dal portale, ad esempio:

```text
docs
```

### Evidenza richiesta

Copia l’output nel file delle evidenze.

---

## Step 14 - Crea un secondo container da Azure CLI

Esegui il comando:

```bash
az storage container create --name cli-container --output json
```

### Che cosa fa questo comando

* `create` crea un nuovo container
* `--name cli-container` assegna il nome `cli-container`
* `--output json` mostra il risultato in JSON

### Che cosa devi osservare

Controlla nell’output se l’operazione è andata a buon fine.

### Perché è utile

Stai ripetendo la stessa logica già vista nel portale, ma da linea di comando.

### Evidenza richiesta

Copia l’output del comando nel file delle evidenze.

---

## Step 15 - Elenca di nuovo i container

Esegui:

```bash
az storage container list --output table
```

### Che cosa devi osservare

Dovresti vedere almeno due container:

* quello creato dal portale, ad esempio `docs`
* quello creato da CLI, cioè `cli-container`

### Evidenza richiesta

Copia l’output nel file delle evidenze.

---

## Step 16 - Elenca i blob del container principale

Esegui il comando seguente, sostituendo `docs` con il nome del tuo container se diverso:

```bash
az storage blob list --container-name docs --output table
```

### Che cosa fa questo comando

* `blob` indica che stai lavorando con i file archiviati
* `list` chiede l’elenco dei blob
* `--container-name docs` limita la ricerca al container `docs`
* `--output table` mostra il risultato in tabella

### Che cosa devi osservare

Verifica che il file caricato, ad esempio `appunti_laba03.txt`, compaia nell’elenco.

### Evidenza richiesta

Copia l’output nel file delle evidenze.

---

## Step 17 - Verifica l’Activity Log

Nel portale Azure cerca:

```text
Activity Log
```

Apri il servizio.

### Che cosa osservare

Prova a individuare eventi legati a:

* creazione dello Storage Account
* operazioni amministrative sulla risorsa
* eventuali aggiornamenti

### Cosa devi capire

L’Activity Log registra operazioni amministrative sulla piattaforma Azure.

Non registra il contenuto del file caricato come farebbe un file system log locale.
Registra gli eventi di gestione e amministrazione della risorsa.

### Evidenza richiesta

Esegui uno screenshot dell’Activity Log con almeno un evento collegato allo Storage Account.

---

## Step 18 - Riepiloga mentalmente la gerarchia

A questo punto devi saper leggere correttamente questa struttura:

```text
Subscription
 └── Resource Group
      └── Storage Account
           └── Container
                └── Blob
```

Questa sequenza è molto importante.

Esempio concreto:

```text
Subscription: Azure for Students
Resource Group: rg-laba02-mario
Storage Account: stlaba03mario01
Container: docs
Blob: appunti_laba03.txt
```

---

# 3. Mini esercizio finale

Rispondi per iscritto alle seguenti domande.

## Domanda 1

Che differenza c’è tra Storage Account, container e blob?

## Domanda 2

Perché il nome di uno Storage Account deve essere univoco?

## Domanda 3

A cosa serve Blob Storage?

## Domanda 4

Perché in uno scenario reale è importante trattare con attenzione le access key e la connection string?

## Domanda 5

Perché è utile saper svolgere la stessa operazione sia dal portale sia da Azure CLI?

---

# 4. Output attesi

Al termine del laboratorio devi aver prodotto le seguenti evidenze:

1. screenshot della pagina **Storage accounts**
2. screenshot dello **Storage Account** creato
3. screenshot del **container** creato
4. screenshot del **blob** caricato
5. screenshot dell’**Activity Log**
6. output del comando:

```bash
az account show --output table
```

7. output del comando:

```bash
az group list --output table
```

8. output del comando:

```bash
ls -l appunti_laba03.txt
```

9. output del comando:

```bash
cat appunti_laba03.txt
```

10. output del comando:

```bash
az storage container list --output table
```

11. output del comando:

```bash
az storage container create --name cli-container --output json
```

12. output del comando:

```bash
az storage container list --output table
```

13. output del comando:

```bash
az storage blob list --container-name docs --output table
```

14. risposte scritte alle 5 domande finali

---

# 5. Modello di file evidenze

Crea un file chiamato:

```text
evidenze_laba03.md
```

e usa una struttura come questa:

```md
# Evidenze LABA03

## 1. Storage Account
- Nome Storage Account:
- Resource Group:
- Region:
- Ridondanza:

## 2. Container principale
- Nome container:

## 3. Blob caricato
- Nome blob:
- Note:

## 4. Output CLI

### az account show --output table
[incollare qui l’output]

### az group list --output table
[incollare qui l’output]

### ls -l appunti_laba03.txt
[incollare qui l’output]

### cat appunti_laba03.txt
[incollare qui l’output]

### az storage container list --output table
[incollare qui l’output]

### az storage container create --name cli-container --output json
[incollare qui l’output]

### az storage container list --output table
[incollare qui l’output]

### az storage blob list --container-name docs --output table
[incollare qui l’output]

## 5. Risposte finali

### Domanda 1
[risposta]

### Domanda 2
[risposta]

### Domanda 3
[risposta]

### Domanda 4
[risposta]

### Domanda 5
[risposta]
```

---

# 6. Errori comuni da evitare

## Errore 1 - Confondere Storage Account e container

Correzione:

* lo Storage Account è la risorsa Azure di alto livello
* il container è il contenitore logico interno
* il blob è il singolo file

---

## Errore 2 - Pensare che Blob Storage sia “una cartella locale”

Correzione:

Blob Storage è un servizio di archiviazione oggetti nel cloud, non una cartella locale del PC.

---

## Errore 3 - Scegliere un nome non valido o già usato

Correzione:

Usa lettere minuscole e numeri, senza spazi, e rendi il nome univoco.

Esempio:

```text
stlaba03mario01
```

---

## Errore 4 - Incollare pubblicamente chiavi e connection string

Correzione:

Le chiavi di accesso e le connection string sono dati sensibili.

---

## Errore 5 - Non capire la gerarchia delle risorse

Correzione:

Ripeti sempre mentalmente questa struttura:

```text
Subscription
 └── Resource Group
      └── Storage Account
           └── Container
                └── Blob
```

---

# 7. Conclusione

Questo laboratorio ti ha introdotto ai concetti fondamentali dell’archiviazione su Azure.

Dopo aver completato il lab, devi avere chiaro che:

* lo Storage Account è la risorsa principale di archiviazione
* Blob Storage serve a conservare oggetti e file
* i file vengono organizzati in container
* il singolo file archiviato è un blob
* il portale e Azure CLI permettono di lavorare sugli stessi oggetti con modalità diverse
* le informazioni di accesso allo storage vanno trattate con attenzione

Queste basi saranno molto utili nei laboratori successivi, dove entreranno in gioco macchine virtuali, deployment applicativo e servizi più avanzati.

```
```
