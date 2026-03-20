# LABA01 - Azure Foundations: identità, tenant, subscription, portale e Cloud Shell

## Obiettivo del laboratorio

In questo laboratorio si imparano i concetti fondamentali necessari per iniziare a lavorare su Azure in modo consapevole.

Prima di creare macchine virtuali, database, container o storage account, è necessario capire:

- con quale identità si sta accedendo
- in quale tenant si sta operando
- quale subscription è attiva
- quali strumenti di base mette a disposizione Azure
- dove verificare le attività amministrative svolte sulla piattaforma

Al termine del laboratorio sarai in grado di:

- distinguere tenant, subscription, resource group e resource
- riconoscere la tua identità nel portale Azure
- individuare la subscription attiva
- usare Azure Cloud Shell
- eseguire i primi comandi con Azure CLI
- consultare l’Activity Log
- raccogliere le evidenze richieste

---

## Scenario

Stai per iniziare un percorso di laboratori Azure. Prima di creare risorse, devi imparare a orientarti nella struttura della piattaforma.

In Azure non si lavora in modo generico “nel cloud”. Ogni attività avviene all’interno di una struttura organizzata.

Gli elementi principali sono:

- **Tenant**
- **Subscription**
- **Resource Group**
- **Resource**

Questi concetti devono essere chiari fin dall’inizio, perché torneranno in tutti i laboratori successivi.

---

## Prerequisiti

Per svolgere il laboratorio occorrono:

- un account Azure funzionante
- accesso al portale Azure
- un browser web aggiornato
- connessione Internet stabile

Non è richiesta esperienza precedente su Azure.

---

## Durata indicativa

**1 ora e 30 minuti - 2 ore**

---

## Competenze in uscita

Al termine del laboratorio sarai in grado di:

- accedere al portale Azure
- individuare tenant e subscription correnti
- usare Cloud Shell per eseguire comandi base
- comprendere il legame tra identità e risorse
- verificare eventi amministrativi nell’Activity Log

---

# 1. Concetti fondamentali

## 1.1 Cos’è un tenant

Il **tenant** è il contenitore identitario dell’organizzazione in Azure.

Oggi il servizio di identità si chiama **Microsoft Entra ID**.

Nel tenant sono contenuti, per esempio:

- utenti
- gruppi
- ruoli
- applicazioni registrate
- configurazioni di accesso

Il tenant risponde alla domanda:

**Chi sei e in quale directory stai lavorando?**

---

## 1.2 Cos’è una subscription

La **subscription** è il contenitore amministrativo ed economico in cui vengono create e gestite le risorse Azure.

La subscription serve a definire:

- il contesto amministrativo
- il contesto di fatturazione
- il perimetro entro cui vengono create molte risorse

La subscription risponde alla domanda:

**Dove stai creando le risorse e a quale contesto amministrativo appartengono?**

---

## 1.3 Cos’è un resource group

Il **resource group** è un contenitore logico di risorse Azure.

Serve per organizzare insieme le risorse appartenenti allo stesso progetto, laboratorio o applicazione.

Per esempio, nello stesso resource group possono stare:

- una VM
- uno storage account
- un database
- un workspace di monitoraggio

Il resource group risponde alla domanda:

**Quali risorse appartengono allo stesso progetto o laboratorio?**

---

## 1.4 Cos’è una resource

Una **resource** è il singolo oggetto creato in Azure.

Esempi di resource:

- una macchina virtuale
- uno storage account
- un database SQL
- un container instance
- un network security group

La resource è l’elemento concreto su cui si lavora.

---

## 1.5 Struttura logica di Azure

La struttura logica di base può essere letta così:

- **Tenant**
  - contiene identità e accessi
- **Subscription**
  - contiene il contesto amministrativo delle risorse
- **Resource Group**
  - organizza logicamente le risorse
- **Resource**
  - è il singolo servizio o oggetto creato

In forma semplice:

```text
Tenant
 └── Subscription
      └── Resource Group
           └── Resources
````

---

# 2. Attività pratiche

## Step 1 - Accedi al portale Azure

Apri il portale Azure nel browser:

```text
https://portal.azure.com
```

Effettua l’accesso con il tuo account.

### Cosa osservare

Una volta entrato nel portale, osserva con attenzione:

* l’account utente in alto a destra
* la barra di ricerca in alto
* il menu dei servizi
* la presenza dei servizi:

  * Microsoft Entra ID
  * Subscriptions
  * Resource groups
  * Cloud Shell

### Obiettivo dello step

L’obiettivo è imparare a riconoscere i punti di riferimento principali del portale.

---

## Step 2 - Verifica l’identità con cui sei connesso

Nel portale, clicca sull’icona del tuo account in alto a destra.

Annota le informazioni visibili, ad esempio:

* nome visualizzato
* email o UPN
* eventuale directory selezionata

### Cosa devi capire

In Azure non sei semplicemente “un utente loggato”. Sei una identità che appartiene a un tenant.

---

## Step 3 - Apri Microsoft Entra ID

Nella barra di ricerca del portale digita:

```text
Microsoft Entra ID
```

Apri il servizio.

### Verifica queste informazioni

Nella pagina principale individua:

* nome del tenant
* Tenant ID
* dominio principale, se visibile

### Cosa significa

Stai osservando il contenitore identitario dell’ambiente Azure in cui lavori.

Qui vengono gestite identità e accessi, non le risorse applicative.

### Evidenza richiesta

Esegui uno screenshot della pagina principale di Microsoft Entra ID in cui siano visibili:

* nome tenant
* Tenant ID
* dominio

---

## Step 4 - Apri la pagina delle subscription

Nella barra di ricerca del portale digita:

```text
Subscriptions
```

Apri il servizio.

### Verifica queste informazioni

Per la subscription che userai nei laboratori, individua:

* nome subscription
* stato
* subscription ID

### Cosa devi capire

La subscription rappresenta il contenitore amministrativo entro cui verranno create molte delle risorse usate nei laboratori.

### Evidenza richiesta

Esegui uno screenshot della pagina Subscriptions in cui si veda chiaramente:

* nome subscription
* stato
* subscription ID

---

## Step 5 - Apri Cloud Shell

Nel portale clicca sull’icona di **Cloud Shell**.

Se è il primo utilizzo, completa la procedura iniziale richiesta dal portale.

Quando richiesto, seleziona:

```text
Bash
```

### Cos’è Cloud Shell

Cloud Shell è un terminale disponibile direttamente nel browser.

Permette di usare strumenti come:

* Bash
* Azure CLI
* comandi Linux di base

### Perché è utile

Cloud Shell permette di lavorare senza installare subito tutti gli strumenti sul computer locale.

---

## Step 6 - Verifica l’account attivo da Azure CLI

Nel terminale di Cloud Shell esegui il comando:

```bash
az account show --output table
```

### Cosa fa questo comando

Mostra le informazioni dell’account Azure attualmente in uso e della subscription attiva.

### Cosa osservare

Verifica la presenza di dati come:

* nome subscription
* stato
* tenant ID
* utente
* indicazione della subscription di default

### Evidenza richiesta

Copia l’output del comando nel file delle evidenze.

---

## Step 7 - Elenca le subscription disponibili

Esegui il comando:

```bash
az account list --output table
```

### Cosa fa questo comando

Mostra tutte le subscription accessibili con il tuo account.

### Cosa osservare

Controlla:

* se esiste una sola subscription o più subscription
* quale risulta predefinita
* quali sono attive

### Nota importante

Se in futuro dovrai lavorare con più subscription, sarà importante verificare sempre quale è attiva prima di creare risorse.

### Evidenza richiesta

Copia l’output del comando nel file delle evidenze.

---

## Step 8 - Visualizza le informazioni essenziali in formato JSON

Esegui il comando:

```bash
az account show --query "{subscriptionName:name, subscriptionId:id, tenantId:tenantId, user:user.name}" --output json
```

### Cosa fa questo comando

Mostra un riepilogo essenziale dell’account in formato JSON con:

* nome subscription
* ID subscription
* tenant ID
* nome utente

### Perché è utile

Questo tipo di output è più strutturato e sarà utile anche nei laboratori successivi.

### Evidenza richiesta

Copia l’output JSON nel file delle evidenze.

---

## Step 9 - Apri l’Activity Log

Nel portale Azure cerca:

```text
Activity Log
```

Apri il servizio.

### Cosa osservare

Controlla se sono presenti eventi recenti e osserva colonne o campi come:

* timestamp
* operazione
* stato
* subscription
* risorsa coinvolta, se presente

### Cosa devi capire

L’Activity Log registra le attività amministrative svolte sulla piattaforma Azure.

Non rappresenta il log interno dell’applicazione.

### Distinzione importante

* **Activity Log** = eventi amministrativi sulla piattaforma Azure
* **Application Log** = eventi generati da un’applicazione
* **Metrics** = misure numeriche nel tempo, come CPU, memoria, richieste, errori

### Evidenza richiesta

Esegui uno screenshot della schermata dell’Activity Log.

---

## Step 10 - Comprendi la gerarchia con un esempio

Considera il seguente esempio:

* Tenant: `academy.local`
* Subscription: `Azure for Students`
* Resource Group: `rg-lab-observability`
* Resource: `vm-lab01`

### Interpretazione

* il tenant contiene identità e accessi
* la subscription definisce il contesto amministrativo
* il resource group organizza le risorse del progetto
* la VM è una resource concreta

Questa struttura va memorizzata, perché sarà la base di tutti i laboratori futuri.

---

# 3. Mini esercizio finale

Rispondi per iscritto alle seguenti domande.

## Domanda 1

Qual è la differenza principale tra tenant e subscription?

## Domanda 2

Dove viene creata normalmente una macchina virtuale: nel tenant, nella subscription o nel resource group?

## Domanda 3

L’utente con cui accedi al portale appartiene a quale componente della struttura Azure?

## Domanda 4

L’Activity Log registra il codice dell’applicazione oppure le operazioni amministrative sulla piattaforma Azure?

## Domanda 5

Cosa può accadere alle risorse contenute in un resource group se il resource group viene eliminato?

---

# 4. Output attesi

Al termine del laboratorio devi aver prodotto le seguenti evidenze:

1. screenshot della pagina principale di **Microsoft Entra ID**
2. screenshot della pagina **Subscriptions**
3. screenshot della pagina **Activity Log**
4. output del comando:

```bash
az account show --output table
```

5. output del comando:

```bash
az account list --output table
```

6. output del comando:

```bash
az account show --query "{subscriptionName:name, subscriptionId:id, tenantId:tenantId, user:user.name}" --output json
```

7. risposte scritte alle 5 domande finali

---

# 5. Modello di file evidenze

Crea un file chiamato:

```text
evidenze_laba01.md
```

e inserisci una struttura come la seguente:

```md
# Evidenze LABA01

## 1. Tenant e identità
- Nome tenant:
- Tenant ID:
- Utente:

## 2. Subscription
- Nome subscription:
- Subscription ID:
- Stato:

## 3. Output CLI

### az account show --output table
[incollare qui l’output]

### az account list --output table
[incollare qui l’output]

### az account show --query "{subscriptionName:name, subscriptionId:id, tenantId:tenantId, user:user.name}" --output json
[incollare qui l’output]

## 4. Risposte finali

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

## Errore 1 - Confondere tenant e subscription

Correzione:

* **tenant** = identità, utenti, accessi
* **subscription** = contenitore amministrativo delle risorse

---

## Errore 2 - Pensare che il resource group sia al di sopra della subscription

Correzione:

Il resource group esiste all’interno di una subscription.

---

## Errore 3 - Confondere Activity Log e log applicativi

Correzione:

L’Activity Log registra le operazioni amministrative sulla piattaforma Azure, non i dettagli interni dell’applicazione.

---

## Errore 4 - Eseguire comandi CLI senza controllare la subscription attiva

Correzione:

Prima di creare risorse, controlla sempre il contesto attivo con:

```bash
az account show --output table
```

---

# 7. Conclusione

Questo laboratorio ha lo scopo di costruire le basi operative minime per lavorare su Azure.

Dopo aver completato questo lab, devi avere chiaro:

* chi sei all’interno di Azure
* in quale tenant stai lavorando
* quale subscription stai usando
* come aprire e usare Cloud Shell
* dove consultare le attività amministrative della piattaforma

Queste conoscenze sono fondamentali per affrontare i laboratori successivi dedicati a resource group, storage, macchine virtuali e deployment applicativo.

