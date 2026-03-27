# LAB09 - ACR + ACI: deploy del container osservabile in Azure

## Obiettivo del laboratorio

In questo laboratorio entri nel primo scenario davvero centrale del Modulo 2: **pubblicare un’applicazione containerizzata su Azure e iniziare a osservarne il comportamento runtime**.

L’obiettivo è:

- creare un **Azure Container Registry (ACR)** in SKU Basic
- eseguire il **push** di un’immagine Docker nel registry
- fare il deploy del container su **Azure Container Instances (ACI)** con IP pubblico
- verificare endpoint e log, compresi **request-id**, **status** e **durata**

Questo è il primo laboratorio in cui l’osservabilità smette di essere solo “osservazione della piattaforma Azure” e inizia a diventare anche **osservazione dell’applicazione che gira su Azure**.

---

## Nota importante per chi usa una trial subscription

Se stai usando una **subscription trial**, in questo laboratorio devi mantenere il flusso operativo **interamente in WSL Ubuntu locale** per tutti i comandi Azure CLI e Docker.

Non usare Azure Cloud Shell per questo LAB.

Il motivo è semplice: con le trial subscriptions e con Cloud Shell possono comparire problemi non uniformi tra partecipanti, soprattutto su:

- autenticazione verso ACR
- recupero dei token
- passaggi successivi alla build
- comandi di verifica e interrogazione del registry

Per evitare differenze inutili tra partecipanti, in questo laboratorio userai:

- **WSL Ubuntu locale** per tutti i comandi Azure CLI e Docker
- **Azure Portal** solo per controlli visivi

---

## Durata indicativa

**4 ore**

---

## Cost control - gestione dei costi

Anche in questo laboratorio valgono le regole di contenimento costi del modulo:

- usare SKU minimi
- evitare tier Premium
- usare **ACR Basic**
- non creare risorse più grandi del necessario

### Modalità percorso continuativo

Non eliminare il Resource Group fino al LAB14.

### Modalità standalone

Solo se esegui il lab in isolamento, puoi fare cleanup finale del RG con:

```bash
az group delete --name rg-observability-lab --yes --no-wait
```

Questo comando è facoltativo in modalità standalone.

---

# PARTE 1 - Concetti fondamentali, architettura e focus Observability

# 1. Perché questo laboratorio è importante

Nel LAB08 hai costruito il contenitore amministrativo e il perimetro operativo del modulo.

Nel LAB09, finalmente, inizi a mettere **un carico applicativo osservabile** dentro quell’ambiente.

Questo laboratorio è importante perché introduce un flusso reale molto comune:

```text
codice applicativo
  ↓
immagine Docker
  ↓
registry container
  ↓
runtime container su cloud
  ↓
endpoint pubblico
  ↓
richieste HTTP
  ↓
log applicativi osservabili
```

In pratica:

- costruisci un’immagine in locale
- la pubblichi in ACR
- la esegui su ACI
- testi gli endpoint
- osservi i log applicativi

---

# 2. ACR, ACI e app osservabile: che cosa stai facendo davvero

## 2.1 Che cos’è ACR

**Azure Container Registry (ACR)** è il registry privato Azure per immagini container.

Serve a conservare immagini Docker in modo controllato e accessibile dalle risorse Azure.

Nel laboratorio, ACR serve come punto di passaggio tra:

- build locale dell’immagine
- deploy cloud della stessa immagine

In forma semplice:

- in locale costruisci l’immagine
- in ACR la pubblichi
- da ACI la esegui

---

## 2.2 Che cos’è ACI

**Azure Container Instances (ACI)** è un servizio Azure che permette di eseguire un container senza dover gestire una VM o un orchestratore Kubernetes.

È utile didatticamente perché:

- è più semplice di AKS
- ti fa concentrare sul runtime del container
- ti permette di ottenere un endpoint pubblico rapidamente
- è perfetto per introdurre l’osservabilità applicativa a runtime

Nel laboratorio userai ACI con:

- IP pubblico
- porta `8000`
- immagine `obsapp:v2`
- variabile ambiente `PORT=8000`

---

## 2.3 Perché non fare tutto direttamente da Docker locale

Perché il punto del laboratorio non è solo “vedere se il container parte”.

Quello lo avevi già affrontato altrove in locale.

Qui il punto è capire il flusso cloud:

- immagine locale
- registry remoto
- deploy gestito su Azure
- endpoint pubblico
- osservazione dell’app da fuori e da dentro

È questo il salto di qualità del modulo.

---

# 3. Che cosa vuol dire “container osservabile”

In questo laboratorio devi verificare:

- endpoint
- log
- `request-id`
- `status`
- `duration`

Questo significa che il container non è solo “un processo che risponde”.

È un servizio che **emette segnali osservabili**.

## 3.1 Segnali applicativi che cercherai

Nel LAB09 cercherai almeno questi segnali:

- esistenza dell’endpoint HTTP
- codici di risposta, per esempio `200` e `404`
- header `X-Request-Id`
- log container contenenti campi come:

  - `request_id`
  - `status`
  - `duration_ms`

---

## 3.2 Perché il request-id è importante

Il `request_id` o correlation id è un identificatore della singola richiesta.

Serve a:

- distinguere una richiesta dall’altra
- correlare client, log e possibili eventi successivi
- rendere il debugging molto più preciso

In forma semplice:

> senza identificatori di correlazione, i log sono rumore.  
> con identificatori di correlazione, i log diventano tracce utili.

---

## 3.3 Perché status e duration contano

Due campi chiave dei log applicativi sono:

- **status**: indica se la richiesta è andata bene o male
- **duration_ms**: indica quanto tempo ha impiegato

Questi campi sono importanti perché saranno la base dei laboratori successivi:

- LAB10: SLI via SQL
- LAB11: SLI via KQL
- LAB12: alerting
- LAB13: dashboard/workbook

In altre parole, qui non stai solo leggendo log: stai preparando i dati osservabili che userai per analisi e monitoraggio successivi.

---

# 4. Pipeline mentale del laboratorio

Il LAB09 va immaginato così:

```text
WSL Ubuntu
  ↓
docker build obsapp:v2
  ↓
Azure Container Registry (ACR)
  ↓
docker push
  ↓
Azure Container Instances (ACI)
  ↓
IP pubblico + porta 8000
  ↓
curl /health, /time, /nope
  ↓
header X-Request-Id
  ↓
log applicativi nel container
```

---

# 5. Differenza tra ciò che osservi nel portale e ciò che osservi nei log

Questo punto va chiarito bene.

## 5.1 Portale Azure / stato risorsa

Dal portale o da `az container show` osservi:

- esistenza della risorsa ACI
- stato runtime, per esempio `Running`
- IP pubblico
- configurazione generale

Questa è osservabilità **del runtime Azure della risorsa**.

---

## 5.2 curl e risposta HTTP

Con `curl` osservi:

- raggiungibilità dell’app
- status code HTTP
- header di risposta
- comportamento esterno del servizio

Questa è osservabilità **dal punto di vista del client**.

---

## 5.3 log del container

Con `az container logs` osservi:

- segnali emessi dall’app
- request_id
- path
- status
- durata

Questa è osservabilità **interna applicativa**.

Capire questi tre livelli è molto più importante del singolo comando.

---

# 6. Perché testare `/health`, `/time` e `/nope`

Userai tre endpoint specifici:

- `/health`
- `/time`
- `/nope`

Questa scelta non è casuale.

## `/health`

Serve tipicamente per verificare che il servizio sia vivo.

## `/time`

Serve per vedere una risposta funzionale valida, diversa dalla health.

## `/nope`

Serve per generare un errore applicativo controllato, tipicamente `404`.

Con questi tre endpoint ottieni già un set minimo molto utile:

- successo
- successo funzionale
- errore controllato

---

# 7. Perché questo lab è Observability anche se non c’è ancora Log Analytics

Perché l’osservabilità non nasce solo quando colleghi Log Analytics o Grafana.

Nasce nel momento in cui un’applicazione espone e produce segnali leggibili.

Nel LAB09 hai già:

- endpoint
- status code
- request-id
- log applicativi
- durata richiesta

Log Analytics arriverà in un laboratorio successivo per centralizzare questi segnali.

Ma la qualità osservabile del servizio nasce qui.

---

# 8. Dove opererai

Per questo laboratorio, se disponi di una subscription trial, esegui **tutti i comandi Azure CLI e Docker in WSL Ubuntu locale**.

Non usare Azure Cloud Shell per questo LAB.

## WSL Ubuntu

In WSL Ubuntu eseguirai:

- `az login`
- `az account show`
- creazione ACR
- recupero `loginServer`
- `az acr login`
- `docker build`
- `docker tag`
- `docker push`
- creazione ACI
- verifica stato ACI
- recupero IP pubblico
- test con `curl`
- recupero log con `az container logs`
- aggiornamento evidenze e file ambiente
- comandi Git finali

## Azure Portal

Userai il portale Azure solo per verifiche visive:

- verificare che ACR sia stato creato
- verificare repository e tag dentro ACR
- verificare che ACI esista
- controllare stato e IP pubblico

## Regola pratica del laboratorio

Per tutto il LAB09 mantieni lo stesso flusso operativo:

1. apri WSL Ubuntu
2. fai login ad Azure
3. costruisci e pubblichi l’immagine
4. distribuisci la container instance
5. testi il servizio
6. leggi i log
7. aggiorni i file di evidenza

In questo modo lavori in un ambiente unico, coerente e ripetibile.

---

# 9. Che cosa imparerai davvero

Al termine di questo laboratorio dovrai aver capito che:

- un container in cloud non nasce “magicamente”: parte da un’immagine
- il registry è il ponte tra build locale e runtime Azure
- ACI è un runtime semplice ma reale per far partire un servizio containerizzato
- l’osservabilità dell’app si vede già da:

  - endpoint
  - status
  - request-id
  - log
  - durata

E soprattutto dovrai aver chiaro che:

> un servizio osservabile non è solo raggiungibile, ma produce segnali utili a capirne il comportamento.

---

# PARTE 2 - Laboratorio guidato passo-passo

## Prerequisiti

Per il LAB09 ti servono:

- LAB08 completato con Resource Group già esistente
- Docker funzionante in locale su WSL
- repository LAB09 contenente i file necessari per build immagine:

  - `Dockerfile`
  - `src/`
  - eventuali file requisiti dell’app

---

## Step 1 - Prepara l’ambiente locale in WSL Ubuntu

Apri WSL Ubuntu ed esegui:

```bash
mkdir -p ~/corso_obs/NOME-REPOSITORY/labs/lab09
cd ~/corso_obs/NOME-REPOSITORY/labs/lab09
script -a cmdlog_lab09.txt
mkdir -p docs
```

### Che cosa fanno i comandi

#### `mkdir -p ~/corso_obs/NOME-REPOSITORY/labs/lab09`

Crea la cartella del laboratorio.

#### `cd ~/corso_obs/NOME-REPOSITORY/labs/lab09`

Entra nella cartella del laboratorio.

#### `script -a cmdlog_lab09.txt`

Avvia la registrazione della sessione terminale e salva il log dei comandi in `cmdlog_lab09.txt`.

#### `mkdir -p docs`

Crea la cartella `docs` per file evidenze.

### Checkpoint

Devi avere:

- cartella `~/corso_obs/NOME-REPOSITORY/labs/lab09`
- log terminale attivo
- cartella `docs`

---

## Step 2 - Esegui login Azure CLI in WSL

Esegui:

```bash
az login
az account show
```

### Spiegazione dei comandi

#### `az login`

Apre il flusso di autenticazione Azure.

#### `az account show`

Mostra il contesto attivo dell’account Azure.

### Perché lo fai

Perché in WSL farai sia operazioni Docker sia operazioni Azure CLI, quindi il contesto Azure deve essere valido.

### Evidenza richiesta

Copia un estratto utile di `az account show` nel file finale.

---

## Step 3 - Imposta le variabili coerenti con LAB08

Esegui:

```bash
export RG="rg-observability-lab"
export LOCATION="westeurope"
```

### Che cosa fanno

- `RG` contiene il nome del Resource Group persistente
- `LOCATION` contiene la region del modulo

### Verifica facoltativa

```bash
echo "$RG"
echo "$LOCATION"
```

---

## Step 4 - Verifica i file del repository del LAB09

Esegui:

```bash
ls -la
ls -la src
```

### Che cosa fanno i comandi

#### `ls -la`

Mostra i file della directory corrente, inclusi quelli nascosti.

#### `ls -la src`

Mostra il contenuto della cartella `src`.

### Che cosa devi osservare

Devi verificare la presenza di:

- `Dockerfile`
- cartella `src`
- eventuali file necessari all’app

### Checkpoint #1

Questo step prepara la build dell’immagine `obsapp:v2`.

---

## Step 5 - Costruisci l’immagine Docker locale

Esegui:

```bash
docker build -t obsapp:v2 .
docker images | head
```

### Spiegazione del primo comando

```bash
docker build -t obsapp:v2 .
```

- `docker build` costruisce un’immagine
- `-t obsapp:v2` assegna nome e tag all’immagine
- `.` indica il contesto di build corrente

### Spiegazione del secondo comando

```bash
docker images | head
```

- `docker images` mostra le immagini presenti localmente
- `| head` limita l’output alle prime righe

### Checkpoint #1

Devi verificare che `obsapp:v2` esista localmente.

### Evidenza richiesta

Copia l’output di `docker images | head` con la riga di `obsapp:v2`.

---

## Step 6 - Crea Azure Container Registry (ACR)

Esegui questo step **in WSL Ubuntu locale**, nella stessa sessione in cui hai già eseguito `az login`.

### Comandi da eseguire

```bash
ACR_NAME="obsacr$RANDOM"

az acr create \
  --resource-group "$RG" \
  --name "$ACR_NAME" \
  --sku Basic \
  --location "$LOCATION"
```

Poi recupera il login server:

```bash
ACR_LOGIN_SERVER=$(az acr show --name "$ACR_NAME" --query loginServer -o tsv)
echo "$ACR_LOGIN_SERVER"
```

### Che cosa stai facendo

In questo step stai creando un **Azure Container Registry**, cioè un registry privato Azure in cui pubblicherai l’immagine Docker costruita localmente.

### Spiegazione dettagliata dei comandi

#### `ACR_NAME="obsacr$RANDOM"`

Crea una variabile chiamata `ACR_NAME`.

- `obsacr` è il prefisso scelto
- `$RANDOM` aggiunge un numero casuale
- il risultato è un nome probabilmente univoco

#### `az acr create ...`

Crea il registry Azure:

- `--resource-group "$RG"` indica dove creare la risorsa
- `--name "$ACR_NAME"` assegna il nome del registry
- `--sku Basic` usa il tier minimo richiesto
- `--location "$LOCATION"` usa la regione già impostata

#### `az acr show --query loginServer -o tsv`

Recupera il nome DNS del registry, ad esempio:

```text
obsacr12345.azurecr.io
```

Questo valore serve per taggare e pubblicare l’immagine.

### Checkpoint #2

Nel portale, cercando **Container registries**, devi vedere il tuo registry.

### Evidenza richiesta

Salva nel file delle evidenze:

- il nome effettivo del registry
- il valore di `ACR_LOGIN_SERVER`
- uno screenshot del portale in cui il registry è visibile

---

## Step 7 - Esegui login al registry e push dell’immagine

Esegui tutto da **WSL Ubuntu locale**, nella stessa sessione in cui hai creato ACR.

### Comandi da eseguire

```bash
az acr login --name "$ACR_NAME"

docker tag obsapp:v2 "$ACR_LOGIN_SERVER/obsapp:v2"
docker push "$ACR_LOGIN_SERVER/obsapp:v2"
```

### Obiettivo dello step

In questo passaggio fai tre cose fondamentali:

1. autentichi Docker verso il registry Azure
2. prepari il nome completo dell’immagine remota
3. invii l’immagine dal tuo computer al registry Azure

### Spiegazione dettagliata dei comandi

#### `az acr login --name "$ACR_NAME"`

Autentica il client Docker verso il registry Azure Container Registry.

#### `docker tag obsapp:v2 "$ACR_LOGIN_SERVER/obsapp:v2"`

Crea un nuovo riferimento alla stessa immagine locale usando il nome completo richiesto dal registry remoto.

#### `docker push "$ACR_LOGIN_SERVER/obsapp:v2"`

Invia l’immagine al registry Azure.

### Checkpoint #3

Nel portale, dentro ACR, devi vedere:

- repository `obsapp`
- tag `v2`

### Evidenza richiesta

Salva nel file delle evidenze:

- estratto del terminale con `docker push`
- screenshot del portale ACR con repository e tag visibili

---

## Step 8 - Definisci il nome della Container Instance

Esegui:

```bash
ACI_NAME="obsapp-aci"
```

### Perché questo step è utile

Assegni un nome chiaro e stabile alla Azure Container Instance che creerai.

---

## Step 9 - Crea Azure Container Instance (ACI) con IP pubblico

Esegui questo step **in WSL Ubuntu locale**.

### Comandi da eseguire

```bash
ACR_LOGIN_SERVER=$(az acr show -n "$ACR_NAME" --query loginServer -o tsv)
CR_USERNAME=$(az acr credential show -n "$ACR_NAME" --query username -o tsv)
CR_PASSWORD=$(az acr credential show -n "$ACR_NAME" --query "passwords[0].value" -o tsv)

az container create \
  --resource-group "$RG" \
  --name "$ACI_NAME" \
  --image "$ACR_LOGIN_SERVER/obsapp:v2" \
  --registry-login-server "$ACR_LOGIN_SERVER" \
  --registry-username "$CR_USERNAME" \
  --registry-password "$CR_PASSWORD" \
  --os-type Linux \
  --cpu 1 \
  --memory 1.5 \
  --ip-address Public \
  --ports 8000 \
  --environment-variables PORT=8000


Poi verifica lo stato:

```bash
az container show --resource-group "$RG" --name "$ACI_NAME" --query instanceView.state -o tsv
```

### Che cosa stai facendo

In questo step chiedi ad Azure di:

- prendere l’immagine dal tuo registry ACR
- creare un container gestito
- esporlo con un IP pubblico
- aprire la porta `8000`
- passare al container la variabile ambiente `PORT=8000`

### Spiegazione dettagliata del comando `az container create`

- `--resource-group "$RG"`: usa il RG persistente del modulo
- `--name "$ACI_NAME"`: nome della container instance
- `--image "$ACR_LOGIN_SERVER/obsapp:v2"`: immagine da eseguire
- `--registry-login-server "$ACR_LOGIN_SERVER"`: registry di provenienza
- `--ip-address Public`: espone l’istanza con IP pubblico
- `--ports 8000`: apre la porta 8000
- `--environment-variables PORT=8000`: passa la variabile d’ambiente usata dall’app

### Spiegazione del comando di verifica stato

```bash
az container show --resource-group "$RG" --name "$ACI_NAME" --query instanceView.state -o tsv
```

Legge lo stato runtime della container instance.

### Checkpoint #4

Lo stato deve risultare:

```text
Running
```

### Evidenza richiesta

Salva nel file delle evidenze:

- output del comando di creazione ACI
- output del comando di verifica stato

---

## Step 10 - Recupera l’IP pubblico della ACI

Esegui questo step **in WSL Ubuntu locale**.

### Comandi da eseguire

```bash
ACI_PUBLIC_IP=$(az container show --resource-group "$RG" --name "$ACI_NAME" --query ipAddress.ip -o tsv)
echo "$ACI_PUBLIC_IP"
```

### Che cosa stai facendo

Recuperi l’indirizzo IP pubblico assegnato da Azure alla container instance.

### Checkpoint

Devi ottenere un IP pubblico valorizzato.

### Evidenza richiesta

Salva nel file delle evidenze:

- valore di `ACI_PUBLIC_IP`

---

## Step 11 - Testa gli endpoint pubblici

Esegui questi test **in WSL Ubuntu locale**.

### Comandi da eseguire

```bash
curl -i "http://$ACI_PUBLIC_IP:8000/health"
curl -i "http://$ACI_PUBLIC_IP:8000/time"
curl -i "http://$ACI_PUBLIC_IP:8000/nope"
```

### Obiettivo dello step

Verificare il comportamento reale dell’applicazione dal punto di vista di un client esterno.

### Spiegazione dei comandi

#### `curl -i`

- `curl` invia una richiesta HTTP
- `-i` mostra anche gli header di risposta

### Che cosa devi osservare

Devi ottenere:

- almeno una risposta `200`
- almeno una risposta `404`
- presenza dell’header `X-Request-Id`

### Evidenza richiesta

Salva nel file delle evidenze:

- output completo di `/health`
- output completo di `/time`
- output completo di `/nope`
- conferma della presenza di `X-Request-Id`

---

## Step 12 - Recupera i log del container

Esegui questo step **in WSL Ubuntu locale**.

### Comando da eseguire

```bash
az container logs --resource-group "$RG" --name "$ACI_NAME"
```

### Che cosa stai facendo

Leggi i log stdout/stderr del container che gira in Azure Container Instances.

### Che cosa devi cercare nei log

Devi trovare campi osservabili significativi, ad esempio:

- `request_id`
- `status`
- `duration_ms`

### Checkpoint #6

Nei log devi trovare campi osservabili coerenti con le richieste eseguite.

### Evidenza richiesta

Salva nel file delle evidenze:

- un estratto dei log contenente almeno `request_id`, `status`, `duration_ms`

---

## Step 13 - Aggiorna il file ambiente persistente del Modulo 2

Apri `docs/azure_env.md` e aggiungi:

```markdown
ACR_NAME=<nome effettivo del registry>
ACR_LOGIN_SERVER=<login server effettivo>
ACI_NAME=obsapp-aci
ACI_PUBLIC_IP=<ip effettivo>
```

### Perché lo fai

Perché queste risorse verranno riutilizzate nei laboratori successivi.

---

## Step 14 - Crea il file evidenze del laboratorio

Crea:

```text
docs/evidence_lab09.md
```

Il file deve contenere almeno:

- screenshot o estratti ACR repository
- output `curl` con `200/404` e header `request-id`
- estratto log ACI

Puoi usare questa struttura:

```markdown
# LAB09 - Evidence

## ACR
- ACR_NAME:
- ACR_LOGIN_SERVER:

### Repository
- Repository visibile: obsapp
- Tag visibile: v2

## ACI
- ACI_NAME: obsapp-aci
- ACI_PUBLIC_IP:

### Stato runtime
- Output `az container show ... instanceView.state`:
[incollare output]

## Test endpoint

### /health
[incollare output curl]

### /time
[incollare output curl]

### /nope
[incollare output curl]

## Verifica header
- X-Request-Id presente: SÌ/NO

## Log container
[incollare estratto con request_id, status, duration_ms]

## Note finali
- Che cosa è chiaro:
- Che cosa è nuovo:
- Qual è la differenza tra stato risorsa, risposta HTTP e log applicativi:
```

---

## Step 15 - Consegna nel repository Git

Esegui:

```bash
git add docs/azure_env.md docs/evidence_lab09.md
git commit -m "[LAB09] ACR + ACI completato"
git push
```

### Che cosa fanno i comandi

- `git add` aggiunge i file aggiornati
- `git commit` salva una versione locale
- `git push` invia i file al repository remoto

---

# PARTE 3 - Checkpoint, criteri di completamento e significato professionale

# 1. Checkpoint riassuntivi

## Checkpoint #1

`obsapp:v2` esiste localmente dopo la build.

## Checkpoint #2

ACR è visibile nel portale dopo la creazione.

## Checkpoint #3

Nel repository ACR sono visibili `obsapp` e tag `v2`.

## Checkpoint #4

ACI risulta `Running`.

## Checkpoint #5

Gli endpoint pubblici restituiscono `200`, `404` e header `X-Request-Id`.

## Checkpoint #6

Nei log compaiono campi osservabili come `request_id`, `status`, `duration_ms`.

---

# 2. Criteri di completamento

Il LAB09 è completato se risultano soddisfatti questi criteri:

- ACR creato e immagine pushata
- ACI running con IP pubblico
- endpoint testati
- log recuperati
- evidence completata

---

# 3. Che cosa stai imparando davvero

Questo laboratorio ti sta insegnando almeno cinque cose importanti.

## 3.1 Un’applicazione containerizzata ha un ciclo di promozione

Non basta averla in locale. Deve passare da immagine a registry a runtime cloud.

## 3.2 Il registry è una parte fondamentale della catena

ACR non è un dettaglio, è il punto di distribuzione dell’immagine.

## 3.3 Il runtime cloud va osservato a più livelli

- stato risorsa
- endpoint client
- log interni

## 3.4 L’osservabilità applicativa inizia dai segnali giusti

Se hai `request_id`, `status`, `duration_ms`, hai già una base seria.

## 3.5 Questo laboratorio prepara direttamente i successivi

I log e i segnali generati qui saranno la base per:

- query SQL nel LAB10
- centralizzazione KQL nel LAB11
- alerting nel LAB12
- workbook nel LAB13

---

# 4. Errori comuni da evitare

## Errore 1 - Fermarsi alla build locale

Se l’immagine esiste solo localmente, il laboratorio non è compiuto.

## Errore 2 - Confondere ACR e ACI

- ACR conserva immagini
- ACI esegue container

## Errore 3 - Guardare solo il portale

Il portale ti dice se la risorsa esiste, ma non basta per capire cosa sta facendo l’applicazione.

## Errore 4 - Non controllare gli header HTTP

L’header `X-Request-Id` è un segnale molto importante, non un dettaglio cosmetico.

## Errore 5 - Non aggiornare `azure_env.md`

Se non salvi nomi e IP, nei laboratori successivi inizierai a rincorrere le risorse inutilmente.

---

# 5. Conclusione

Questo laboratorio segna l’ingresso reale nell’**observability applicativa su Azure**.

Dopo aver completato il LAB09 devi avere chiaro che:

- hai costruito e pubblicato un’immagine container
- hai distribuito un servizio su Azure Container Instances
- hai ottenuto un endpoint pubblico funzionante
- hai verificato status HTTP, errori e header di correlazione
- hai letto i primi log applicativi osservabili con campi utili come `request_id`, `status` e `duration_ms`

Questo è il punto da cui, nel LAB10, potrai iniziare a ragionare sulla persistenza dei dati osservabili e sul calcolo di SLI via query SQL.
