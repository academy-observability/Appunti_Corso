# Modulo 3 - Fase preparatoria

##  Creazione di una organizzazione Azure DevOps e di un progetto Azure DevOps

---

# 1. Scopo di questo documento

Questo file è pensato per i partecipanti che, prima di iniziare **LAB01-devops**, non dispongono ancora di:

* una **organizzazione Azure DevOps**
* un **progetto Azure DevOps**

Nel percorso del modulo useremo **GitHub** come repository del codice e **Azure DevOps Pipelines** come motore CI/CD. Per poter usare Azure DevOps in modo corretto, però, ogni partecipante deve almeno:

* poter accedere a una organizzazione Azure DevOps
* lavorare dentro un progetto Azure DevOps già pronto

Microsoft descrive l’organizzazione come il contenitore di gruppi di progetti correlati, mentre il progetto è l’unità logica che isola repository, backlog, pipeline e altre risorse dal resto dell’organizzazione. ([Microsoft Learn][1])

---

# 2. Obiettivo di questo documento

Alla fine di questa procedura il partecipante dovrà avere:

* accesso a una **organizzazione Azure DevOps Services**
* un **progetto Azure DevOps** creato e raggiungibile dal browser
* una base minima pronta per i laboratori successivi

---

# 3. Che cosa sono organizzazione e progetto in Azure DevOps

## 3.1 Organizzazione

Una **organizzazione Azure DevOps** è il contenitore principale che raccoglie più progetti e permette di scalare il lavoro di team e gruppi diversi. Microsoft documenta che l’organizzazione può essere creata usando:

* account Microsoft personale
* account GitHub
* account aziendale o scolastico

Quando si usa un account aziendale o scolastico, l’organizzazione viene collegata automaticamente a **Microsoft Entra ID**. ([Microsoft Learn][1])

## 3.2 Progetto

Un **progetto Azure DevOps** è il contenitore operativo in cui si trovano, ad esempio:

* repository
* pipeline
* board
* permessi di progetto
* impostazioni del team

Microsoft spiega che ogni progetto isola i propri dati rispetto agli altri progetti presenti nella stessa organizzazione. ([Microsoft Learn][2])

---

# 4. Prerequisiti minimi

Prima di iniziare questa procedura, il partecipante deve avere almeno uno dei seguenti account:

* un **account Microsoft personale**
* un **account GitHub**
* un **account aziendale o scolastico**

Microsoft documenta tutti e tre questi metodi di accesso per la creazione di un’organizzazione Azure DevOps Services. ([Microsoft Learn][1])

---

# 5. Parte 1 - Creazione della organizzazione Azure DevOps

## 5.1 Accesso ad Azure DevOps

Apri il browser e vai al portale di Azure DevOps.

Poi scegli il metodo di accesso più adatto al tuo caso:

* account Microsoft personale
* account GitHub
* account aziendale o scolastico

Microsoft documenta che è possibile creare un’organizzazione usando uno qualunque di questi tipi di account. Se usi GitHub e il tuo indirizzo email GitHub è associato a una organizzazione collegata a Microsoft Entra ID, Microsoft raccomanda di usare l’account Entra ID invece dell’accesso GitHub diretto. ([Microsoft Learn][1])

## 5.2 Creazione della organizzazione

Una volta effettuato l’accesso:

1. completa l’eventuale procedura iniziale di registrazione
2. scegli l’opzione per creare una nuova organizzazione
3. assegna un nome chiaro all’organizzazione
4. completa la procedura guidata

Microsoft indica che, dopo la registrazione, l’organizzazione è raggiungibile con un URL del tipo:

```text
https://dev.azure.com/NOME_ORGANIZZAZIONE
```

([Microsoft Learn][1])

## 5.3 Suggerimento per il nome dell’organizzazione

Per evitare nomi casuali o poco leggibili, conviene usare un nome semplice, per esempio:

* `corso-observability-nomecognome`
* `academy-observability-nome`
* `devops-lab-nome`

Lo scopo è rendere l’organizzazione riconoscibile durante i laboratori.

## 5.4 Verifica finale della organizzazione

Dopo la creazione, verifica di riuscire a:

* aprire la home dell’organizzazione
* vedere la URL corretta nel browser
* accedere alle impostazioni dell’organizzazione

Se arrivi alla home dell’organizzazione, il primo prerequisito è soddisfatto. Microsoft documenta che l’organizzazione è il contenitore di base da cui si gestiscono progetti e impostazioni. ([Microsoft Learn][1])

---

# 6. Parte 2 - Creazione del progetto Azure DevOps

## 6.1 Perché creare un progetto

Il progetto è l’unità pratica in cui lavorerai nei laboratori.

Nel nostro percorso, il progetto conterrà almeno:

* pipeline
* service connections
* eventuali configurazioni di sicurezza di progetto
* integrazione con GitHub

Microsoft spiega che il progetto isola i propri dati e costituisce il contenitore logico per il lavoro del team. ([Microsoft Learn][2])

## 6.2 Passi per creare il progetto

Dalla home dell’organizzazione:

1. cerca il comando per creare un nuovo progetto
2. inserisci il nome del progetto
3. scegli la visibilità
4. scegli il tipo di version control
5. scegli il processo di lavoro
6. conferma la creazione

Microsoft documenta che, nella creazione del progetto, vengono richieste almeno impostazioni relative a nome, visibilità, controllo versione e processo. ([Microsoft Learn][2])

## 6.3 Valori consigliati per il corso

Per questo modulo puoi usare questi valori suggeriti:

* **Project name**: `Observability-DevOps`
* **Visibility**: `Private`
* **Version control**: `Git`
* **Work item process**: `Basic`

Anche se il codice resterà su GitHub e non in Azure Repos, mantenere il progetto in modalità **Git** è la scelta più lineare e coerente con il corso.

## 6.4 Verifica finale del progetto

Dopo la creazione del progetto, verifica di poter aprire queste sezioni del portale:

* **Overview**
* **Pipelines**
* **Project settings**

Se riesci ad aprire il progetto e le sue aree principali, il secondo prerequisito è soddisfatto.

---

# 7. Parte 3 - Controlli minimi prima di LAB01

Prima di passare a **LAB01**, ogni partecipante dovrebbe verificare di avere:

* una URL funzionante dell’organizzazione Azure DevOps
* un progetto Azure DevOps aperto e navigabile
* accesso a **Project settings**
* accesso alla sezione **Pipelines**

Questi controlli non sono un vezzo burocratico. Servono a evitare che, al momento di creare service connection e pipeline, metà della classe scopra di non avere nemmeno il contenitore di base pronto.

---

# 8. Permessi e sicurezza: chiarimento minimo

Quando si crea una organizzazione o un progetto, Azure DevOps crea automaticamente gruppi di sicurezza predefiniti e assegna permessi di default. I permessi effettivi possono dipendere anche dall’appartenenza ai gruppi. Microsoft documenta sia l’esistenza dei gruppi di sicurezza predefiniti sia la possibilità di modificare i permessi a livello di progetto da **Project settings > Permissions**. ([Microsoft Learn][3])

Per il corso, è importante almeno questo:

* se il partecipante crea da solo organizzazione e progetto, in genere avrà i permessi necessari di base
* se invece entra in una organizzazione creata da altri, potrebbe non avere tutti i permessi richiesti per creare pipeline o service connection

---

# 9. Problemi comuni

## 9.1 Non riesco a creare l’organizzazione

Possibili cause:

* account non ancora registrato correttamente
* conflitto tra accesso GitHub e account aziendale
* browser o sessione autenticata in modo ambiguo

Microsoft segnala che, se l’email GitHub è associata a una organizzazione collegata a Microsoft Entra ID, conviene usare l’account Entra ID invece dell’accesso GitHub. ([Microsoft Learn][4])

## 9.2 Non vedo il comando per creare il progetto

Possibili cause:

* non sei nella home dell’organizzazione corretta
* non hai permessi sufficienti
* stai entrando come utente invitato con diritti limitati

## 9.3 Entro nell’organizzazione ma non posso fare nulla

Possibile causa:

* il tuo utente non ha i permessi necessari sul progetto

Microsoft documenta che i permessi di progetto possono essere rivisti e modificati in **Project settings > Permissions**. ([Microsoft Learn][5])

---

# 10. Nomi consigliati per il corso

Per evitare caos durante i laboratori, conviene adottare nomi coerenti.

## 10.1 Organizzazione

Esempi:

* `academy-observability-nome`
* `corso-devops-nome`
* `observability-lab-nome`

## 10.2 Progetto

Esempio consigliato:

* `Observability-DevOps`

Questo aiuta molto quando il docente deve seguire i partecipanti e controllare che stiano lavorando nell’ambiente giusto.

---

# 11. Checklist finale

Prima di iniziare LAB01, verifica che tutte queste condizioni siano vere:

* [ ] ho un account valido per accedere ad Azure DevOps
* [ ] ho creato o posso aprire una organizzazione Azure DevOps
* [ ] conosco la URL della mia organizzazione
* [ ] ho creato o posso aprire il progetto `Observability-DevOps`
* [ ] riesco a entrare in **Project settings**
* [ ] riesco a entrare in **Pipelines**

Se tutte le caselle sono vere, sei pronto per LAB01.

---



[1]: https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/create-organization?view=azure-devops&utm_source=chatgpt.com "Create an organization - Azure DevOps"
[2]: https://learn.microsoft.com/en-us/azure/devops/organizations/projects/create-project?view=azure-devops&utm_source=chatgpt.com "Create a project - Azure DevOps"
[3]: https://learn.microsoft.com/en-us/azure/devops/organizations/security/about-permissions?view=azure-devops&utm_source=chatgpt.com "About permissions and security groups - Azure DevOps"
[4]: https://learn.microsoft.com/en-us/azure/devops/repos/get-started/sign-up-invite-teammates?view=azure-devops&utm_source=chatgpt.com "Sign up for Azure Repos - Azure DevOps"
[5]: https://learn.microsoft.com/en-us/azure/devops/organizations/security/change-project-level-permissions?view=azure-devops&utm_source=chatgpt.com "Change Project-Level Permissions or Group Membership"
