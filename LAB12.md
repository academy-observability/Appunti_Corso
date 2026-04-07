---

# LAB12 – Alerting: Alert Rule + Action Group su SLI (error_rate)
Durata stimata: 4 ore

---
#  Cost Control – Gestione dei costi (OBBLIGATORIO)

## Modalità Percorso Continuativo (Aula – Ufficiale)
NON eliminare il Resource Group fino al LAB14.

## Modalità Standalone (facoltativa)
```bash
az group delete --name rg-observability-lab --yes --no-wait
````

---

# 1. Obiettivi del laboratorio

* Creare un Action Group (email)
* Creare una Alert Rule basata su query KQL
* Simulare errori e verificare lo stato “Fired”
* Documentare soglia e motivazione (evitare alert “rumorosi”)

---

# 2. Prerequisiti

* LAB11 completato (Log Analytics e KQL funzionanti)
* ACI attiva e ingestione log ok

---

# 3. Dove operare (OBBLIGATORIO)

* **Azure Portal**: Monitor → Alerts → Alert rules / Action groups
* **WSL Ubuntu**: generazione traffico errori, evidenze e git

---

# 4. Setup (WSL Ubuntu)

```bash
mkdir -p ~/course/lab12 && cd ~/course/lab12
script -a cmdlog_lab12.txt
mkdir -p docs
```

---

# 5. Action Group (Azure Portal)

Percorso:

* Cerca: **Monitor**
* Menu: **Alerts**
* Menu: **Action groups**
* **Create**

Imposta:

* Resource group: `rg-observability-lab`
* Nome: `ag-observability`
* Action: Email/SMS/Push/Voice → Email (tuo indirizzo)

Checkpoint #1:
Action Group creato.

---

# 6. Alert Rule su Log Analytics (Azure Portal)

Percorso:

* Monitor → Alerts → **Create** → **Alert rule**
* Scope: seleziona il **Log Analytics workspace** `law-observability`
* Condition: **Custom log search** (Log query)

Query (error_rate):

```kql
ContainerInstanceLog_CL
| extend status = toint(parse_json(LogEntry_s).status)
| summarize error_rate = todouble(countif(status >= 400)) / count()
```

Impostazioni consigliate (semplici):

* Threshold: `error_rate > 0.20`
* Evaluation frequency: 5 min
* Window size: 5 min
* Severity: 2 (o 3)

Actions:

* collega `ag-observability`

Checkpoint #2:
Alert rule creata e abilitata.

---

# 7. Simulazione errori (WSL Ubuntu)

Recupera IP ACI:

```bash
ACI_PUBLIC_IP=$(az container show --resource-group rg-observability-lab --name obsapp-aci --query ipAddress.ip -o tsv)
echo "$ACI_PUBLIC_IP"
```

Genera errori:

```bash
for i in {1..40}; do
  curl -s "http://$ACI_PUBLIC_IP:8000/nope" > /dev/null
done
```

Attendi 5–10 minuti (tempo di valutazione + ingestione).

Checkpoint #3:
In Portale:

* Monitor → Alerts → **Fired alerts** (o Alert history)
* Alert in stato Fired (se la soglia è stata superata)

---

# 8. Evidenza nel repository

Crea `docs/evidence_lab12.md` con:

* nome Action Group
* configurazione soglia
* screenshot alert fired / alert history
* breve spiegazione: perché quella soglia e cosa significa

---

# 9. Consegna (WSL Ubuntu)

```bash
git add docs/evidence_lab12.md
git commit -m "[LAB12] Alerting completato"
git push
```

---

# 10. Criteri di completamento

* Action group creato
* Alert rule attiva
* Alert Fired verificato
* Evidence completa


---
