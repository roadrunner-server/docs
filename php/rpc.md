# RPC to RoadRunner

RoadRunner provides a powerful RPC (Remote Procedure Call) interface for communication between PHP applications and the
server using [Goridge library](https://github.com/roadrunner-php/goridge).

## Goridge

Goridge is a high-performance PHP-to-Golang/Golang-to-PHP library developed specifically for communication between PHP
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

## Configuration

You can change the RPC port from the default (`127.0.0.1:6001`) using the following configuration:

{% code title=".rr.yaml" %}

```yaml
version: "3"

rpc:
  listen: tcp://127.0.0.1:6001
```

{% endcode %}

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
used only inside PHP worker.
{% endhint %}

## Calling RPC Methods

Once you have created `$rpc` instance, you can use it to call embedded RPC services.

{% code title="worker.php" %}

```php
$result = $rpc->call('informer.Workers', 'http');

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
  resources. [Github](https://github.com/roadrunner-server/lock/blob/master/rpc.go)
- [Service](../plugins/service.md) - Provides a simple API to monitor and control
  processes [Github](https://github.com/roadrunner-server/service/blob/master/rpc.go)
- [RPC](https://github.com/roadrunner-server/rpc/blob/master/rpc.go)

## What's Next?

1. [Writing a custom plugin](../customization/plugin.md) - Learn how to create your own services and RPC methods.
