# php-mqtt/client

[![Latest Stable Version](https://poser.pugx.org/php-mqtt/client/v)](https://packagist.org/packages/php-mqtt/client)
[![Total Downloads](https://poser.pugx.org/php-mqtt/client/downloads)](https://packagist.org/packages/php-mqtt/client)
[![Tests](https://github.com/php-mqtt/client/workflows/Tests/badge.svg)](https://github.com/php-mqtt/client/actions?query=workflow%3ATests)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=php-mqtt_client&metric=alert_status)](https://sonarcloud.io/dashboard?id=php-mqtt_client)
[![Maintainability Rating](https://sonarcloud.io/api/project_badges/measure?project=php-mqtt_client&metric=sqale_rating)](https://sonarcloud.io/dashboard?id=php-mqtt_client)
[![Reliability Rating](https://sonarcloud.io/api/project_badges/measure?project=php-mqtt_client&metric=reliability_rating)](https://sonarcloud.io/dashboard?id=php-mqtt_client)
[![Security Rating](https://sonarcloud.io/api/project_badges/measure?project=php-mqtt_client&metric=security_rating)](https://sonarcloud.io/dashboard?id=php-mqtt_client)
[![Vulnerabilities](https://sonarcloud.io/api/project_badges/measure?project=php-mqtt_client&metric=vulnerabilities)](https://sonarcloud.io/dashboard?id=php-mqtt_client)
[![License](https://poser.pugx.org/php-mqtt/client/license)](https://packagist.org/packages/php-mqtt/client)

[`php-mqtt/client`](https://packagist.org/packages/php-mqtt/client) was created by, and is maintained
by [Namoshek](https://github.com/namoshek).
It allows you to connect to an MQTT broker where you can publish messages and subscribe to topics.
The current implementation supports all QoS levels ([with limitations](#limitations)).

## Installation

```bash
composer require php-mqtt/client
```

This library requires PHP version 7.4 or higher.

## Usage

### Publish

A very basic publish example requires only three steps: connect, publish and close

```php
$server   = 'some-broker.example.com';
$port     = 1883;
$clientId = 'test-publisher';

$mqtt = new \PhpMqtt\Client\MqttClient($server, $port, $clientId);
$mqtt->connect();
$mqtt->publish('php-mqtt/client/test', 'Hello World!', 0);
$mqtt->disconnect();
```

If you do not want to pass a `$clientId`, a random one will be generated for you. This will basically force a clean session implicitly.

Be also aware that most of the methods can throw exceptions. The above example does not add any exception handling for brevity.

### Subscribe

Subscribing is a little more complex than publishing as it requires to run an event loop:

```php
$clientId = 'test-subscriber';

$mqtt = new \PhpMqtt\Client\MqttClient($server, $port, $clientId);
$mqtt->connect();
$mqtt->subscribe('php-mqtt/client/test', function ($topic, $message) {
    echo sprintf("Received message on topic [%s]: %s\n", $topic, $message);
}, 0);
$mqtt->loop(true);
$mqtt->disconnect();
```

While the loop is active, you can use `$mqtt->interrupt()` to send an interrupt signal to the loop.
This will terminate the loop before it starts its next iteration. You can call this method using `pcntl_signal(SIGINT, $handler)` for example:

```php
pcntl_async_signals(true);

$clientId = 'test-subscriber';

$mqtt = new \PhpMqtt\Client\MqttClient($server, $port, $clientId);
pcntl_signal(SIGINT, function (int $signal, $info) use ($mqtt) {
    $mqtt->interrupt();
});
$mqtt->connect();
$mqtt->subscribe('php-mqtt/client/test', function ($topic, $message) {
    echo sprintf("Received message on topic [%s]: %s\n", $topic, $message);
}, 0);
$mqtt->loop(true);
$mqtt->disconnect();
```

### Client Settings

As shown in the examples above, the `MqttClient` takes the server, port and client id as first, second and third parameter.
As fourth parameter, the protocol level can be passed. Currently supported is MQTT v3.1,
available as constant `MqttClient::MQTT_3_1`.
A fifth parameter allows passing a repository (currently, only a `MemoryRepository` is available by default).
Lastly, a logger can be passed as sixth parameter. If none is given, a null logger is used instead.

Example:
```php
$mqtt = new \PhpMqtt\Client\MQTTClient(
    $server, 
    $port, 
    $clientId,
    \PhpMqtt\Client\MqttClient::MQTT_3_1,
    new \PhpMqtt\Client\Repositories\MemoryRepository(),
    new Logger()
);
```

The `Logger` must implement the `Psr\Log\LoggerInterface`.

### Connection Settings

The `connect()` method of the `MQTTClient` takes two optional parameters:
1. A `ConnectionSettings` instance
2. A `boolean` flag indicating whether a clean session should be requested (a random client id does this implicitly)

Example:
```php
$mqtt = new \PhpMqtt\Client\MQTTClient($server, $port, $clientId);

$connectionSettings = new \PhpMqtt\Client\ConnectionSettings();
$mqtt->connect($connectionSettings, true);
```

The `ConnectionSettings` class provides a few settings through a fluent interface. The type itself is immutable,
and a new `ConnectionSettings` instance will be created for each added option.
This also prevents changes to the connection settings after a connection has been established.

The following is a complete list of options with their respective default:
```php
$connectionSettings = (new \PhpMqtt\Client\ConnectionSettings())
    ->setUsername(null)
    ->setPassword(null)
    ->setConnectTimeout(60)
    ->setSocketTimeout(5)
    ->setKeepAliveInterval(10)
    ->setResendTimeout(10)
    ->setLastWillTopic(null)
    ->setLastWillMessage(null)
    ->setLastWillQualityOfService(0)
    ->setRetainLastWill(false)
    ->setUseTls(false)
    ->setTlsVerifyPeer(true)
    ->setTlsVerifyPeerName(true)
    ->setTlsSelfSignedAllowed(false)
    ->setTlsCertificateAuthorityFile(null)
    ->setTlsCertificateAuthorityPath(null)
    ->setTlsClientCertificateFile(null)
    ->setTlsClientCertificateKeyFile(null)
    ->setTlsClientCertificatePassphrase(null);
```

## Features

- MQTT Versions
  - [x] v3 (just don't use v3.1 features like username & password)
  - [x] v3.1
  - [ ] v3.1.1
  - [ ] v5.0
- Transport
  - [x] TCP (unsecured)
  - [x] TLS (secured, verifies the peer using a certificate authority file)
- Connect
  - [x] Last Will
  - [x] Last Will Topic
  - [x] Last Will Message
  - [x] Required QoS
  - [x] Message Retention
  - [x] Authentication (username & password)
  - [ ] Clean Session (can be set and sent, but the client has no persistence for QoS 2 messages)
- Publish
  - [x] QoS Level 0
  - [x] QoS Level 1 (limitation: no persisted state across sessions)
  - [x] QoS Level 2 (limitation: no persisted state across sessions)
- Subscribe
  - [x] QoS Level 0
  - [x] QoS Level 1
  - [x] QoS Level 2 (limitation: no persisted state across sessions)
- Supported Message Length: unlimited _(no limits enforced, although the MQTT protocol supports only up to 256MB which one shouldn't use even remotely anyway)_
- Logging possible (`Psr\Log\LoggerInterface` can be passed to the client)
- Persistence Drivers
  - [x] In-Memory Driver
  - [ ] Redis Driver
  
## Limitations

- There is no guarantee that message identifiers are not used twice (while the first usage is still pending).
  The current implementation uses a simple counter which resets after all 65535 identifiers were used.
  This means that as long as the client isn't used to an extent where acknowledgements are open for a very long time, you should be fine.
  This also only affects QoS levels higher than 0, as QoS level 0 is a simple fire and forget mode.
- Message flows with a QoS level higher than 0 are not persisted as the default implementation uses an in-memory repository for data.
  To avoid issues with broken message flows, use the clean session flag to indicate that you don't care about old data.

## License

`php-mqtt/client` is open-sourced software licensed under the [MIT license](LICENSE.md).
