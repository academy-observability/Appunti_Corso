
# script parse_logs.py
versione **semplificata e spiegata riga per riga**



---

```python
#!/usr/bin/env python3
# ↑ indica al sistema che questo file va eseguito con Python3

import json
# ↑ serve per leggere e scrivere file JSON

from pathlib import Path
# ↑ serve per gestire i percorsi dei file in modo pulito

# -----------------------------
# 1. Definizione percorsi file
# -----------------------------

LOG_PATH = Path("logs/app.log")
# ↑ percorso del file di log (input)

OUT_PATH = Path("metrics.json")
# ↑ file di output dove scriveremo le metriche


# -----------------------------
# 2. Funzione per calcolare percentili
# -----------------------------

def percentile(values, p):
    """
    values = lista di numeri (es. latenze)
    p = percentile (50 o 95)
    """

    if not values:
        # ↑ se la lista è vuota non possiamo calcolare nulla
        return None

    values = sorted(values)
    # ↑ ordina i valori dal più piccolo al più grande

    index = int(len(values) * p / 100)
    # ↑ calcolo posizione percentile (approssimato)

    # protezione per non uscire fuori indice
    index = min(index, len(values) - 1)

    return values[index]
    # ↑ restituisce il valore percentile


# -----------------------------
# 3. Funzione principale
# -----------------------------

def main():

    # controllo se il file esiste
    if not LOG_PATH.exists():
        print("Errore: file log non trovato")
        return

    # lista che conterrà tutte le righe valide del log
    records = []

    # contatore righe non valide (JSON rotto)
    bad_lines = 0

    # -----------------------------
    # 4. Lettura file log
    # -----------------------------

    with LOG_PATH.open("r", encoding="utf-8") as f:
        # ↑ apre il file in lettura

        for line in f:
            # ↑ legge il file riga per riga

            line = line.strip()
            # ↑ rimuove spazi e newline

            if not line:
                # ↑ salta righe vuote
                continue

            try:
                obj = json.loads(line)
                # ↑ converte la riga JSON in oggetto Python

                records.append(obj)
                # ↑ salva il record nella lista

            except json.JSONDecodeError:
                # ↑ se la riga non è JSON valido
                bad_lines += 1

    # -----------------------------
    # 5. Calcolo metriche base
    # -----------------------------

    total_requests = len(records)
    # ↑ numero totale richieste

    error_count = 0
    # ↑ contatore errori

    durations = []
    # ↑ lista delle latenze

    path_counter = {}
    # ↑ conteggio richieste per endpoint

    status_counter = {}
    # ↑ conteggio per status HTTP

    request_ids = []
    # ↑ lista request_id per correlazione

    # -----------------------------
    # 6. Analisi di ogni record
    # -----------------------------

    for r in records:

        status = r.get("status")
        path = r.get("path")
        duration = r.get("duration_ms")
        request_id = r.get("request_id")

        # ---- status ----
        if status is not None:

            status_str = str(status)

            # aggiorna conteggio status
            status_counter[status_str] = status_counter.get(status_str, 0) + 1

            # conta errori (>=400)
            if isinstance(status, int) and status >= 400:
                error_count += 1

        # ---- path ----
        if path:
            path_counter[path] = path_counter.get(path, 0) + 1

        # ---- durata ----
        if isinstance(duration, (int, float)):
            durations.append(duration)

        # ---- request_id ----
        if request_id:
            request_ids.append(request_id)

    # -----------------------------
    # 7. Calcolo indicatori
    # -----------------------------

    if total_requests > 0:
        error_rate = round(error_count / total_requests, 4)
    else:
        error_rate = 0.0

    p50 = percentile(durations, 50)
    p95 = percentile(durations, 95)

    # top 5 endpoint più usati
    top_paths = sorted(path_counter.items(), key=lambda x: x[1], reverse=True)[:5]

    # trasformiamo in formato leggibile
    top_paths = [{"path": p, "count": c} for p, c in top_paths]

    # -----------------------------
    # 8. Costruzione output finale
    # -----------------------------

    metrics = {
        "total_requests": total_requests,
        "error_rate": error_rate,
        "latency_p50_ms": p50,
        "latency_p95_ms": p95,
        "top_paths": top_paths,
        "status_breakdown": status_counter,
        "sample_request_ids": request_ids[:5],
        "bad_lines_skipped": bad_lines
    }

    # -----------------------------
    # 9. Scrittura file output
    # -----------------------------

    with OUT_PATH.open("w", encoding="utf-8") as f:
        json.dump(metrics, f, indent=2)
        # ↑ salva il JSON formattato

    print("File metrics.json creato")
    print(json.dumps(metrics, indent=2))
    # ↑ stampa a video per verifica


# -----------------------------
# 10. Avvio script
# -----------------------------

if __name__ == "__main__":
    main()
```

---

#  Conclusione

Questo script fa esattamente questo:

```text
1. legge il file logs/app.log
2. interpreta ogni riga JSON
3. conta richieste, errori, endpoint
4. raccoglie tempi di risposta
5. calcola p50 e p95
6. crea metrics.json
```

---

# Key Concept

```text
log = eventi singoli
metrics = visione del sistema
```

