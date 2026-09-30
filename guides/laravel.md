# Laravel

## Install

```bash
composer require payloadshield/laravelps
php artisan vendor:publish --tag=payloadshield-config
```

## Configure

Set the default handler and key in `.env`; do not commit real secrets. For AES-GCM-256, the key must resolve to exactly 32 bytes. Other handlers require the corresponding RSA, EC, or X25519 PEM keys.

```dotenv
PAYLOADSHIELD_DEFAULT=aes-gcm-256
PAYLOADSHIELD_KEY=<base64-encoded-32-byte-secret>
```

The config published by the package reads these values. See [Encryption and cross-language communication](encryption-and-cross-language.md) for shared keys and public-key usage.

## Protect routes

The static helper can be used with route definitions:

```php
use Illuminate\Support\Facades\Route;
use PayloadShield\LaravelPS\PayloadShield;

Route::get('/api/data', fn () => ['message' => 'hello'])
    ->middleware(PayloadShield::encrypt('aes-gcm-256'));

Route::post('/api/receive', [DataController::class, 'receive'])
    ->middleware(PayloadShield::decrypt('aes-gcm-256'));

Route::post('/api/echo', [DataController::class, 'echo'])
    ->middleware(PayloadShield::crypt('aes-gcm-256'));
```

The `decrypt` middleware replaces the encrypted body with its decoded JSON value before the controller runs, so controllers can read it through `$request->all()`. `encrypt` wraps a JSON response; `crypt` does both. The package also registers `payloadshield.encrypt`, `payloadshield.decrypt`, and `payloadshield.crypt` middleware aliases.

## Run tests

From the LaravelPS project directory:

```bash
vendor/bin/phpunit
```
