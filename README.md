# PayloadShield Documentation

PayloadShield provides request/response payload encryption integrations for Django, FastAPI, Flask, Laravel, and Symfony. The Python integrations use ComPyPS; the PHP integrations use ComPHPPS.

## Index

1. [Django](guides/django.md)
2. [FastAPI](guides/fastapi.md)
3. [Flask](guides/flask.md)
4. [Laravel](guides/laravel.md)
5. [Symfony](guides/symfony.md)
6. [Encryption and cross-language communication](guides/encryption-and-cross-language.md)

## Shared API envelope

For encrypted requests, send JSON with a string `encrypted` property:

```json
{"encrypted":"<handler output>"}
```

For encrypted responses, the integrations return the same envelope. The encrypted bytes encode a JSON object or array; the selected handler and its key configuration must match at both ends. Use HTTPS as well: payload encryption does not replace transport security, authentication, authorization, or replay protection.

## Choose an integration

| Framework | Package | Integration style | Guide |
|---|---|---|---|
| Django | `django_payloadshield` | View decorators | [Django](guides/django.md) |
| FastAPI | `fastapi_payloadshield` | Route decorators | [FastAPI](guides/fastapi.md) |
| Flask | `flask-payloadshield` | View decorators | [Flask](guides/flask.md) |
| Laravel | `payloadshield/laravelps` | Route middleware | [Laravel](guides/laravel.md) |
| Symfony | `payloadshield/symfonyps` | Route defaults and event subscriber | [Symfony](guides/symfony.md) |
