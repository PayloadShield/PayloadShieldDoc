# Encryption and Cross-Language Communication

PayloadShield is an application-level payload format shared by ComPyPS (Python) and ComPHPPS (PHP). A Python client can encrypt a request that a PHP Laravel or Symfony endpoint decrypts, and PHP can encrypt data that Python Django, FastAPI, or Flask decrypts, provided both sides use a compatible handler, matching key material, and the same data shape.

## What happens to a payload

1. The application value is serialized as JSON.
2. The selected handler encodes or encrypts those JSON bytes. Authenticated encryption uses a fresh nonce and an authentication tag; hybrid handlers also create an ephemeral symmetric key or key-agreement value.
3. The handler returns a Base64 string (some asymmetric handlers Base64-encode a JSON bundle containing their fields).
4. The framework integration sends or receives that string in a JSON `encrypted` envelope. Request decryption happens before the view/controller; response encryption happens after it returns JSON.

The `base64` handler only encodes data and provides no confidentiality or integrity. Use TLS (`https://`) in production even when payload encryption is enabled. Payload encryption does not authenticate users, prevent replay, or replace access control.

## Built-in handlers

| Handler | Mechanism | Configuration | Cross-language notes |
|---|---|---|---|
| `base64` | Base64 over JSON/text | None | Interoperable as encoding, not encryption. Use JSON objects/arrays for predictable values. |
| `fernet` | Fernet token: AES-CBC and HMAC | `Key` | Use the same canonical URL-safe Base64 Fernet key on Python and PHP. Avoid a raw 32-byte key: the implementations normalize that form differently. |
| `aes-gcm-256` | AES-256-GCM | `Key`, exactly 32 decoded bytes | Matching format: Base64 of `nonce (12 bytes) || ciphertext || tag (16 bytes)`. |
| `chacha20-poly1305` | ChaCha20-Poly1305 | `Key`, exactly 32 decoded bytes | Matching format: Base64 of `nonce (12 bytes) || ciphertext || tag (16 bytes)`. |
| `rsa-hybrid` | RSA-OAEP-SHA256 wraps a random AES-256-GCM key | RSA public key for encryption; private key for decryption | Matching bundle and OAEP parameters in ComPyPS and ComPHPPS. Use matching RSA PEM key pairs. |
| `ecdh-aes-gcm` | Ephemeral P-256 ECDH, HKDF-SHA256, AES-256-GCM | EC public key for encryption; private key for decryption | Matching bundle, HKDF context, and P-256 keys. |
| `ecies` | Ephemeral P-256 ECDH, HKDF-SHA256, AES-256-CTR, HMAC-SHA256 | EC public key for encryption; private key for decryption | Matching bundle, HKDF context, and P-256 keys. |
| `hpke` | RFC 9180 base mode: X25519/HKDF-SHA256/ChaCha20-Poly1305 | X25519 public key for encryption; private key for decryption | Matching suite and bundle in both implementations; use matching X25519 PEM keys. |

For AES-GCM and ChaCha, a Base64 representation of 32 random bytes avoids ambiguity when transporting the key through environment variables. Both libraries accept that representation. Never reuse nonces manually; the built-in handlers generate them for each message.

## Cross-language API example: Python client to PHP server

Configure the Laravel/Symfony server and Python client with the same 32-byte AES key (represented here as a Base64 environment value). Protect the PHP route with `crypt('aes-gcm-256')` as shown in the [Laravel](laravel.md) or [Symfony](symfony.md) guide.

```python
import os

import requests
from compyps import PayloadShieldEnc, get_handler

PayloadShieldEnc.init({"Key": os.environ["PAYLOADSHIELD_KEY"]})
handler = get_handler("aes-gcm-256")

request_payload = {"user": "alice", "action": "lookup"}
encoded_request = handler.encode(request_payload, PayloadShieldEnc.get_config())
response = requests.post(
    "https://api.example.test/api/echo",
    json={"encrypted": encoded_request},
    timeout=10,
)
response.raise_for_status()

encoded_response = response.json()["encrypted"]
decoded_response = handler.decode(encoded_response, PayloadShieldEnc.get_config())
print(decoded_response)
```

The response can be decoded in Python because the PHP endpoint uses the same AES-GCM key and wire format. For a PHP client talking to Python, reverse the roles: use `Crypto::encode()` / `Crypto::decode()` from ComPHPPS with the same handler and key, and send/read the same JSON envelope.

## Key direction and `crypt`

Symmetric handlers such as AES-GCM and ChaCha use the same secret for both directions, so a framework `crypt` route can decrypt a request and encrypt its response with that shared key. Fernet can also be used this way when the canonical Fernet key is shared.

Public-key handlers are directional: encrypt with the recipient's public key and decrypt with that recipient's private key. The framework `crypt` integrations use the configured handler configuration for both request decryption and response encryption. Do not assume a single recipient key pair automatically provides secure client/server responses. Configure each direction for its intended recipient using separate encryption/decryption operations or a custom integration, and never distribute a private key to clients. The direct ComPyPS and ComPHPPS handler APIs allow the key configuration to be selected for each operation.

## Interoperability checklist

- Use the same exact handler name and payload shape on both sides.
- Use a matching key or recipient key pair; keep private and symmetric keys secret.
- For AES-GCM/ChaCha, share the same 32 key bytes, not a separately generated key per service.
- For Fernet, share a canonical URL-safe Base64 Fernet key; do not rely on the raw-key shortcut across languages.
- Send and expect `{"encrypted":"..."}` with `Content-Type: application/json`.
- Test both request and response directions using real messages before deployment.
- Keep HTTPS enabled and apply normal authentication, authorization, input validation, and replay controls.
