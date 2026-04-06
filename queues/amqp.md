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
Every message pushed to the RabbitMQ server uses publisher confirms. A message is considered sent only after the server confirms it. This is a reliable way to ensure delivery to the server.
{% endhint %}


After that, you should configure the connection to the server in the `amqp` section. This configuration section
contains exactly one `addr` key with a [connection DSN](https://www.rabbitmq.com/uri-spec.html). The `TLS` configuration sits in the `amqp.tls` section and consists of the following options:

- `key`: path to a key file.
- `cert`: path to a certificate file.
- `root_ca`: path to Root CAs used by the AMQP client to trust and verify the broker/server certificate during TLS dial.
- `client_auth_type`: also known as `mTLS`. Possible values are: `no_client_cert`, `request_client_cert`, `require_any_client_cert`, `verify_client_cert_if_given`, `require_and_verify_client_cert`.

You should also configure `rabbitMQ` with `TLS` support: [link](https://www.rabbitmq.com/ssl.html).

{% code title=".rr.yaml" %}

```yaml
amqp:
  addr: amqp://guest:guest@127.0.0.1:5672

  # AMQPS TLS configuration
  #
  # This section is optional
  tls:
    # Path to the key file
    #
    # This option is required
    key: ""

    # Path to the certificate
    #
    # This option is required
    cert: ""

    # Path to Root CAs used by the AMQP client to trust and verify the broker/server certificate during TLS dial.
    #
    # This option is optional
    root_ca: ""

    # Client auth type (mTLS, peer verification).
    #
    # This option is optional. Default value: no_client_cert. Possible values: no_client_cert, request_client_cert, require_any_client_cert, verify_client_cert_if_given, require_and_verify_client_cert
    client_auth_type: no_client_cert
```

{% endcode %}

Upon establishing a connection to the server, you can create a new queue that utilizes this connection and encompasses
the queue settings, including those specific to AMQP.

## Pipeline Configuration Formats

AMQP pipeline configuration is selected by `jobs.pipelines.<name>.config.version`:

- `0`, `1`, or omitted: legacy flat parser.
- `2`: nested parser with `exchange` and `queue` sections (recommended).

## Version 2 Configuration (Recommended)

Set `jobs.pipelines.<name>.config.version: 2` and define `exchange` and `queue` sections.

### Options in `config`

- `priority`: pipeline priority. If a job has priority `0`, it inherits the pipeline priority. Default: `10`.
- `prefetch`: RabbitMQ QoS prefetch. Default: `10`.
- `redial_timeout`: reconnect timeout in seconds. Default: `60`.

### Exchange settings

- `name`: exchange name. Default: `amqp.default`.
- `type`: exchange type. Supported: `direct`, `fanout`, `topic`, `headers`. Default: `direct`.
- `durable`: durable exchange flag. Default: `false`.
- `auto_delete`: auto-delete exchange when last queue is unbound. Default: `false`.
- `declare`: declare exchange on startup. Default: `true`.

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
- `declare`: declare and bind queue on startup. Default: `true`.

{% hint style="info" %}
See also [AMQP model](https://www.rabbitmq.com/tutorials/amqp-concepts.html#amqp-model) documentation section.
{% endhint %}

{% hint style="info" %}
Producer-only pipeline: `queue.name` can be empty, `push` works, but `run`, `resume`, and `pause` will fail without a queue name.
{% endhint %}

{% hint style="info" %}
If `exchange.type` is not `fanout`, `queue.routing_key` must be set.
{% endhint %}

{% hint style="info" %}
Read more about Nack in RabbitMQ official docs: https://www.rabbitmq.com/confirms.html#consumer-nacks-requeue
{% endhint %}

{% code title=".rr.yaml" %}

```yaml
version: "3"

amqp:
  addr: amqp://guest:guest@127.0.0.1:5672/
  tls:
    key: ""
    cert: ""
    root_ca: ""
    client_auth_type: no_client_cert

jobs:
  pipelines:
    p1:
      driver: amqp
      config:
        version: 2
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
          name: p1-queue
          routing_key: p1
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
```

{% endcode %}

## Read-only RabbitMQ Permissions

If the RabbitMQ user does not have `configure` permission, disable declarations in pipeline configuration:

- `jobs.pipelines.<name>.config.exchange.declare: false`
- `jobs.pipelines.<name>.config.queue.declare: false`

This avoids startup declaration failures and passive queue-check failures in restricted environments.

{% code title=".rr.yaml" %}

```yaml
version: "3"

amqp:
  addr: amqp://readonly:readonly@127.0.0.1:5675/TEST

jobs:
  pipelines:
    readonly:
      driver: amqp
      config:
        version: 2
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

Dynamic pipeline declaration over RPC remains flat. It does not use nested `config.exchange` / `config.queue` sections.

Declaration control keys in `jobs.Declare` payload:

- `exchange_declare`
- `queue_declare`

## Legacy Flat Configuration (`version: 0|1`)

Legacy flat keys remain supported under `jobs.pipelines.<name>.config` when `config.version` is `0`, `1`, or omitted:

- `priority`
- `prefetch`
- `queue`
- `exchange`
- `exchange_type`
- `routing_key`
- `exclusive`
- `multiple_ack`
- `requeue_on_fail`
- `queue_headers`
- `durable`
- `delete_queue_on_stop`
- `redial_timeout`
- `exchange_durable`
- `exchange_auto_delete`
- `queue_auto_delete`
- `consumer_id`

{% code title=".rr.yaml" %}

```yaml
version: "3"

amqp:
  addr: amqp://guest:guest@127.0.0.1:5672/

jobs:
  pipelines:
    p1:
      driver: amqp
      config:
        version: 1
        priority: 10
        prefetch: 10
        queue: p1-queue
        exchange: amqp.default
        exchange_type: direct
        routing_key: p1
        exclusive: false
        multiple_ack: false
        requeue_on_fail: false
        queue_headers: {} # optional, e.g. { x-queue-mode: lazy }
        durable: false
        delete_queue_on_stop: false
        redial_timeout: 60
        exchange_durable: false
        exchange_auto_delete: false
        queue_auto_delete: false
        consumer_id: ""
```

{% endcode %}

{% hint style="info" %}
In legacy format, `queue` can be omitted for push-only pipelines, but `run`, `resume`, and `pause` require a queue name. Also, `routing_key` is required when `exchange_type != fanout`.
{% endhint %}

## Migration

- Prefer `config.version: 2` with nested `exchange` and `queue` sections for new configurations.
- Existing flat configurations remain compatible through `config.version: 0`, `config.version: 1`, or omitted `config.version`.
- For restricted RabbitMQ permissions, set `exchange.declare: false` and `queue.declare: false`.

## What's Next?

1. [Queues and Jobs overview](overview-queues.md) - Review the full jobs pipeline model before configuring AMQP in production.
2. [Read-only RabbitMQ permissions](https://www.rabbitmq.com/docs/access-control) - See declaration flags for restricted users and review the Runtime / RPC (`jobs.Declare`) section on this page for flat declaration keys.
3. [Pipeline configuration formats](../intro/config.md) - Confirm when to use `config.version: 2` versus legacy `0|1`, and review the general RoadRunner configuration structure.
4. [Exchange settings](kafka.md) and [Queue settings](sqs.md) - Compare exchange and queue responsibilities when tuning routing and consumption behavior.
5. [Allocate Timeout](../known-issues/allocate-timeout.md) and [CRC validation failed](../known-issues/stdout-crc.md) - Use these troubleshooting references when workers fail to process queue jobs as expected.
