# NSQ Driver

RoadRunner publishes jobs to an NSQ topic and consumes them through an NSQ channel.

{% hint style="info" %}
NSQ is newly bundled in the RoadRunner development binary that uses v6 plugins. It is not included in RoadRunner `v2025.1.15`. This page describes the `nsq` driver at `v6.0.0-beta.1`.
{% endhint %}

## Configuration

Start `nsqd` before RoadRunner. The example uses a [Jobs consumer](./overview-queues.md) in `consumer.php`.

{% code title=".rr.yaml" %}

```yaml
version: "3"

rpc:
  listen: tcp://127.0.0.1:6001

server:
  command: php consumer.php
  relay: pipes

nsq:
  addr: tcp://127.0.0.1:4150
  lookupd_poll_interval: 5s

jobs:
  consume: [tasks]
  pipelines:
    tasks:
      driver: nsq
      config:
        topic: tasks
        channel: workers
        prefetch: 10
        priority: 10
        max_attempts: 5
```

{% endcode %}

The global `nsq` section is required. Put connection settings in this section. Put pipeline settings under `jobs.pipelines.<name>.config`. Global values override matching fields in YAML pipeline configurations.

- `addr`: The nsqd TCP address. Both `host:port` and `tcp://host:port` are accepted. The default is `127.0.0.1:4150`. The producer always uses this address, including when consumer discovery is enabled. Driver initialization fails if the producer cannot connect.
- `lookupd`: An optional list of nsqlookupd HTTP addresses, such as `["127.0.0.1:4161"]`. With this option, consumers discover nsqd servers through nsqlookupd. Without it, consumers connect to `addr`.
- `lookupd_poll_interval`: The duration between discovery polls. It also controls the delay before a direct consumer connection reconnects. The default is `60s`.
- `dial_timeout`: The connection timeout. The default is `1s`. Duration settings use a unit suffix, such as `5s`.

## Pipeline Options

- `topic`: The topic to publish to and consume from. It defaults to the pipeline name.
- `channel`: The consumer channel. It defaults to `default`. Consumers on the same topic and channel share the work. Separate channels each receive a copy of the topic's messages.
- `prefetch`: The maximum number of messages in flight for the consumer. It defaults to `10`. Zero or negative values also select `10`.
- `priority`: The default job priority in RoadRunner. It defaults to `10`. Lower numbers have higher priority. This affects RR's priority queue, not NSQ delivery order.
- `max_attempts`: The delivery-attempt limit. See [Retry Limit](#retry-limit) before changing it.

## Delivery Behavior

- Acknowledgement completes the NSQ message. Auto-ack completes it before PHP processes it. Do not enable auto-ack when failed jobs must be retried.
- Job delays and explicit retry delays use seconds. A negative acknowledgement can requeue the original message. Disabling requeue completes the message without another attempt.
- A retry that changes headers publishes a new message to the topic before acknowledging the original. The two operations are not atomic. Workers must tolerate repeated deliveries.
- Pause stops requesting new messages without closing the consumer connection. Jobs already received can still run. Publishing remains available while paused. Resume restores the configured prefetch limit.
- The driver can consume raw messages from other producers. It passes the raw body to PHP with the job name `deduced_by_rr`.

## Retry Limit

{% hint style="warning" %}
In `v6.0.0-beta.1`, omitting `max_attempts` or setting it to `0` keeps the client default of five attempts. Zero does not enable unlimited retries. On a delivery beyond the limit, the client acknowledges the message without sending it to PHP. The driver does not forward it to a dead-letter queue.
{% endhint %}

Use a positive `max_attempts` value for a different limit. A retry that publishes a new message starts a new broker attempt count. This setting is not a total retry limit across those new messages. Applications that must retain failed jobs need their own failure storage.

## Limitations

RR statistics report pipeline identity, listener readiness, and a locally tracked delayed-job count. They do not report the broker backlog. Use [nsqadmin](https://nsq.io/components/nsqadmin.html) for topic and channel counts.

The beta driver does not expose TLS or authentication settings.
