# Migration from v1.0 to v2.0/v2023+

To migrate integration from RoadRunner v1.* to v2.*/v2023+, follow these steps.

## Update Configuration

The second version of RoadRunner uses a single worker factory for all of its plugins. This means that you must include a new
`server` section in your config, which is responsible for worker creation. The limit service is no longer presented as a separate
entity but rather as part of specific service configuration.

{% code title=".rr.yaml" %}

```yaml
rpc:
  listen: tcp://127.0.0.1:6001

server:
  command: "php tests/psr-worker-bench.php"

http:
  address: "0.0.0.0:8080"
  pool:
    num_workers: 4
```

{% endcode %}

{% hint style="warning" %}
Read more in the [config reference](/intro/config.md).
{% endhint %}

## No longer worry about echoing

RoadRunner 2.0+ intercepts all output to STDOUT. This means you can start using default var_dump and other echo
functions without breaking the communication. Yay!

## Explicitly declare PSR-15 dependency

We no longer ship a default PSR implementation with RoadRunner; make sure to include one you like the most
yourself.
For example:

```bash
$ composer require nyholm/psr7
```

## Update Worker Code

RoadRunner simplifies worker creation: use the static `create()` method to automatically configure your worker:

{% code title="worker.php" %}

```php
<?php

use Spiral\RoadRunner;

include "vendor/autoload.php";

$worker = RoadRunner\Worker::create();
```

{% endcode %}

Pass the PSR-15 factories to your PSR worker:

{% code title="worker.php" %}

```php
<?php

use Spiral\RoadRunner;
use Nyholm\Psr7;

include "vendor/autoload.php";

$worker = RoadRunner\Worker::create();
$psrFactory = new Psr7\Factory\Psr17Factory();

$worker = new RoadRunner\Http\PSR7Worker($worker, $psrFactory, $psrFactory, $psrFactory);
```

{% endcode %}

RoadRunner 2 unifies all workers to use similar naming: change `acceptRequest` to `waitRequest`:

{% code title="worker.php" %}

```php
<?php

while ($req = $worker->waitRequest()) {
    try {
        $rsp = new Psr7\Response();
        $rsp->getBody()->write('Hello world!');

        $worker->respond($rsp);
    } catch (\Throwable $e) {
        $worker->getWorker()->error((string)$e);
    }
}
```

{% endcode %}

## Update RPCs

To create an RPC client, use the new Goridge API:

{% code title="worker.php" %}

```php
<?php

$rpc = \Spiral\Goridge\RPC\RPC::create('tcp://127.0.0.1:6001');
```

{% endcode %}
