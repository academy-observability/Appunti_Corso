# LAB02 - Pipeline CI/CD più ordinata sulla standalone

## GitHub, Azure DevOps, ACR e ACI con build, deploy e smoke test separati

---

# 1. Obiettivo del laboratorio

In questo laboratorio manterrai lo stesso flusso del LAB01:

- **GitHub** come repository del codice
- **Azure DevOps Pipelines** come motore CI/CD
- **Azure Container Registry (ACR)** come registry immagini
- **Azure Container Instances (ACI)** come primo target cloud

La differenza è che qui renderai la pipeline più ordinata e più professionale.

Alla fine del laboratorio dovrai essere in grado di:

- spiegare perché una pipeline monolitica è meno leggibile
- usare una pipeline **multistage**
- separare **build**, **deploy** e **verifica**
- usare `Build.BuildId` come tag immagine coerente
- distribuire su ACI la stessa immagine costruita nella pipeline
- eseguire uno **smoke test** finale
- capire perché una pipeline “verde” non basta, se nessuno verifica davvero il comportamento dell’applicazione 

---

# 2. Scopo del laboratorio

Nel LAB01 hai realizzato un primo flusso completo:

- codice su GitHub
- pipeline Azure DevOps
- build immagine
- push su ACR
- deploy su ACI

Quel flusso andava bene come primo step, ma era ancora troppo compatto: tutto in un solo blocco.

In questo LAB02 introduci una struttura più chiara:

- **Stage 1 - Build**
- **Stage 2 - Deploy**
- **Stage 3 - SmokeTest**

Azure DevOps documenta le pipeline multistage proprio per dividere il processo in parti più leggibili e controllabili. 
È un miglioramento piccolo, ma è il tipo di miglioramento che separa una demo iniziale da una pipeline in cui il codice YAML sia veramente leggibile. 

---

# Parte 1 - Spiegazione dei termini e degli strumenti

# 3. Flusso architetturale del laboratorio

Il flusso architetturale non cambia rispetto al LAB01:

```text
Sviluppatore
   |
   v
GitHub Repository
   |
   v
Azure DevOps Pipeline
   |
   +--> Stage Build
   +--> Stage Deploy
   +--> Stage SmokeTest
   |
   v
ACR
   |
   v
ACI
   |
   v
Applicazione raggiungibile via HTTP
````

Azure Pipelines può costruire repository GitHub e attivarsi automaticamente su commit e pull request. La pipeline rimane quindi collegata al repository GitHub come sorgente del codice. 

---

# 4. Che cosa significa pipeline multistage

Una pipeline multistage è una pipeline divisa in **stages**, cioè fasi logiche del processo.

Nel nostro caso useremo:

* **Build**
* **Deploy**
* **SmokeTest**

Azure DevOps documenta che una pipeline multistage dà più visibilità al processo e rende più semplice integrare controlli, verifiche e approvazioni nelle varie fasi. 

## 4.1 Perché è utile

Separare le fasi serve a capire meglio:

* dove fallisce la pipeline
* che cosa è già stato completato
* se il problema è nella build, nel deploy o nei controlli finali

## 4.2 Perché è meglio della versione del LAB01

Nel LAB01 avevi un solo grande blocco.

Qui invece puoi leggere la pipeline come un flusso più disciplinato:

1. costruisco
2. distribuisco
3. verifico

Questa distinzione è molto più vicina al modo in cui un team serio ragiona sui rilasci.

---

# 5. Che cos’è `Build.BuildId`

`Build.BuildId` è una variabile di sistema di Azure Pipelines. Microsoft documenta che identifica il singolo run della pipeline e può essere usata per assegnare un valore univoco a un build. 

## 5.1 Perché la usiamo

La useremo come tag dell’immagine container.

Per esempio:

```text
nomeregistry.azurecr.io/obsapp-v3:123
```

dove `123` è il valore del `Build.BuildId`.

## 5.2 Perché è meglio di `latest`

Per un laboratorio introduttivo, usare un tag univoco è meglio di `latest` perché:

* rende più chiaro quale build ha prodotto quella immagine
* evita ambiguità
* aiuta a capire che una pipeline dovrebbe produrre risultati tracciabili

---

# 6. Che cosa significa smoke test

Uno **smoke test** è un controllo rapido eseguito dopo il deploy per verificare che l’applicazione risponda almeno nelle funzioni essenziali.

Nel nostro LAB02 lo smoke test controllerà endpoint come:

* `/health`
* `/`
* `/time`
* `POST /echo`

## 6.1 Perché è utile

Perché una pipeline può aver:

* costruito l’immagine correttamente
* pubblicato l’immagine correttamente
* creato il container correttamente

e l’applicazione può comunque non funzionare.

Lo smoke test serve proprio a evitare questa falsa sensazione di successo.

---

# 7. Il ruolo di GitHub in LAB02

GitHub continua a essere il repository del codice sorgente.

Conterrà:

* codice dell’app
* Dockerfile
* requirements
* file YAML aggiornato della pipeline

Azure Pipelines supporta repository GitHub come sorgente e può costruirli automaticamente tramite integrazione dedicata. ([Microsoft Learn][2])

---

# 8. Il ruolo di Azure DevOps in LAB02

Azure DevOps, nel nostro laboratorio, continua a essere il motore della pipeline.

Qui però lo userai in modo un po’ più evoluto:

* con più stage
* con variabili più ordinate
* con verifiche finali più esplicite

Per eseguire i comandi Azure continueremo a usare il task **AzureCLI@2**, che è il task documentato per eseguire script Azure CLI in pipeline. ([Microsoft Learn][4])

---

# 9. Il ruolo di ACR in LAB02

ACR continua a essere il registry immagini.

La pipeline:

* costruisce l’immagine
* le assegna un tag con `Build.BuildId`
* la pubblica in ACR

Microsoft documenta ACR come il servizio Azure per archiviare e gestire immagini container private e altri artifact correlati, utilizzabile con la Docker CLI per login, push e pull. 

---

# 10. Il ruolo di ACI in LAB02

ACI continua a essere il primo target cloud del laboratorio.

In questo LAB02 non cambiamo ancora piattaforma di deploy: ci serve ancora un target semplice, così il focus resta sulla qualità della pipeline e non sull’aumento della complessità dell’ambiente. Microsoft documenta ACI come un servizio che permette di distribuire rapidamente un container isolato e di renderlo disponibile tramite FQDN. 

---

# 11. Concetti chiave da ricordare prima della parte pratica

Prima di passare allo step-by-step, tieni fissi questi punti:

* **GitHub** conserva il codice
* **Azure DevOps** esegue la pipeline
* **ACR** conserva l’immagine
* **ACI** esegue il container
* la pipeline è ora divisa in:

  * Build
  * Deploy
  * SmokeTest

Se il LAB01 ti ha mostrato il flusso completo, il LAB02 ti mostra come iniziare a renderlo leggibile e controllabile.

---

# Parte 2 - Step-by-step guidato

# 12. Prerequisiti del laboratorio

Per eseguire LAB02 devi aver completato con successo LAB01.

Devono quindi essere già disponibili:

* repository GitHub del LAB01
* pipeline Azure DevOps collegata al repository GitHub
* Azure Resource Manager service connection funzionante
* Azure Container Registry già esistente
* Azure Container Instance già deployabile
* Azure CLI e Docker funzionanti in locale

Inoltre è utile che l’admin account di ACR sia già abilitato, perché in questo laboratorio continuiamo a usare `az acr credential show` per semplificare il deploy verso ACI. L’account admin di ACR è disabilitato di default. 

---

# 13. Apertura del progetto locale

Apri il terminale WSL:

```bash id="ue2q8r"
cd ~/corso_obs/lab01-github-appv3
git status
find . -maxdepth 2 -type f | sort
```

Dovresti vedere almeno:

* `src/app.py`
* `requirements.txt`
* `Dockerfile`
* `.dockerignore`
* `azure-pipelines.yml`

---

# 14. Verifica rapida dell’applicazione locale

Questa parte non è nuova, ma è utile per verificare di non aver rotto nulla prima di cambiare la pipeline.

```bash id="s7wq7q"
python3 -m py_compile src/app.py
```

Se non compare output, la sintassi Python è valida.

---

# 15. Sostituzione del file `azure-pipelines.yml`

Apri il file:

```bash id="u0kqk5"
code azure-pipelines.yml
```

Sostituisci il contenuto con questo, adattando i valori del tuo ambiente:

```yaml id="gxj5j6"
trigger:
  - main

pr: none

pool:
  vmImage: ubuntu-latest

variables:
  azureServiceConnection: 'sc-obs-azure-rg'
  resourceGroupName: 'rg-observability-dev'
  acrName: 'nomeregistry'
  imageName: 'obsapp-v3'
  imageTag: '$(Build.BuildId)'
  aciName: 'obsapp-v3-aci'
  aciDnsName: 'obsappv3-NOMEUNIVOCO'
  location: 'westeurope'
  containerPort: '8000'

stages:
- stage: Build
  displayName: Build and Push
  jobs:
  - job: BuildPush
    displayName: Build image and push to ACR
    steps:
    - checkout: self

    - task: AzureCLI@2
      displayName: Login to Azure and ACR, then build and push image
      inputs:
        azureSubscription: '$(azureServiceConnection)'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          set -e

          echo "Subscription attiva"
          az account show

          echo "Login ACR"
          az acr login --name $(acrName)

          FULL_IMAGE_NAME="$(acrName).azurecr.io/$(imageName):$(imageTag)"

          echo "Build immagine: ${FULL_IMAGE_NAME}"
          docker build -t ${FULL_IMAGE_NAME} .

          echo "Push immagine: ${FULL_IMAGE_NAME}"
          docker push ${FULL_IMAGE_NAME}

- stage: Deploy
  displayName: Deploy to ACI
  dependsOn: Build
  condition: succeeded()
  jobs:
  - job: DeployACI
    displayName: Deploy container to Azure Container Instances
    steps:
    - checkout: self

    - task: AzureCLI@2
      displayName: Deploy image from ACR to ACI
      inputs:
        azureSubscription: '$(azureServiceConnection)'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          set -e

          FULL_IMAGE_NAME="$(acrName).azurecr.io/$(imageName):$(imageTag)"

          echo "Recupero credenziali ACR"
          ACR_USER=$(az acr credential show --name $(acrName) --query username -o tsv)
          ACR_PASS=$(az acr credential show --name $(acrName) --query "passwords[0].value" -o tsv)

          echo "Elimino eventuale container precedente"
          az container delete \
            --resource-group $(resourceGroupName) \
            --name $(aciName) \
            --yes || true

          echo "Creo il nuovo container"
          az container create \
            --resource-group $(resourceGroupName) \
            --name $(aciName) \
            --image ${FULL_IMAGE_NAME} \
            --os-type Linux \
            --cpu 1 \
            --memory 1.5 \
            --ports $(containerPort) \
            --ip-address Public \
            --dns-name-label $(aciDnsName) \
            --location $(location) \
            --registry-login-server $(acrName).azurecr.io \
            --registry-username ${ACR_USER} \
            --registry-password ${ACR_PASS} \
            --environment-variables PORT=$(containerPort)

- stage: SmokeTest
  displayName: Verify deployment
  dependsOn: Deploy
  condition: succeeded()
  jobs:
  - job: VerifyACI
    displayName: Run smoke tests against deployed app
    steps:
    - task: AzureCLI@2
      displayName: Recover FQDN and test application
      inputs:
        azureSubscription: '$(azureServiceConnection)'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          set -e

          FQDN=$(az container show \
            --resource-group $(resourceGroupName) \
            --name $(aciName) \
            --query ipAddress.fqdn \
            -o tsv)

          if [ -z "$FQDN" ]; then
            echo "FQDN non disponibile"
            exit 1
          fi

          APP_URL="http://${FQDN}:$(containerPort)"

          echo "Attendo alcuni secondi prima dei test"
          sleep 15

          echo "Test /health"
          curl --fail --silent --show-error "${APP_URL}/health"
          echo

          echo "Test /"
          curl --fail --silent --show-error "${APP_URL}/"
          echo

          echo "Test /time"
          curl --fail --silent --show-error "${APP_URL}/time"
          echo

          echo "Test POST /echo"
          curl --fail --silent --show-error \
            -X POST "${APP_URL}/echo" \
            -H 'Content-Type: application/json' \
            -d '{"lab":"02","check":"smoke-test"}'
          echo

          echo "Lettura log container"
          az container logs \
            --resource-group $(resourceGroupName) \
            --name $(aciName)
```

---

# 16. Perché questa pipeline è migliore di quella del LAB01

Questa versione della pipeline introduce cinque miglioramenti reali:

1. **build separata dal deploy**
2. **deploy separato dalla verifica**
3. **tag immagine coerente con il build**
4. **più leggibilità nella UI di Azure DevOps**
5. **verifica finale automatizzata**

Le pipeline multistage e le variabili di sistema come `Build.BuildId` sono proprio pensate per supportare questo tipo di struttura. 

---

# 17. Salvataggio e push su GitHub

Salva il file e poi esegui:

```bash id="l7b76i"
git add azure-pipelines.yml
git commit -m "LAB02 - Multistage pipeline with build deploy smoke test"
git push
```

Verifica che il file aggiornato sia presente sul repository GitHub.

---

# 18. Esecuzione della pipeline in Azure DevOps

Nel portale Azure DevOps:

1. vai in **Pipelines**
2. apri la pipeline collegata al repository GitHub del LAB01
3. verifica che usi il file `/azure-pipelines.yml`
4. esegui la pipeline

Dovresti vedere chiaramente tre stage:

* **Build**
* **Deploy**
* **SmokeTest**

Azure DevOps visualizza gli stage nella UI della pipeline proprio per rendere leggibile il processo. 

---

# 19. Verifica dello stage Build

Apri i log dello stage `Build` e verifica che compaiano:

* `az account show`
* `az acr login`
* `docker build`
* `docker push`

Inoltre controlla che l’immagine venga pubblicata con un tag coerente con il build corrente.
`Build.BuildId` serve esattamente a questo. 

Per ulteriore verifica, da CLI locale puoi eseguire:

```bash id="uqdjlwm"
az acr repository show-tags \
  --name NOME_ACR \
  --repository obsapp-v3 \
  -o table
```

---

# 20. Verifica dello stage Deploy

Apri i log dello stage `Deploy` e verifica che compaiano:

* recupero credenziali ACR
* delete del vecchio container
* create del nuovo container
* uso della stessa immagine pubblicata nello stage Build

ACI può distribuire immagini private da ACR quando riceve i parametri corretti del registry e delle credenziali. 

Puoi verificare anche da CLI:

```bash id="635q9c"
az container show \
  --resource-group NOME_RESOURCE_GROUP \
  --name NOME_ACI \
  --query instanceView.state
```

---

# 21. Verifica dello stage SmokeTest

Apri i log dello stage `SmokeTest`.

Devi vedere:

* recupero FQDN
* test `/health`
* test `/`
* test `/time`
* test `POST /echo`
* lettura dei log del container

Il valore vero di questa stage è che la pipeline non si limita a dire “deploy completato”, ma controlla che il servizio risponda davvero.

---

# 22. Verifiche manuali finali

Recupera l’FQDN del container:

```bash id="9wyxjx"
FQDN=$(az container show \
  --resource-group NOME_RESOURCE_GROUP \
  --name NOME_ACI \
  --query ipAddress.fqdn \
  -o tsv)

echo $FQDN
```

Esegui poi i test finali:

```bash id="6ljwqd"
curl http://$FQDN:8000/
curl http://$FQDN:8000/health
curl http://$FQDN:8000/time
curl -X POST http://$FQDN:8000/echo -H 'Content-Type: application/json' -d '{"manual":"true"}'
curl http://$FQDN:8000/error
```

Leggi anche i log:

```bash id="8kzf8x"
az container logs \
  --resource-group NOME_RESOURCE_GROUP \
  --name NOME_ACI
```

---

# 23. Troubleshooting minimo

## 23.1 La pipeline non parte

Possibili cause:

* il file `azure-pipelines.yml` non è sul branch `main`
* la pipeline Azure DevOps punta a un file YAML diverso
* il collegamento col repository GitHub non è corretto

Azure Pipelines supporta i trigger su repository GitHub, ma la pipeline deve essere collegata al repository e al file YAML corretti.

## 23.2 Lo stage Build fallisce su `docker build`

Possibili cause:

* file mancanti
* `Dockerfile` errato
* contesto di build sbagliato

Verifica:

```bash id="2i0xgc"
find . -maxdepth 2 -type f | sort
```

## 23.3 Lo stage Deploy fallisce su `az acr credential show`

Possibile causa:

* admin account ACR non abilitato

Verifica:

```bash id="9r2lk7"
az acr credential show --name NOME_ACR
```

Se non funziona:

```bash id="sh8h2i"
az acr update --name NOME_ACR --admin-enabled true
```

## 23.4 Lo stage Deploy fallisce su `az container create`

Possibili cause:

* DNS label già usata
* Resource Group errato
* credenziali ACR errate
* immagine non trovata

Cambia il valore di `aciDnsName` nel file YAML e rilancia.

## 23.5 Lo SmokeTest fallisce su `/health`

Possibili cause:

* container ancora in avvio
* errore runtime dell’app
* porta errata

Verifica stato e log:

```bash id="m6btw5"
az container show \
  --resource-group NOME_RESOURCE_GROUP \
  --name NOME_ACI \
  --query "{state:instanceView.state,fqdn:ipAddress.fqdn,ports:ipAddress.ports}"

az container logs \
  --resource-group NOME_RESOURCE_GROUP \
  --name NOME_ACI
```

---

# 24. Evidenze richieste

Crea il file:

```bash id="36livj"
mkdir -p docs
code docs/evidence_lab02.md
```

Inserisci questa struttura:

```md id="d3if2f"
# Evidence LAB02

## 1. Obiettivo compreso
Spiego con parole mie perché questa pipeline è migliore di quella del LAB01.

## 2. Repository GitHub
Indico il nome del repository usato e il branch principale.

## 3. Nuova struttura pipeline
Descrivo i tre stage:
- Build
- Deploy
- SmokeTest

## 4. Tag immagine
Indico il tag usato e spiego perché è stato scelto.

## 5. Verifica stage Build
Descrivo cosa accade nello stage Build.

## 6. Verifica stage Deploy
Descrivo cosa accade nello stage Deploy.

## 7. Verifica stage SmokeTest
Riporto i test eseguiti sugli endpoint e l’esito.

## 8. Log del container
Incollo alcune righe JSON e commento i campi principali.

## 9. Differenze rispetto al LAB01
Elenco almeno 5 miglioramenti introdotti.

## 10. Problemi incontrati
Descrivo eventuali errori e come li ho risolti.
```

---

# 25. Consegna

Al termine del laboratorio devi consegnare:

* repository GitHub aggiornato
* pipeline Azure DevOps multistage funzionante
* file `docs/evidence_lab02.md`

Salva il lavoro:

```bash id="db2uwq"
git add .
git commit -m "LAB02 completato - pipeline multistage con smoke test"
git push
```

---

# 26. Conclusione del laboratorio

In questo laboratorio non hai cambiato applicazione, ma hai migliorato il processo in modo sostanziale:

* più leggibilità
* più separazione delle responsabilità
* più controllo del rilascio
* prima verifica operativa automatizzata

Questo è il passaggio corretto prima del salto successivo:

**non più una sola app standalone, ma il progetto FE/BE completo con build multi-image**.




[1]: https://learn.microsoft.com/en-us/azure/devops/pipelines/process/create-multistage-pipeline?view=azure-devops&utm_source=chatgpt.com "Tutorial: Create a multistage pipeline with Azure DevOps"
[2]: https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/github?view=azure-devops&utm_source=chatgpt.com "Build GitHub repositories - Azure Pipelines"
[3]: https://learn.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&utm_source=chatgpt.com "Define variables - Azure Pipelines"
[4]: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/?view=azure-pipelines&utm_source=chatgpt.com "Azure Pipelines task reference"
[5]: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-get-started-docker-cli?utm_source=chatgpt.com "Push & Pull Container Image using Azure Container Registry"
[6]: https://learn.microsoft.com/en-us/azure/container-instances/container-instances-quickstart?utm_source=chatgpt.com "Deploy Docker container to container instance - Azure CLI"
[7]: https://learn.microsoft.com/en-us/azure/container-instances/container-instances-using-azure-container-registry?utm_source=chatgpt.com "Deploy container image from Azure Container Registry ..."
