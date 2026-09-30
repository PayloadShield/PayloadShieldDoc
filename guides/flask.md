# Flask

## Install

```bash
python -m pip install flask-payloadshield
```

## Configure

Initialize keys once before handling requests. Read keys from environment variables or a secrets manager.

```python
import os

from flask_payloadshield import PayloadShieldEnc

PayloadShieldEnc.init({
    "Key": os.environ["PAYLOADSHIELD_KEY"],
})
```

AES-GCM-256 and ChaCha20-Poly1305 require exactly 32 key bytes. Public-key PEM options and cross-language details are covered in [Encryption and cross-language communication](encryption-and-cross-language.md).

## Protect views

```python
from flask import Flask
from flask_payloadshield import PayloadShield

app = Flask(__name__)

@app.get("/api/data")
@PayloadShield.encrypt("aes-gcm-256")
def get_data():
    return {"message": "hello"}

@app.post("/api/receive")
@PayloadShield.decrypt("aes-gcm-256")
def receive_data(data: dict):
    return {"received": data}

@app.post("/api/echo")
@PayloadShield.crypt("aes-gcm-256")
def secure_echo(data: dict):
    return {"received": data}
```

Encrypted requests use a JSON `{"encrypted":"..."}` body. The `decrypt` decorator injects the decoded value into the view argument; `crypt` also encrypts the JSON response. Flask PayloadShield routes are synchronous.

## Run tests

From the FlaskPS project directory:

```bash
pytest
```
