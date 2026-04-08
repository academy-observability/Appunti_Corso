
```python
from flask import Flask, jsonify, request, abort, g
import os
import time
import uuid
import logging
import json

app = Flask(__name__)

# Disabilitiamo i log standard per vedere solo i nostri
log = logging.getLogger('werkzeug')
log.disabled = True

@app.before_request
def start_timer():
    g.start = time.time()
    g.request_id = request.headers.get('X-Request-Id', str(uuid.uuid4()))

@app.after_request
def log_request(response):
    duration_ms = int((time.time() - g.start) * 1000)
    
    # Inserisce l'header di risposta
    response.headers['X-Request-Id'] = g.request_id
    
    # Crea il log strutturato
    log_data = {
        "request_id": g.request_id,
        "path": request.path,
        "status": response.status_code,
        "duration_ms": duration_ms
    }
    print(json.dumps(log_data), flush=True)
    
    return response

@app.route("/health")
def health():
    return jsonify({"status": "ok", "service": "demo-docker"})

@app.route("/time")
def get_time():
    return jsonify({"time": time.strftime("%Y-%m-%d %H:%M:%S")})

@app.route("/nope")
def nope():
    abort(404)

if __name__ == "__main__":
    port = int(os.environ.get("PORT", "8000"))
    app.run(host="0.0.0.0", port=port)

```
