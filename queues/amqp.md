# AMQP Driver

Strictly speaking, AMQP (and 0.9.1 version used) is a protocol, not a full-fledged driver, so you can use any servers
that support this protocol (on your own, only rabbitmq was tested), such as:

- [RabbitMQ](https://www.rabbitmq.com/),
- [Apache Qpid](http://qpid.apache.org/)
- [Apache ActiveMQ](http://activemq.apache.org/).

However, it is recommended to use RabbitMQ as the main implementation, and reliable performance with other
implementations is not guaranteed.

To install and configure the RabbitMQ, use the
corresponding [documentation page](https://www.rabbitmq.com/download.html).

{% hint style="info" %}
Immediate publishes wait for a publisher confirm from RabbitMQ. Delayed publishes do not wait for that confirmation before returning. A publisher confirm does not prove that a message reached a queue.
{% endhint %}

## Named Connections (Development)

{% hint style="warning" %}
Named connections and nested-only static configuration are development/unreleased changes for the next major release. The pinned [RR source build](../intro/install.md) at `b0cccd9` uses AMQP `v6.0.0-beta.9` and includes neither change. That beta allowed both flat and nested static configuration. All YAML examples below require an AMQP dependency with both changes.
{% endhint %}

Define each connection under `amqp.<name>`. Each entry requires an explicit `addr` with a [connection DSN](https://www.rabbitmq.com/uri-spec.html). Each entry can also have an optional `tls` section. Every YAML AMQP pipeline must set `config.connection` to a configured name.

RR rejects pipeline creation if the connection selector is missing or empty. An unknown connection name also causes an error. RR also rejects a selected connection with a missing or empty `addr`.

There is no implicit default connection or localhost fallback. Top-level `amqp.addr` and `amqp.tls` are not supported. Connection names select configuration. Each pipeline keeps its own broker sockets.

Connection names are separate from [named worker pools](overview-queues.md#named-worker-pools). Connection selection does not add a message header.

TLS options belong under `amqp.<name>.tls`:

- `key`: path to a key file.
- `cert`: path to a certificate file.
- `root_ca`: path to Root CAs used by the AMQP client to trust and verify the broker/server certificate during TLS dial.
- `client_auth_type`: possible values are `no_client_cert`, `request_client_cert`, `require_any_client_cert`, `verify_client_cert_if_given`, `require_and_verify_client_cert`. This setting does not control broker certificate verification.

Configure [RabbitMQ TLS support](https://www.rabbitmq.com/ssl.html) on the broker.

In the AMQP v6 beta line (`v6.0.0-beta.9`), `root_ca` adds trust roots for broker certificate verification. Reconnects reuse the configured TLS settings. Use an `amqps://` address with valid `key` and `cert` files. A `root_ca`-only TLS block is not supported. Omit `tls` when using a plain `amqp://` connection.

{% code title=".rr.yaml (development/unreleased)" %}

```yaml
version: "3"

amqp:
  brokerA:
    addr: amqps://guest:guest@rabbitmq.example.com:5671/

    # AMQPS TLS configuration
    #
    # This section is optional
    tls:
      # Path to the key file
      #
      # This option is required
      key: /etc/rr/tls/client.key

      # Path to the certificate
      #
      # This option is required
      cert: /etc/rr/tls/client.crt

      # Path to Root CAs used by the AMQP client to trust and verify the broker/server certificate during TLS dial.
      #
      # This option is optional
      root_ca: /etc/rr/tls/ca.crt

      # Legacy client_auth_type setting.
      #
      # This option is optional. Default value: no_client_cert. Possible values: no_client_cert, request_client_cert, require_any_client_cert, verify_client_cert_if_given, require_and_verify_client_cert
      client_auth_type: no_client_cert
```

{% endcode %}

Configure each pipeline's exchange and queue as shown below.

## Pipeline Configuration

Static AMQP configuration uses only nested `exchange` and `queue` sections under `jobs.pipelines.<name>.config`. At least one section is required. Missing sections receive default values.

AMQP `config.version` is removed. Keep the root `version: "3"`. Use the [migration table](#migration) to move old flat entity settings.

### Options in `config`

- `connection`: required connection name from `amqp` in the development configuration.
- `priority`: pipeline priority. If a job has priority `0`, it inherits the pipeline priority. Default: `10`.
- `prefetch`: RabbitMQ QoS prefetch. Default: `10`.
- `redial_timeout`: reconnect timeout in seconds. Default: `60`.

Zero or negative values for `priority`, `prefetch`, and `redial_timeout` use their defaults.

### Exchange settings

- `name`: exchange name. Default: `amqp.default`.
- `type`: exchange type. Supported: `direct`, `fanout`, `topic`, `headers`. Default: `direct`.
- `durable`: durable exchange flag. Default: `false`.
- `auto_delete`: auto-delete exchange when last queue is unbound. Default: `false`.
- `declare`: declare exchange during pipeline creation. Default: `true`.

### Queue settings

- `name`: queue name. Optional for producer-only pipelines; required for `run`, `resume`, and `pause`.
- `routing_key`: routing key. Required when `exchange.type != fanout`.
- `durable`: durable queue flag. Default: `false`.
- `auto_delete`: auto-delete queue after the last consumer unsubscribes. Default: `false`.
- `exclusive`: exclusive queue flag. Default: `false`.
- `consumer_id`: consumer identifier. Default: `roadrunner-<uuid>`.
- `delete_on_stop`: delete queue when pipeline stops. Default: `false`.
- `multiple_ack`: ACK this and prior unacked deliveries on the same channel. Default: `false`.
- `requeue_on_fail`: use RabbitMQ requeue on failure (Nack). Default: `false`.
- `headers`: queue declaration arguments (for example, `x-queue-mode: lazy`).
- `declare`: declare and bind queue when `Run` or `Resume` starts consumption. Default: `true`.

For both entities, an omitted `declare` has the same effect as `true`. Explicit `false` disables declaration. Pipeline creation does not declare or bind the queue.

{% hint style="info" %}
See also [AMQP model](https://www.rabbitmq.com/tutorials/amqp-concepts.html#amqp-model) documentation section.
{% endhint %}

{% hint style="info" %}
Producer-only pipeline: `queue.name` can be empty, `push` works, but `run`, `resume`, and `pause` will fail without a queue name.
{% endhint %}

{% hint style="info" %}
If `exchange.type` is not `fanout`, `queue.routing_key` must be set when RR initializes or declares the pipeline. This also applies to consume-only pipelines and pipelines with declarations disabled.
{% endhint %}

{% hint style="info" %}
Read more about Nack in RabbitMQ official docs: https://www.rabbitmq.com/confirms.html#consumer-nacks-requeue
{% endhint %}

This development example consumes from `brokerA` through `consume-a`. It publishes to `brokerB` through `publish-b`.

{% code title=".rr.yaml (development/unreleased)" %}

```yaml
version: "3"

rpc:
  listen: tcp://127.0.0.1:6001

server:
  command: php worker.php
  relay: pipes

amqp:
  brokerA:
    addr: amqp://guest:guest@broker-a:5672/
  brokerB:
    addr: amqp://guest:guest@broker-b:5672/

jobs:
  consume: ["consume-a"]
  pipelines:
    consume-a:
      driver: amqp
      config:
        connection: brokerA
        priority: 10
        prefetch: 10
        redial_timeout: 60
        exchange:
          name: amqp.default
          type: direct
          durable: false
          auto_delete: false
          declare: true
        queue:
          name: team-a-queue
          routing_key: team-a
          durable: false
          auto_delete: false
          exclusive: false
          consumer_id: ""
          delete_on_stop: false
          multiple_ack: false
          requeue_on_fail: false
          headers:
            x-queue-mode: lazy
          declare: true
    publish-b:
      driver: amqp
      config:
        connection: brokerB
        exchange:
          name: team-b-exchange
          type: direct
        queue:
          routing_key: team-b
```

{% endcode %}

Only `consume-a` is in `jobs.consume`. The producer-only `publish-b` pipeline has no queue name. Create a destination queue on broker B before publishing. Bind it to `team-b-exchange` with routing key `team-b`.

{% code title="worker.php" %}

```php
<?php

use Spiral\Goridge\RPC\RPC;
use Spiral\RoadRunner\Jobs\Consumer;
use Spiral\RoadRunner\Jobs\Jobs;

require __DIR__ . '/vendor/autoload.php';

$jobs = new Jobs(RPC::create('tcp://127.0.0.1:6001'));
$destination = $jobs->connect('publish-b');
$consumer = new Consumer();

while ($task = $consumer->waitTask()) {
    try {
        $destination->dispatch($destination->create($task->getName(), $task->getPayload()));
        $task->ack();
    } catch (\Throwable $e) {
        $task->nack($e, redelivery: true);
    }
}
```

{% endcode %}

RR does not provide a cross-broker transaction. A failure after publishing to B but before acknowledging on A can cause duplicate messages on B. Make the destination handler safe to repeat.

## Read-only RabbitMQ Permissions

Disable declarations when the RabbitMQ user has no `configure` permission:

- `jobs.pipelines.<name>.config.exchange.declare: false`
- `jobs.pipelines.<name>.config.queue.declare: false`

Create the exchange, queue and binding before starting RR. The broker user still needs `read` permission to consume and `write` permission to publish. Keep `queue.delete_on_stop: false` to avoid a queue deletion request.

With `queue.declare: false`, RR skips both queue binding and passive queue inspection. `jobs.Stat` reports `active: 0` without querying queue depth. This does not mean the queue is empty.

Delayed publishing and delayed requeue still declare and bind temporary queues. These operations still require `configure` permission.

{% code title=".rr.yaml (development/unreleased)" %}

```yaml
version: "3"

amqp:
  brokerA:
    addr: amqp://readonly:readonly@127.0.0.1:5675/TEST

jobs:
  pipelines:
    readonly:
      driver: amqp
      config:
        connection: brokerA
        exchange:
          name: test-1-exchange
          type: fanout
          durable: true
          auto_delete: false
          declare: false
        queue:
          name: test-1-queue
          routing_key: test-1
          durable: true
          auto_delete: false
          exclusive: false
          declare: false
```

{% endcode %}

## Runtime / RPC (`jobs.Declare`)

Dynamic pipeline declaration over RPC remains a flat string map. It is separate from static YAML configuration. It does not use nested `config.exchange` or `config.queue` sections.

The `jobs.Declare` payload accepts these declaration controls as strings:

- `exchange_declare`: `"true"` (default) or `"false"`.
- `queue_declare`: `"true"` (default) or `"false"`.

For named connections (development/unreleased), set the flat `connection` field to a configured name such as `"brokerB"`. Do not send a DSN or TLS settings in the RPC payload.

The existing PHP 4.x [AMQPCreateInfo API](https://github.com/roadrunner-php/jobs/blob/4.x/src/Queue/AMQPCreateInfo.php) accepts `queueHeaders`. `Jobs` serializes this map as JSON in the flat `queue_headers` field. Set the reserved `rr_connection` key in that map to select a connection. No PHP package change is required.

RR reads `rr_connection` only when the flat `connection` field is absent. The flat field takes precedence whenever it is present, even when it is empty. An empty value causes an error. RR removes the reserved key from queue arguments before broker declaration and configuration storage. Other queue arguments stay unchanged. This key is RPC configuration metadata, not a message header. YAML pipelines must still set `config.connection`.

This runtime example creates the `runtime-b` pipeline on `brokerB` and declares its exchange. It does not declare or bind the queue. PHP 4.x `Jobs` requires an RPC instance:

{% code title="create.php (development/unreleased AMQP)" %}

```php
use Spiral\Goridge\RPC\RPC;
use Spiral\RoadRunner\Jobs\Jobs;
use Spiral\RoadRunner\Jobs\Queue\AMQPCreateInfo;

$jobs = new Jobs(RPC::create('tcp://127.0.0.1:6001'));
$queue = $jobs->create(new AMQPCreateInfo(
    name: 'runtime-b',
    queue: 'team-b-queue',
    exchange: 'team-b-exchange',
    routingKey: 'team-b',
    queueHeaders: ['rr_connection' => 'brokerB'],
));
```

{% endcode %}

Call `$jobs->resume('runtime-b')` after creation to declare and bind the queue and start consumption. For a producer-only pipeline, omit this call. Its destination queue and binding must exist before publishing.

## Migration

To use the development/unreleased configuration:

1. Select an AMQP dependency with named connections and nested-only static configuration.
2. Move `amqp.addr` to `amqp.<name>.addr`. Move optional `amqp.tls` to `amqp.<name>.tls`. Set an explicit address for each connection.
3. Set `config.connection` on every YAML AMQP pipeline. There is no implicit default connection or localhost fallback.
4. Remove `config.version` from every YAML AMQP pipeline. Keep the root `version: "3"`.
5. For runtime declarations, send flat `connection` or use the existing PHP `queueHeaders` map with `rr_connection`. The flat field takes precedence even when empty. RR removes the reserved key before broker declaration and configuration storage.

Move old flat entity settings to the nested fields below. All paths are relative to `jobs.pipelines.<name>.config`.

| Old Field | New Field |
| --- | --- |
| `exchange` | `exchange.name` |
| `exchange_type` | `exchange.type` |
| `exchange_durable` | `exchange.durable` |
| `exchange_auto_delete` | `exchange.auto_delete` |
| `queue` | `queue.name` |
| `durable` | `queue.durable` |
| `queue_auto_delete` | `queue.auto_delete` |
| `delete_queue_on_stop` | `queue.delete_on_stop` |
| `queue_headers` | `queue.headers` |
| `routing_key`, `exclusive`, `consumer_id`, `multiple_ack`, `requeue_on_fail` | Same key under `queue` |

Keep `priority`, `prefetch` and `redial_timeout` directly in `config`.

The configuration schema rejects `config.version` and removed flat keys. The normal config provider can ignore unknown keys, but old scalar `exchange` or `queue` values fail to decode. Flat flags do not set nested values. For example, a flat `durable: true` does not set `queue.durable`. Migrate all entity settings before starting RR.

For restricted RabbitMQ permissions, set `exchange.declare: false` and `queue.declare: false`. Review the [declaration limitations](#read-only-rabbitmq-permissions) before using delayed jobs.

## What's Next?

1. [Queues and Jobs overview](overview-queues.md) - Review the full jobs pipeline model before configuring AMQP in production.
2. [Read-only RabbitMQ permissions](https://www.rabbitmq.com/docs/access-control) - See declaration flags for restricted users and review the Runtime / RPC (`jobs.Declare`) section on this page for flat declaration keys.
3. [Pipeline configuration](#pipeline-configuration) - Use nested AMQP entity settings. See [RoadRunner configuration](../intro/config.md) for the general configuration structure.
4. [Exchange settings](#exchange-settings) and [Queue settings](#queue-settings) - Review routing and consumption settings.
5. [Allocate Timeout](../known-issues/allocate-timeout.md) and [CRC validation failed](../known-issues/stdout-crc.md) - Use these troubleshooting references when workers fail to process queue jobs as expected.
