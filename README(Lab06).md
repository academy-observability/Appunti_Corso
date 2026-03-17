# LAB06 – Observability base: log JSON → SLI minimi (monorepo)

**Obiettivo:** trasformare i log JSON del mini-servizio (LAB05) in indicatori operativi (SLI) e produrre un report tecnico ripetibile.

---

## 1. Obiettivi
Al termine del LAB06 dovrai essere in grado di:
- Leggere e filtrare log JSON
- Calcolare indicatori operativi (SLI):
  - total_requests
  - error_rate
  - latency p50
  - latency p95
  - top_paths
- Correlare request-id → riga di log
- Interpretare i numeri e proporre azioni

---

## 2. Dove lavorare (nuova alberatura)
**Terminale:** Ubuntu (WSL)

Posizionati nella cartella del LAB06:

```bash
cd ~/corso_obs/NOME-REPOSITORY/labs/lab06
pwd
ls -la
```

File:
- script e parser in `src/`
- evidenze in `docs/`
- log di lavoro in `logs/` (ignorati da git)
- output versionabile: `metrics.json` (in root del lab)

---

## 3. Prerequisiti
- LAB05 completato (servizio + log disponibili)
- python3 installato
- curl disponibile
- Linux base (grep, wc, sort, uniq)

Verifica:

```bash
python3 --version
which python3 curl grep wc sort uniq
```

---

## 4. Setup (log di lavoro + copia log da LAB05)

### 4.1 Assicurati che il servizio del LAB05 stia girando (Terminale 1)
In un terminale separato:

```bash
cd ~/corso_obs/NOME-REPOSITORY/labs/lab05
PORT=9000 LOG_PATH="logs/app.log" python3 src/app.py
```

### 4.2 Prepara LAB06 e copia il log (Terminale 2)
```bash
cd ~/corso_obs/NOME-REPOSITORY/labs/lab06
mkdir -p logs
cp ../lab05/logs/app.log logs/app.log 2>/dev/null || echo "logs/app.log non trovato: avvia prima il LAB05 e genera traffico"
```

Verifica:

```bash
wc -l logs/app.log || true
head -n 2 logs/app.log || true
tail -n 2 logs/app.log || true
```

---

## 5. Genera traffico controllato (per aumentare il log del LAB05)
Crea `src/traffic.sh`:

```bash
cat > src/traffic.sh << 'EOF'
#!/bin/bash
set -e
BASE="http://localhost:9000"
N=${1:-40}

for i in $(seq 1 $N); do
  curl -s "$BASE/health" > /dev/null
  curl -s "$BASE/time" > /dev/null

  if (( i % 5 == 0 )); then
    curl -s "$BASE/nope" > /dev/null
  fi

  if (( i % 7 == 0 )); then
    curl -s -X POST "$BASE/echo" -H 'Content-Type: application/json' -d '{"msg":' > /dev/null || true
  else
    curl -s -X POST "$BASE/echo" -H 'Content-Type: application/json' -d '{"msg":"ciao"}' > /dev/null
  fi

  sleep 0.05
done
EOF

chmod 755 src/traffic.sh
```

Esegui:

```bash
cd ~/corso_obs/NOME-REPOSITORY/labs/lab06
src/traffic.sh 40
```

Dopo il traffico, ricopia il log aggiornato dal LAB05:

```bash
cp ../lab05/logs/app.log logs/app.log
wc -l logs/app.log
grep '"status": 404' logs/app.log | head -n 2
grep '"status": 400' logs/app.log | head -n 2
```

---

## 6. Parser log → metrics.json
Crea `src/parse_logs.py` (copiando il codice fornito in aula).

**Requisito:** il parser deve leggere `logs/app.log` e produrre `metrics.json` (nella root del LAB06).

Esegui:

```bash
cd ~/corso_obs/NOME-REPOSITORY/labs/lab06
python3 src/parse_logs.py
cat metrics.json
```

`metrics.json` deve contenere:
- total_requests
- error_rate
- latency_p50_ms
- latency_p95_ms
- top_paths
- status_breakdown
- sample_request_ids (almeno 2)

---

## 7. Correlazione request-id → log
Estrai 2 request_id:

```bash
python3 -c "import json; print('\n'.join(json.load(open('metrics.json'))['sample_request_ids'][:2]))"
```

Per ciascuno:

```bash
grep "<RID>" logs/app.log
```

---

## 8. Evidenza nel repository
Creare:

```
docs/evidence_lab06.md
```

Struttura obbligatoria:

```markdown
# LAB06 – Evidence

## Baseline (dimensione log)
(righe totali + 2 righe di esempio)

## Traffico generato
(comando eseguito + conferma errori 404/400 presenti)

## Indicatori (SLI)
- total_requests
- error_rate
- p50 / p95
- top_paths
- status_breakdown

## Correlazione request-id (2 esempi)
(RID + riga di log)

## Interpretazione operativa
- Cosa significa error_rate?
- Cosa significa p95?
- Quale metrica aggiungerei?
- Che alert imposterei?

## Note finali
```

---

## 9. Consegna
```bash
cd ~/corso_obs/NOME-REPOSITORY/labs/lab06
git status
git add src/traffic.sh src/parse_logs.py metrics.json docs/evidence_lab06.md
git commit -m "[LAB06] SLI e analisi log completato"
git push
```

---

## 10. Criteri di completamento
Il LAB06 è completato quando:
- `metrics.json` contiene indicatori validi
- `error_rate > 0` (perché hai indotto errori)
- p50 e p95 sono valorizzati
- 2 correlazioni request-id funzionano su `logs/app.log`
- report interpretativo presente in `docs/evidence_lab06.md`

