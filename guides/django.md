# Django

## Install

```bash
python -m pip install django_payloadshield
```

## Configure

Initialize the handler keys once before serving requests, such as from Django app startup. Keep secrets in environment variables or a secrets manager, not in source control.

```python
import os

from django_payloadshield import PayloadShieldEnc

PayloadShieldEnc.init({
    "Key": os.environ["PAYLOADSHIELD_KEY"],
})
```

For AES-GCM-256 or ChaCha20-Poly1305, `Key` must resolve to exactly 32 bytes. RSA, EC, and HPKE handlers use their corresponding public/private PEM configuration fields. See [Encryption and cross-language communication](encryption-and-cross-language.md).

## Protect views

```python
from django.http import JsonResponse
from django.urls import path
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_POST
from django_payloadshield import PayloadShield

@PayloadShield.encrypt("aes-gcm-256")
def get_data(request):
    return {"message": "hello"}

@csrf_exempt
@require_POST
@PayloadShield.decrypt("aes-gcm-256")
def receive_data(request):
    return JsonResponse({"received": request.decrypted_data})

@csrf_exempt
@require_POST
@PayloadShield.crypt("aes-gcm-256")
def secure_echo(request):
    return {"received": request.decrypted_data}

urlpatterns = [
    path("api/data", get_data),
    path("api/receive", receive_data),
    path("api/echo", secure_echo),
]
```

`encrypt` wraps the view's JSON-able return value as `{"encrypted":"..."}`. `decrypt` reads that envelope and makes the decoded value available as `request.decrypted_data`; `crypt` does both. Restrict request methods as appropriate and retain the application's normal CSRF protections for browser-session endpoints.

## Run tests

From the DjangoPS project directory:

```bash
pytest
```
