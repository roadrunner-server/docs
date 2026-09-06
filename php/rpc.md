# RPC to RoadRunner

RoadRunner provides a powerful RPC (Remote Procedure Call) interface for communication between PHP applications and the
server using the [Goridge library](https://github.com/roadrunner-php/goridge).

## Goridge

Goridge is a high-performance PHP-to-Go/Go-to-PHP library developed specifically for communication between PHP
applications and RoadRunner. It is designed to provide a reliable and efficient way to communicate between the two
components, allowing PHP developers to take advantage of the performance benefits of Golang-based systems while still
writing their applications in PHP.

## Installation

To use Goridge, you first need to install it via Composer.

{% code %}

```bash
composer require spiral/goridge
```

{% endcode %}

## v6 compatibility

The v6 plugin beta still uses Goridge with Go `net/rpc`. Keep the existing TCP or Unix listener address and `plugin.Method` calls. No Connect client migration is required.

The Go transport, `goridge/v4 v4.0.0-beta.3`, no longer supports MessagePack. PHP clients that select a MessagePack codec must switch to a supported codec, even if their PHP Goridge version still offers MessagePack. The server supports JSON, protobuf, Gob, and raw bytes. Use JSON for JSON RPC arguments and protobuf for methods that accept protobuf DTOs.

The Go module version does not change the frame protocol version. The base header, payload-length encoding, CRC calculation, and frame version remain compatible. This does not add a new frame-size limit or make MessagePack calls compatible.

RoadRunner protobuf source, generated Go bindings, and Go plugin contracts now have separate repositories. This relocation alone does not require a PHP worker-loop rewrite. See [plugin migration](../customization/plugin.md#v6-migration) for the Go imports and DTO exceptions.

## Configuration

You can change the RPC port from the default (`127.0.0.1:6001`) using the following configuration:

{% code title=".rr.yaml" %}

```yaml
version: "3"

rpc:
  listen: tcp://127.0.0.1:6001
```

{% endcode %}

### Development: Unix Socket

The development RPC plugin can set [Unix socket attributes](../intro/config.md#unix-socket-attributes) independently of worker credentials:

{% code title=".rr.yaml fragment" %}

```yaml
rpc:
  listen: "unix:///run/roadrunner/rpc.sock"
  unix_socket:
    mode: "0600"
```

{% endcode %}

Direct clients must use the same listener address. PHP workers can read it through the environment-based client shown below. Keep RPC permissions separate from sockets used by a web server.

## Connecting to RoadRunner

Once you have installed Goridge, you can connect to the RoadRunner server. To do so, create an instance of
the `Spiral\Goridge\RPC\RPC`.

**Here's an example:**

{% code title="worker.php" %}

```php
<?php

use Spiral\Goridge;
require "vendor/autoload.php";

$rpc = new Goridge\RPC\RPC(
    Goridge\Relay::create('tcp://127.0.0.1:6001')
);
```

{% endcode %}

Or you can use the `Spiral\RoadRunner\Environment` class to get the RPC address from environment variables:

{% code title="worker.php" %}

```php
<?php

use Spiral\Goridge;
use Spiral\RoadRunner\Environment;
require "vendor/autoload.php";

$address = Environment::fromGlobals()->getRPCAddress();
$rpc = new Goridge\RPC\RPC(
    Goridge\Relay::create($address)
);
```

{% endcode %}

{% hint style="warning" %}
The `Environment::getRPCAddress()` method returns the RPC address from the `RR_RPC` environment variable and can be
used only inside a PHP worker.
{% endhint %}

## Calling RPC Methods

Once you have created an `$rpc` instance, you can use it to call embedded RPC services.

{% code title="worker.php" %}

```php
$result = $rpc->call('informer.AddWorker', 'http');

var_dump($result);
```

{% endcode %}

{% hint style="warning" %}
In the case of running workers in debug mode `http: { pool.debug: true }` the number of http workers will be zero
(i.e. an empty array `[]` will be returned).

This behavior may be changed in the future, you should not rely on this result to check that the
RoadRunner was launched in development mode.
{% endhint %}

## Available RPC Methods

RoadRunner provides several built-in RPC methods that you can use in your PHP applications:

- `rpc.Version`: Returns the RoadRunner version.
- `rpc.Config`: Returns the RoadRunner configuration.

There are also several plugins that provide RPC methods, but not described in the documentation. You may be able to find
the RPC Go definitions for these plugins in the following repositories:

- [Jobs](https://github.com/roadrunner-server/jobs/blob/master/rpc.go) - Provides a way to create and manage job
  pipelines and push jobs to the queue.
- [KV](https://github.com/roadrunner-server/kv/blob/master/rpc.go) - Provides a way to store and retrieve key-value
  pairs.
- [Informer](https://github.com/roadrunner-server/informer/blob/master/rpc.go)
- [Resetter](https://github.com/roadrunner-server/resetter/blob/master/rpc.go) - Provides a way to reset workers
  globally or separately for each plugin.
- [Status](https://github.com/roadrunner-server/status/blob/master/rpc.go)
- [Metrics](https://github.com/roadrunner-server/metrics/blob/master/rpc.go)
- [Lock](../plugins/locks.md) - Provides a way to obtain and release locks on
  resources. [GitHub](https://github.com/roadrunner-server/lock/blob/master/rpc.go)
- [Service](../plugins/service.md) - Provides a simple API to monitor and control
  processes [GitHub](https://github.com/roadrunner-server/service/blob/master/rpc.go)
- [RPC](https://github.com/roadrunner-server/rpc/blob/master/rpc.go)

### Async PHP RPC interface

You can use `Spiral\Goridge\RPC\AsyncRPCInterface` and an implementation with multiple relays to offer non-blocking I/O for RoadRunner communication.

The interface provides these methods:

- `callIgnoreResponse(string $method, mixed $payload): void` - Invoke the method without waiting for its response.
- `callAsync(string $method, mixed $payload): int` - Invoke the method and return an identifier for its response.
- `hasResponse(int $seq): bool, getResponse(int $seq, mixed $options = null): mixed`
- `hasResponses(array $seqs): array`
- `getResponses(array $seqs, mixed $options = null): iterable` - Retrieve responses for the supplied identifiers.

The `callIgnoreResponse` method can be used to invoke RPC methods without waiting for a response. If you don't need a response, this can greatly improve performance. For example, consider sending metric data.

{% code %}

```php
use Spiral\Goridge\RPC\AsyncRPCInterface;
use Spiral\Goridge\RPC\Exception\ServiceException;
use Spiral\RoadRunner\Metrics\Exception\MetricsException;
use Spiral\RoadRunner\Metrics\MetricsInterface;

final class MetricsIgnoreResponse implements MetricsInterface
{
    public function __construct(
        private readonly AsyncRPCInterface $rpc,
    ) {
    }

    public function add(string $name, float $value, array $labels = []): void
    {
      $this->rpc->callIgnoreResponse('metrics.Add', compact('name', 'value', 'labels'));
    }

    public function sub(string $name, float $value, array $labels = []): void
    {

        $this->rpc->callIgnoreResponse('metrics.Sub', compact('name', 'value', 'labels'));
    }

    public function observe(string $name, float $value, array $labels = []): void
    {

        $this->rpc->callIgnoreResponse('metrics.Observe', compact('name', 'value', 'labels'));
    }

    public function set(string $name, float $value, array $labels = []): void
    {

        $this->rpc->callIgnoreResponse('metrics.Set', compact('name', 'value', 'labels'));
    }

    public function declare(string $name, CollectorInterface $collector): void
    {
        $this->rpc->call('metrics.Declare', [
            'name' => $name,
            'collector' => $collector->toArray(),
        ]);
    }

    public function unregister(string $name): void
    {
        $this->rpc->call('metrics.Unregister', $name);
    }
}
```

{% endcode %}

This code is greatly simplified and does not include error handling, for example. However, it demonstrates how to use the new functionality.

The `callAsync` method allows you to invoke an RPC method and obtain a request identifier, but it does not immediately request a response and does not block further execution while waiting for a response. Instead, you can save the identifier and continue executing other operations that do not require an immediate response from the service. Afterward, you can request the response using the saved identifier.

Using this method, you can implement, for example, sending data to a cache and reading the response only after sending all the necessary data. The code in the example below will be greatly simplified, and the implementation of some methods will be omitted:

{% code %}

```php
use RoadRunner\KV\DTO\V1\Request;
use RoadRunner\KV\DTO\V1\Response;
use Spiral\Goridge\RPC\AsyncRPCInterface;
use Spiral\Goridge\RPC\Exception\ServiceException;

final class AsyncCache
{
    private array $responses = [];

    public function __construct(
        private readonly AsyncRPCInterface $rpc,
    ) {
    }

    public function deleteAsync(string $key): bool
    {
        return $this->deleteMultipleAsync([$key]);
    }

    public function deleteMultipleAsync(iterable $keys): bool
    {
        $this->responses[] = $this->rpc->callAsync('kv.Delete', $this->requestKeys($keys));

        return true;
    }

    public function setAsync(string $key, mixed $value, null|int|\DateInterval $ttl = null): bool
    {
        return $this->setMultipleAsync([$key => $value], $ttl);
    }

    public function setMultipleAsync(iterable $values, null|int|\DateInterval $ttl = null): bool
    {
        $this->responses[] = $this->rpc->callAsync(
            'kv.Set',
            $this->requestValues($values, $this->ttlToRfc3339String($ttl))
        );

        return true;
    }

    public function commitAsync(): bool
    {
        try {
            foreach ($this->rpc->getResponses($this->responses, Response::class) as $response) {
                // Read each response to detect RPC errors.
            }
        } catch (ServiceException) {
            return false;
        } finally {
            $this->responses = [];
        }

        return true;
    }

    private function requestKeys(iterable $keys): Request
    {
        // ...
    }

    protected function requestValues(iterable $values, string $ttl): Request
    {
        // ...
    }

    protected function ttlToRfc3339String(null|int|\DateInterval $ttl): string
    {
        // ...
    }
}

```

{% endcode %}

`getResponses()` returns a lazy iterator. `commitAsync()` returns `true` only after it reads all responses. It returns `false` on the first server-side RPC error; other exceptions propagate. It does not undo completed operations or read the remaining responses after an error.

Usage:

{% code %}

```php
$cache = new AsyncCache($rpc);

$cache->setAsync('key', ['foo' => 'bar']);
$cache->setAsync('second', ['key' => 'value']);
$cache->setAsync('third', 'data');

$result = $cache->commitAsync();
```

{% endcode %}

## What's Next?

1. [Writing a custom plugin](../customization/plugin.md) - Learn how to create your own services and RPC methods.
