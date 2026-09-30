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

## Step-by-step encryption flows

Each flow starts with the application value and ends with the recipient's decoded value. The diagrams show one-way delivery; for a response, the sender and recipient roles reverse. Base64 in these formats makes binary bytes transportable as text; it does not itself add security.

### `base64`

1. The handler serializes an object or array as JSON and encodes its UTF-8 bytes with standard Base64. Other inputs are converted to text.
2. The recipient Base64-decodes the text and parses it as JSON. Use objects or arrays for predictable cross-language round trips.
3. No key, encryption, or integrity check is involved; anyone who gets the value can decode it.

```text
Sender / handler                         Recipient / handler
    |                                           |
    | JSON or text -> UTF-8 -> Base64           |
    |------------- Base64 text ---------------->|
    |                                           | Base64-decode
    |                                           | Parse JSON when valid
    |                                           | Return decoded value
```

### `fernet`

1. The handler serializes the value as UTF-8 JSON and splits the shared 32-byte Fernet key into a signing key and an encryption key.
2. It generates a timestamp and random 16-byte IV, encrypts with AES-128-CBC, and computes HMAC-SHA256 over the version, timestamp, IV, and ciphertext.
3. It concatenates those token fields and emits URL-safe Base64. The recipient decodes the token, checks its version and HMAC, decrypts the ciphertext, then parses the JSON.

```text
Sender / handler                         Recipient / handler
    |                                           |
    | JSON -> AES-128-CBC + random IV           |
    | HMAC(version|time|IV|ciphertext)          |
    | Build Fernet token                        |
    |------------ URL-safe Base64 ------------->|
    |                                           | Decode token fields
    |                                           | Verify HMAC
    |                                           | Decrypt -> parse JSON
```

### `aes-gcm-256`

1. The handler resolves `Key` to exactly 32 bytes, serializes the value as UTF-8 JSON, and generates a fresh 12-byte nonce.
2. AES-256-GCM encrypts and authenticates the JSON, producing ciphertext and a 16-byte tag.
3. The wire value is standard Base64 of `nonce || ciphertext || tag`. The recipient splits those fields; GCM verifies the tag as it decrypts, and only then is the plaintext parsed as JSON.

```text
Sender / handler                         Recipient / handler
    |                                           |
    | JSON -> AES-256-GCM(key, random nonce)    |
    | Build nonce|ciphertext|tag                |
    |------------- Base64 value --------------->|
    |                                           | Decode and split fields
    |                                           | Verify tag + decrypt
    |                                           | Parse JSON
```

### `chacha20-poly1305`

1. The handler resolves `Key` to exactly 32 bytes, serializes the value as UTF-8 JSON, and generates a fresh 12-byte nonce.
2. ChaCha20-Poly1305 encrypts and authenticates the JSON, producing ciphertext and a 16-byte tag.
3. The wire value is standard Base64 of `nonce || ciphertext || tag`. The recipient splits those fields, verifies the tag while decrypting, and parses the authenticated plaintext as JSON.

```text
Sender / handler                         Recipient / handler
    |                                           |
    | JSON -> ChaCha20-Poly1305(key, nonce)     |
    | Build nonce|ciphertext|tag                |
    |------------- Base64 value --------------->|
    |                                           | Decode and split fields
    |                                           | Verify tag + decrypt
    |                                           | Parse JSON
```

### `rsa-hybrid`

1. The sender generates a random 32-byte AES key and a fresh 12-byte nonce, then encrypts the UTF-8 JSON with AES-256-GCM.
2. It wraps the AES key using the recipient's RSA public key with OAEP and SHA-256. The Base64-encoded fields `key`, `nonce`, and `data` (ciphertext plus tag) are placed in a JSON bundle; the complete bundle is Base64-encoded.
3. The recipient decodes the bundle, unwraps the AES key with its RSA private key, then verifies/decrypts the GCM data and parses the JSON.

```text
Sender / handler                         Recipient / handler
    |                                           |
    | Random AES key -> AES-GCM JSON            |
    | RSA-OAEP-SHA256 wraps AES key             |
    | Bundle key|nonce|ciphertext+tag           |
    |----------- Base64(JSON bundle) ---------->|
    |                                           | Decode bundle
    |                                           | RSA private key unwraps key
    |                                           | Verify/decrypt -> parse JSON
```

### `ecdh-aes-gcm`

1. The sender creates a fresh P-256 ephemeral key pair and performs ECDH between its ephemeral private key and the recipient's static public key.
2. HKDF-SHA256 derives a 32-byte AES key using the `ecdh-aes-gcm` context. AES-256-GCM encrypts the UTF-8 JSON with a fresh 12-byte nonce.
3. The handler Base64-encodes a JSON bundle containing the ephemeral public key, nonce, and ciphertext plus tag. The recipient uses its static private key and the included ephemeral public key to derive the same AES key, verifies/decrypts, then parses the JSON.

```text
Sender / handler                         Recipient / handler
    |                                           |
    | Ephemeral P-256 key pair                  |
    | ECDH(ephemeral private, recipient public) |
    | HKDF-SHA256 -> AES key -> AES-GCM          |
    | Bundle ephemeral public|nonce|data        |
    |----------- Base64(JSON bundle) ---------->|
    |                                           | ECDH(recipient private, ephemeral public)
    |                                           | HKDF -> same AES key
    |                                           | Verify/decrypt -> parse JSON
```

### `ecies`

1. The sender creates a fresh P-256 ephemeral key pair and derives an ECDH shared secret with the recipient's static public key.
2. HKDF-SHA256 with context `ecies-encryption` derives separate 32-byte encryption and MAC keys. AES-256-CTR encrypts the UTF-8 JSON using a random 16-byte IV; HMAC-SHA256 authenticates `IV || ciphertext`.
3. The Base64-encoded JSON bundle contains the ephemeral public key, IV, ciphertext, and tag. The recipient derives the same keys, verifies the HMAC before decrypting, then parses the JSON.

```text
Sender / handler                         Recipient / handler
    |                                           |
    | Ephemeral P-256 key pair                  |
    | ECDH -> HKDF -> AES key + MAC key         |
    | AES-CTR JSON; HMAC(IV|ciphertext)         |
    | Bundle ephemeral public|IV|data|tag       |
    |----------- Base64(JSON bundle) ---------->|
    |                                           | ECDH -> HKDF -> same keys
    |                                           | Verify HMAC before decrypting
    |                                           | AES-CTR decrypt -> parse JSON
```

### `hpke`

1. The sender creates a fresh X25519 encapsulation key pair and performs the RFC 9180 DHKEM operation with the recipient's static X25519 public key.
2. The HPKE HKDF-SHA256 key schedule derives a one-message ChaCha20-Poly1305 key and base nonce. The handler seals the UTF-8 JSON in base mode with no additional authenticated data.
3. The Base64-encoded JSON bundle contains `enc` (the ephemeral public key) and `data` (ciphertext plus tag). The recipient repeats DHKEM and the key schedule using its private key, opens the ciphertext, and parses the authenticated JSON.

```text
Sender / handler                         Recipient / handler
    |                                           |
    | Ephemeral X25519 key pair                 |
    | DHKEM -> shared secret                   |
    | HPKE HKDF schedule -> key + base nonce   |
    | ChaCha20-Poly1305 seal(JSON)              |
    | Bundle enc|data                           |
    |----------- Base64(JSON bundle) ---------->|
    |                                           | DHKEM -> same shared secret
    |                                           | HPKE schedule -> same key/nonce
    |                                           | Open + verify -> parse JSON
```

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
