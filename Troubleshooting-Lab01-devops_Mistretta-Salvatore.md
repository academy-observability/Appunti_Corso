### TROBLESHOOTING MISTRETTA SALVATORE
# POOL AGENT
Aggiungere pool Self-Hosted
- Andare su project settings
- Agent pools
- Add pool
- Selezionare Self-hosted
- Indicare un nome
- Spuntare la casella sui permessi
- Create

Aggiungere agent
- Entrare dentro il pool creato
- In Agent e selezionare New Agent
- Scaricare agent Linux
- Eseguire le istruzioni da riga di comando  
- Quando si lancia il comando .\config.cmd bisogna inserire come URL: https://dev.azure.com/TUA-ORG e il token ottenuto
- Eseguire .\run.cmd
- Se tutto è corretto dovrebbe restituire Listening for job

# AZURE CREATE
- Abilitare admin su ACR, lanciando da riga di comando questo: 
```bash 
    az acr update -n obsacr11697 --admin-enabled true
``` 
- Aggiungere questa riga --os-type Linux \ al comando create

- Modificare i comandi per la ricerca di ACR_USER e ACR_PASS per evitare che copi male i valori nelle variabili

- Comando completo Azure Create
```bash
ACR_USER=$(az acr credential show -n $(acrName) --query username -o tsv | tr -d '\r\n')
                ACR_PASS=$(az acr credential show -n $(acrName) --query "passwords[0].value" -o tsv | tr -d '\r\n')

                echo "Creo nuovo container su ACI"
                az container create \
                  --resource-group $(resourceGroupName) \
                  --name $(aciName) \
                  --image $FULL_IMAGE_NAME \
                  --os-type Linux \
                  --cpu 1 \
                  --memory 1.5 \
                  --ports $(containerPort) \
                  --ip-address Public \
                  --dns-name-label $(aciDnsName) \
                  --location $(location) \
                  --registry-login-server $(acrName).azurecr.io \
                  --registry-username $ACR_USER \
                  --registry-password $ACR_PASS
```
