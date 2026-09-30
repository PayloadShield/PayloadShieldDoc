# FastAPI

## Install

```bash
python -m pip install fastapi_payloadshield
```

## Configure

Initialize handler keys once during application startup. Load secret values from the environment or a secrets manager.

```python
import os

from fastapi_payloadshield import PayloadShieldEnc

PayloadShieldEnc.init({
    "Key": os.environ["PAYLOADSHIELD_KEY"],
})
```

AES-GCM-256 and ChaCha20-Poly1305 require a key that resolves to exactly 32 bytes. Public-key handlers use their documented PEM fields; details are in [Encryption and cross-language communication](encryption-and-cross-language.md).

## Protect routes

```python
from fastapi import FastAPI
from fastapi_payloadshield import PayloadShield

app = FastAPI()

@app.get("/api/data")
@PayloadShield.encrypt("aes-gcm-256")
async def get_data():
    return {"message": "hello"}

@app.post("/api/receive")
@PayloadShield.decrypt("aes-gcm-256")
async def receive_data(data: dict):
    return {"received": data}

@app.post("/api/echo")
@PayloadShield.crypt("aes-gcm-256")
async def secure_echo(data: dict):
    return {"received": data}
```

Send request bodies as JSON in the `{"encrypted":"..."}` envelope. `decrypt` supplies the decoded JSON value to the route's body parameter; `crypt` also encrypts the route response. The decorator supports asynchronous route functions.

## Run tests

From the FastAPIPS project directory:

```bash
pytest
```
