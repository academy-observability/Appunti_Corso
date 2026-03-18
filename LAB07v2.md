# LAB07 – Docker locale: containerizzazione completa (monorepo)

**Obiettivo:** containerizzare un servizio, gestire versioni, volumi, log e troubleshooting. Concludere il Modulo 1 con un ciclo completo di deployment locale.

---

## 1. Obiettivi
Al termine del LAB07 dovrai essere in grado di:
- Scrivere un Dockerfile
- Buildare e taggare immagini (v1, v2)
- Eseguire container con port mapping
- Leggere log container con `docker logs`
- Usare volumi per persistenza
- Usare variabili ENV
- Usare Docker Compose per ripetibilità
- Eseguire troubleshooting strutturato

---

## 2. Dove lavorare (nuova alberatura)
**Terminale:** Ubuntu (WSL)

Posizionati nella cartella del LAB07:

```bash
cd ~/corso_obs/NOME-REPOSITORY/labs/lab07
pwd
ls -la
```

Regole:
- codice in `src/`
- evidenze in `docs/`
- runtime state (volume, log locali, cmdlog) in `logs/` (ignorata da git)

---

## 3. Prerequisiti
- LAB05 completato (concetti HTTP/log)
- Docker installato e funzionante

Verifica:

```bash
docker version
docker ps
docker compose version
```

---

## 4. Setup (workspace runtime non versionato)
```bash
cd ~/corso_obs/NOME-REPOSITORY/labs/lab07
mkdir -p logs/data
```

(Opzionale) registra i comandi in un file non versionato:

```bash
script -a logs/cmdlog_lab07.txt
```

---

## 5. Mini-app Flask (src/app.py)
Copiare il codice Flask fornito in aula e creare:
- `src/app.py`
- `src/requirements.txt`
- `Dockerfile`

## Codice applicazione (src/app.py)

```python
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route("/health")
def health():
    return jsonify({"status": "ok", "service": "demo-docker"})

if __name__ == "__main__":
    port = int(os.environ.get("PORT", "8000"))
    app.run(host="0.0.0.0", port=port)

```

---

## Codice requirements.txt (src/requirements.txt)
- Aggiungere questa riga al file requirements.txt:
```

flask==3.0.3

```

---

- `Dockerfile`

Esempio Dockerfile (root del lab):

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY src/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY src/app.py .
ENV PORT=8000
EXPOSE 8000
CMD ["python","app.py"]
```

Nota: il codice Flask deve esporre `/health`.

---

## 6. Build immagine v1
```bash
cd ~/corso_obs/NOME-REPOSITORY/labs/lab07
docker build -t obsapp:v1 .
docker images | head
```

---

## 7. Run + test
```bash
docker rm -f obsapp 2>/dev/null || true
docker run -d --name obsapp -p 8000:8000 obsapp:v1
docker ps
```

Test:

```bash
curl -s http://localhost:8000/health
docker logs --tail 20 obsapp
```

---

## 8. Drill: porta sbagliata
```bash
docker rm -f obsapp
docker run -d --name obsapp -p 8080:8000 obsapp:v1
curl -s http://localhost:8000/health || echo FAIL
curl -s http://localhost:8080/health
```

Fix: rimappare correttamente.

---

## 9. Persistenza con volume (v2)
Modifica l’app per scrivere anche su `/data/app_events.log`.

Build:

```bash
docker build -t obsapp:v2 .
```

Run con volume (host = `logs/data`, ignorata da git):

```bash
docker rm -f obsapp 2>/dev/null || true
docker run -d --name obsapp -p 8000:8000 -v "$(pwd)/logs/data:/data" obsapp:v2
```

Verifica:

```bash
tail -n 5 logs/data/app_events.log
docker restart obsapp
tail -n 5 logs/data/app_events.log
```

---

## 10. ENV configurabile
```bash
docker rm -f obsapp
docker run -d --name obsapp -p 9000:9000 -e PORT=9000 obsapp:v2
curl -s http://localhost:9000/health
```

---

## 11. Docker Compose
Creare `docker-compose.yml` (root del lab):

```yaml
services:
  obsapp:
    image: obsapp:v2
    container_name: obsapp
    environment:
      - PORT=8000
    ports:
      - "8000:8000"
    volumes:
      - ./logs/data:/data
    restart: unless-stopped
```

Comandi:

```bash
docker compose up -d
docker compose ps
docker compose logs --tail 20
docker compose down
```

---

## 12. Versioning & rollback
1. Modifica endpoint `/health` aggiungendo `"version":"v2"` (o simile).
2. Rebuild `obsapp:v2` e riavvia.
3. Rollback a `obsapp:v1` per dimostrazione.

---

## 13. Troubleshooting obbligatorio (2 scenari)
Scegli 2:
- Porta già in uso
- Volume non montato
- Container crash
- ENV mancante
- Health non risponde

Per ciascuno documentare:
- Sintomo
- Ipotesi
- Test
- Causa
- Fix
- Evidenza

---

## 14. Evidenza nel repository
Creare:

```
docs/evidence_lab07.md
```

Struttura:

```markdown
# LAB07 – Evidence (Docker)

## Build & run
## Log container
## Persistenza volume
## Compose up/down
## Versioning v1 ↔ v2
## Troubleshooting (2 scenari)
## Conclusione: cosa cambia passando ad Azure
```

---

## 15. Consegna
Se hai avviato `script`, chiudilo con `exit`.

Poi:

```bash
cd ~/corso_obs/NOME-REPOSITORY/labs/lab07
git status
git add Dockerfile docker-compose.yml src docs/evidence_lab07.md
git commit -m "[LAB07] Docker locale completato"
git push
```

---

## 16. Criteri di completamento
LAB07 è completato quando:
- `obsapp:v1` e `obsapp:v2` sono buildate
- `/health` risponde
- volume persistente verificato (file in `logs/data/app_events.log`)
- compose funziona (up/down/logs)
- 2 troubleshooting documentati
- report presente su GitHub
