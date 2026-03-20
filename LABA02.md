````md
# LABA02 - Azure Foundations: Resource Group, tagging e ciclo di vita delle risorse

## Obiettivo del laboratorio

In questo laboratorio imparerai a usare uno degli elementi più importanti di Azure: il **Resource Group**.

Prima di creare storage account, macchine virtuali, database o container, è fondamentale capire:

- cos’è un Resource Group
- perché si usa
- come si crea
- come si organizza con una naming convention
- come si applicano i tag
- cosa significa ciclo di vita delle risorse
- cosa succede quando un Resource Group viene eliminato

Al termine del laboratorio sarai in grado di:

- creare un Resource Group dal portale e da Azure CLI
- applicare tag alle risorse logiche
- interpretare il Resource Group come contenitore di progetto
- verificare le operazioni nell’Activity Log
- comprendere il legame tra organizzazione tecnica, governance e costi

---

## Scenario

In Azure le risorse non vengono create “sparse nel cloud”.  
Devono essere organizzate dentro un contenitore logico chiamato **Resource Group**.

Un Resource Group serve a raccogliere insieme le risorse che appartengono allo stesso contesto, ad esempio:

- un laboratorio
- un ambiente di test
- una piccola applicazione
- un progetto temporaneo
- un ambiente di esercitazione

In questo laboratorio creerai alcuni Resource Group di esempio, applicherai tag e osserverai come Azure registra queste attività.

---

## Prerequisiti

Per svolgere il laboratorio occorrono:

- account Azure funzionante
- accesso al portale Azure
- accesso a Cloud Shell
- completamento del laboratorio LABA01
- connessione Internet stabile

---

## Durata indicativa

**1 ora e 30 minuti - 2 ore**

---

## Competenze in uscita

Al termine del laboratorio sarai in grado di:

- spiegare il ruolo del Resource Group
- creare e visualizzare Resource Group
- usare una naming convention semplice
- applicare tag utili alla classificazione delle risorse
- verificare il ciclo di vita di un Resource Group
- leggere le operazioni amministrative nell’Activity Log

---

# 1. Concetti fondamentali

## 1.1 Cos’è un Resource Group

Un **Resource Group** è un contenitore logico di risorse Azure.

Dentro un Resource Group possono essere collocate risorse correlate, per esempio:

- una VM
- uno storage account
- un database
- una rete virtuale
- un workspace di monitoraggio

Il Resource Group non è una risorsa applicativa finale.  
Non esegue codice, non salva file, non elabora dati.  
Serve a **organizzare** e **gestire** le risorse.

---

## 1.2 Perché si usa un Resource Group

Il Resource Group serve a:

- raggruppare risorse appartenenti allo stesso progetto
- semplificare la gestione amministrativa
- facilitare la cancellazione di ambienti di test o lab
- applicare convenzioni organizzative
- semplificare il monitoraggio e la governance

In pratica, se un laboratorio prevede più risorse, conviene raccoglierle nello stesso Resource Group.

---

## 1.3 Relazione con la subscription

Un Resource Group esiste **all’interno di una subscription**.

Questo significa che:

- prima esiste la subscription
- poi dentro la subscription si crea il Resource Group
- poi dentro il Resource Group si creano le singole risorse

La struttura è quindi:

```text
Tenant
 └── Subscription
      └── Resource Group
           └── Resources
````

---

## 1.4 Il Resource Group non è una cartella

Un errore molto comune è immaginare il Resource Group come una semplice cartella grafica.

È meglio considerarlo come un contenitore amministrativo e logico.

Serve a:

* vedere insieme le risorse correlate
* gestire il loro ciclo di vita
* semplificare la pulizia dell’ambiente
* applicare tag e convenzioni

---

## 1.5 Naming convention

Quando si lavora bene su Azure, si usano nomi coerenti e leggibili.

Una naming convention semplice aiuta a capire subito:

* a cosa serve la risorsa
* in quale ambiente si trova
* a quale progetto appartiene

Esempio per un Resource Group:

```text
rg-laba02-nomecognome
```

Oppure:

```text
rg-obs-dev
```

In questo laboratorio userai nomi semplici, chiari e leggibili.

---

## 1.6 Cosa sono i tag

I **tag** sono coppie chiave-valore applicate alle risorse Azure.

Esempi:

* `env=lab`
* `owner=mario.rossi`
* `module=azure-foundations`
* `purpose=training`

I tag servono a:

* classificare le risorse
* filtrare e cercare più facilmente
* supportare governance e controllo costi
* capire chi ha creato cosa

Un tag non cambia il funzionamento tecnico della risorsa, ma la rende più organizzata e leggibile.

---

## 1.7 Ciclo di vita delle risorse

Il ciclo di vita di una risorsa o di un Resource Group può essere visto così:

1. creazione
2. utilizzo
3. modifica
4. controllo
5. eliminazione

Nei laboratori Azure è molto importante anche l’ultima fase: **cleanup**.

Se si dimenticano risorse attive, si rischia di:

* creare confusione
* lasciare ambienti inutili
* consumare budget o quote disponibili

---

# 2. Attività pratiche

## Step 1 - Verifica la subscription attiva

Apri Cloud Shell in modalità Bash ed esegui:

```bash
az account show --output table
```

### Cosa fare

Controlla che la subscription attiva sia quella corretta per i laboratori.

### Cosa osservare

Verifica almeno questi elementi:

* Name
* State
* TenantId
* IsDefault

### Evidenza richiesta

Copia l’output nel file delle evidenze.

---

## Step 2 - Apri la pagina Resource Groups nel portale

Nel portale Azure cerca:

```text
Resource groups
```

Apri il servizio.

### Cosa osservare

Controlla se esistono già Resource Group nella subscription.

Osserva la schermata e individua:

* elenco dei Resource Group
* subscription associata
* region
* numero di risorse, se visibile

### Obiettivo dello step

Imparare a riconoscere il punto del portale in cui vengono organizzate molte risorse del progetto.

### Evidenza richiesta

Esegui uno screenshot della pagina Resource groups.

---

## Step 3 - Crea un Resource Group dal portale

Dalla pagina **Resource groups**, clicca su **Create**.

Compila i campi richiesti con una convenzione simile a questa:

* **Subscription**: seleziona la subscription del laboratorio
* **Resource group name**: `rg-laba02-<tuonome>`
* **Region**: seleziona una regione disponibile, ad esempio `West Europe` o quella indicata dal docente

Esempio:

```text
rg-laba02-mario
```

Procedi fino alla creazione.

### Cosa devi capire

Il Resource Group è il contenitore che userai per raccogliere le risorse del laboratorio.

### Evidenza richiesta

Esegui uno screenshot del Resource Group appena creato.

---

## Step 4 - Osserva i dettagli del Resource Group

Apri il Resource Group creato.

Osserva con attenzione:

* nome
* subscription
* region
* eventuali tag
* pannello Overview
* menu laterale del Resource Group

### Cosa devi capire

Questo contenitore in questo momento può anche essere vuoto.
Il suo ruolo è organizzativo e amministrativo.

### Evidenza richiesta

Annota nel file evidenze:

* nome del Resource Group
* subscription
* region

---

## Step 5 - Crea un secondo Resource Group di test da Azure CLI

Apri Cloud Shell ed esegui il comando seguente, sostituendo `<tuonome>` con il tuo nome o cognome:

```bash
az group create --name rg-laba02-test-<tuonome> --location westeurope
```

Esempio:

```bash
az group create --name rg-laba02-test-mario --location westeurope
```

### Cosa fa questo comando

Crea un nuovo Resource Group nella regione indicata.

### Cosa osservare

L’output viene mostrato in formato JSON e contiene informazioni come:

* id
* location
* name
* properties
* provisioningState
* tags

### Evidenza richiesta

Copia l’output del comando nel file delle evidenze.

---

## Step 6 - Elenca i Resource Group disponibili

Esegui:

```bash
az group list --output table
```

### Cosa fa questo comando

Mostra tutti i Resource Group disponibili nella subscription attiva.

### Cosa osservare

Controlla che siano visibili almeno i due Resource Group creati:

* uno creato dal portale
* uno creato da CLI

### Evidenza richiesta

Copia l’output del comando nel file delle evidenze.

---

## Step 7 - Visualizza i dettagli di un Resource Group specifico

Esegui il comando seguente, sostituendo il nome corretto del tuo Resource Group:

```bash
az group show --name rg-laba02-<tuonome> --output json
```

Esempio:

```bash
az group show --name rg-laba02-mario --output json
```

### Cosa fa questo comando

Mostra il dettaglio completo di un singolo Resource Group.

### Cosa osservare

Individua almeno questi campi:

* `id`
* `location`
* `managedBy`
* `name`
* `properties.provisioningState`
* `tags`

### Evidenza richiesta

Copia l’output JSON nel file delle evidenze.

---

## Step 8 - Aggiungi tag al Resource Group dal portale

Apri dal portale il Resource Group principale, ad esempio:

```text
rg-laba02-<tuonome>
```

Cerca la sezione **Tags** e aggiungi almeno questi tag:

* `env = lab`
* `owner = <tuonome>`
* `module = azure-foundations`

Esempio:

* `env = lab`
* `owner = mario`
* `module = azure-foundations`

Salva le modifiche.

### Cosa devi capire

I tag non cambiano il comportamento tecnico del Resource Group, ma sono molto utili per classificare e organizzare le risorse.

### Evidenza richiesta

Esegui uno screenshot dei tag salvati.

---

## Step 9 - Aggiungi tag al secondo Resource Group da CLI

Esegui il comando seguente, sostituendo il nome del tuo Resource Group di test:

```bash
az group update --name rg-laba02-test-<tuonome> --set tags.env=lab tags.owner=<tuonome> tags.purpose=test
```

Esempio:

```bash
az group update --name rg-laba02-test-mario --set tags.env=lab tags.owner=mario tags.purpose=test
```

### Cosa fa questo comando

Aggiorna il Resource Group aggiungendo i tag indicati.

### Cosa osservare

Nell’output JSON verifica la presenza della sezione `tags`.

### Evidenza richiesta

Copia l’output del comando nel file delle evidenze.

---

## Step 10 - Verifica i tag da CLI

Esegui:

```bash
az group show --name rg-laba02-test-<tuonome> --query "{name:name, location:location, tags:tags}" --output json
```

Esempio:

```bash
az group show --name rg-laba02-test-mario --query "{name:name, location:location, tags:tags}" --output json
```

### Cosa fa questo comando

Mostra un riepilogo semplice del Resource Group con nome, location e tag.

### Evidenza richiesta

Copia l’output nel file delle evidenze.

---

## Step 11 - Verifica nell’Activity Log le operazioni eseguite

Nel portale cerca:

```text
Activity Log
```

Apri il servizio.

Filtra, se necessario, gli eventi recenti relativi alla tua subscription.

### Cosa osservare

Prova a individuare eventi legati a:

* creazione del Resource Group
* aggiornamento del Resource Group
* modifica dei tag

### Cosa devi capire

L’Activity Log registra operazioni amministrative svolte su Azure.

In questo laboratorio puoi verificare che le azioni di creazione e aggiornamento siano state effettivamente registrate.

### Evidenza richiesta

Esegui uno screenshot dell’Activity Log con almeno un evento relativo ai Resource Group.

---

## Step 12 - Comprendi il ciclo di vita con un esempio concreto

Considera questo scenario:

* `rg-laba02-mario` contiene le risorse di un laboratorio
* `rg-laba02-test-mario` contiene un ambiente temporaneo di prova

Se il laboratorio è terminato e le risorse non servono più, il Resource Group può essere eliminato per rimuovere l’ambiente.

### Cosa devi capire

La gestione del ciclo di vita è fondamentale per:

* evitare ambienti inutili
* mantenere ordine
* contenere il consumo di risorse
* lavorare in modo professionale

---

## Step 13 - Elimina il Resource Group di test da CLI

Elimina solo il Resource Group di test, non quello principale.

Esegui il comando seguente, sostituendo il nome corretto:

```bash
az group delete --name rg-laba02-test-<tuonome> --yes --no-wait
```

Esempio:

```bash
az group delete --name rg-laba02-test-mario --yes --no-wait
```

### Cosa fa questo comando

Avvia l’eliminazione del Resource Group specificato senza chiedere conferma interattiva.

### Attenzione

Eliminando un Resource Group, vengono eliminate anche le risorse contenute al suo interno.

In questo laboratorio il Resource Group di test è stato creato appositamente per comprendere il ciclo di vita.

### Evidenza richiesta

Copia il comando eseguito nel file delle evidenze e annota che l’eliminazione è stata avviata.

---

## Step 14 - Verifica che il Resource Group di test non sia più disponibile

Attendi qualche istante e poi esegui:

```bash
az group list --output table
```

### Cosa osservare

Verifica se il Resource Group di test è stato rimosso dall’elenco.

Se l’eliminazione è ancora in corso, ripeti il comando dopo un breve intervallo.

### Evidenza richiesta

Copia il nuovo output nel file delle evidenze.

---

# 3. Mini esercizio finale

Rispondi per iscritto alle seguenti domande.

## Domanda 1

Perché in Azure è utile usare i Resource Group invece di creare risorse senza una struttura organizzata?

## Domanda 2

Qual è la differenza tra Resource Group e singola resource?

## Domanda 3

A cosa servono i tag?

## Domanda 4

Che cosa può succedere quando si elimina un Resource Group?

## Domanda 5

Perché il ciclo di vita e il cleanup sono importanti nei laboratori Azure?

---

# 4. Output attesi

Al termine del laboratorio devi aver prodotto le seguenti evidenze:

1. screenshot della pagina **Resource groups**
2. screenshot del **Resource Group principale** creato dal portale
3. screenshot dei **tag** applicati dal portale
4. screenshot dell’**Activity Log**
5. output del comando:

```bash
az account show --output table
```

6. output del comando:

```bash
az group create --name rg-laba02-test-<tuonome> --location westeurope
```

7. output del comando:

```bash
az group list --output table
```

8. output del comando:

```bash
az group show --name rg-laba02-<tuonome> --output json
```

9. output del comando:

```bash
az group update --name rg-laba02-test-<tuonome> --set tags.env=lab tags.owner=<tuonome> tags.purpose=test
```

10. output del comando:

```bash
az group show --name rg-laba02-test-<tuonome> --query "{name:name, location:location, tags:tags}" --output json
```

11. comando di eliminazione del Resource Group di test:

```bash
az group delete --name rg-laba02-test-<tuonome> --yes --no-wait
```

12. output finale del comando:

```bash
az group list --output table
```

13. risposte scritte alle 5 domande finali

---

# 5. Modello di file evidenze

Crea un file chiamato:

```text
evidenze_laba02.md
```

e inserisci una struttura come la seguente:

```md
# Evidenze LABA02

## 1. Resource Group principale
- Nome:
- Subscription:
- Region:

## 2. Output CLI

### az account show --output table
[incollare qui l’output]

### az group create --name rg-laba02-test-<tuonome> --location westeurope
[incollare qui l’output]

### az group list --output table
[incollare qui l’output]

### az group show --name rg-laba02-<tuonome> --output json
[incollare qui l’output]

### az group update --name rg-laba02-test-<tuonome> --set tags.env=lab tags.owner=<tuonome> tags.purpose=test
[incollare qui l’output]

### az group show --name rg-laba02-test-<tuonome> --query "{name:name, location:location, tags:tags}" --output json
[incollare qui l’output]

### az group delete --name rg-laba02-test-<tuonome> --yes --no-wait
[annotare il comando eseguito]

### az group list --output table
[incollare qui l’output finale]

## 3. Risposte finali

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

## Errore 1 - Pensare che il Resource Group sia una semplice cartella

Correzione:

Il Resource Group è un contenitore logico e amministrativo, non una semplice cartella grafica.

---

## Errore 2 - Dimenticare la subscription attiva

Correzione:

Prima di creare risorse, controlla sempre il contesto attivo con:

```bash
az account show --output table
```

---

## Errore 3 - Usare nomi poco chiari

Correzione:

Usa nomi leggibili e coerenti, ad esempio:

```text
rg-laba02-mario
rg-laba02-test-mario
```

---

## Errore 4 - Non applicare tag

Correzione:

I tag aiutano a classificare, filtrare e gestire le risorse in modo più ordinato.

---

## Errore 5 - Eliminare il Resource Group sbagliato

Correzione:

Prima di eseguire una cancellazione, verifica sempre con attenzione il nome del Resource Group.

---

# 7. Conclusione

Questo laboratorio ha introdotto uno dei concetti più importanti di Azure: il **Resource Group**.

Dopo aver completato questo lab, devi avere chiaro che:

* le risorse vanno organizzate in modo logico
* il Resource Group è il contenitore base di un progetto o laboratorio
* i tag aiutano a classificare e governare le risorse
* il ciclo di vita delle risorse include anche la fase di cleanup
* l’Activity Log permette di verificare le operazioni amministrative svolte

Queste conoscenze sono fondamentali per affrontare i laboratori successivi dedicati a storage account, VM e deployment applicativo.

```
```
