# Symfony

## Install

```bash
composer require payloadshield/symfonyps
```

Symfony Flex normally registers the bundle. If it does not, add it to `config/bundles.php`:

```php
PayloadShield\SymfonyPS\PayloadShieldBundle::class => ['all' => true],
```

## Configure

Create `config/packages/payload_shield.yaml` and load secrets from Symfony's environment/secrets system:

```yaml
payload_shield:
    default_handler: aes-gcm-256
    key: '%env(PAYLOADSHIELD_KEY)%'
```

AES-GCM-256 requires a key resolving to exactly 32 bytes. RSA, EC, and HPKE handlers additionally use their corresponding PEM key options. See [Encryption and cross-language communication](encryption-and-cross-language.md).

## Protect routes

The helper returns route defaults that the bundle's event subscriber uses to select the mode and handler:

```php
use PayloadShield\SymfonyPS\PayloadShield;

$routes->add('api_data', '/api/data')
    ->controller([DataController::class, 'show'])
    ->defaults(PayloadShield::encrypt('aes-gcm-256'));

$routes->add('api_receive', '/api/receive')
    ->controller([DataController::class, 'receive'])
    ->methods(['POST'])
    ->defaults(PayloadShield::decrypt('aes-gcm-256'));

$routes->add('api_echo', '/api/echo')
    ->controller([DataController::class, 'echo'])
    ->methods(['POST'])
    ->defaults(PayloadShield::crypt('aes-gcm-256'));
```

Encrypted request bodies must contain a string `encrypted` field and decode to a JSON object or array. The subscriber decrypts before the controller and encrypts JSON responses afterward; non-JSON responses are not encrypted.

## Run tests

From the symfonyPS project directory:

```bash
composer test
```
