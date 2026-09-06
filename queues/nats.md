# NATS Driver

Support for NATS (JetStream only) has been available in RR since `v2.5.0`.

## Configuration

{% code title=".rr.yaml" %}

```yaml .rr.yaml
version: "3"

nats:
  addr: "demo.nats.io"

jobs:
  num_pollers: 10
  pipeline_size: 100000
  pool:
    num_workers: 10
    max_jobs: 0
    allocate_timeout: 60s
    destroy_timeout: 60s

  pipelines:
    test-1:
      driver: nats
      config:
        # Pipeline priority
        # If the job has priority set to 0, it will inherit the pipeline's priority. Default: 10.
        priority: 2

        # NATS prefetch
        # Messages to read into the channel
        prefetch: 100

        # NATS subject
        # Default: default
        subject: default

        # NATS stream
        # Default: default-stream
        stream: foo

        # The consumer will only start receiving messages that were created after the consumer was created
        # Default: false (deliver all messages from the stream beginning)
        # Not applied in v6.0.0-beta.5.
        deliver_new: true

        # Consumer rate-limiter in bytes https://docs.nats.io/jetstream/concepts/consumers#ratelimit
        # Default: 1000
        # Not applied in v6.0.0-beta.5.
        rate_limit: 100

        # Delete the stream after the pipeline is stopped
        # Default: false
        delete_stream_on_stop: false

        # Delete message from the stream after successful acknowledge
        # Default: false
        delete_after_ack: false

        # v6 beta: time before redelivery if no acknowledgment is received.
        # Default: 30s
        ack_wait: 30s
```

{% endcode %}

## Configuration options

**Here is a detailed description of each of the nats-specific options:**

### Subject

`subject` - nats [subject](https://docs.nats.io/nats-concepts/subjects).

### Stream

`stream` - stream name.

{% hint style="info" %}
To prevent duplicate message consumption, ensure that each pipeline is configured with a unique NATS stream. Using the same stream for multiple pipelines will result in the same message being processed multiple times.
{% endhint %}

### Deliver new

`deliver_new` - the consumer will only start receiving messages that were created after the consumer was created.

In NATS `v6.0.0-beta.5`, this setting is parsed but not applied to the consumer. It does not prevent replay after a consumer restart.

### Rate limit

`rate_limit` - NATS rate [limiter](https://docs.nats.io/jetstream/concepts/consumers#ratelimit).

In NATS `v6.0.0-beta.5`, this setting is parsed but not applied to the consumer.

### Delete stream on stop

`delete_stream_on_stop` deletes the whole stream when the pipeline stops. Default: `false`.

In the NATS v6 beta line (`v6.0.0-beta.5`), stopping an active pipeline with this option set to `false` no longer purges the stream. Messages remain subject to the stream's retention policy. Setting this option to `true` still deletes the stream and its messages.

RR creates a new consumer on each run or resume. Retained messages can be delivered again, including messages acknowledged by an earlier consumer when `delete_after_ack` is `false`. Review stream retention before upgrading. Make job handlers safe to process the same job more than once.

### Delete after ack

`delete_after_ack` - delete message after it successfully acknowledged.

### Ack wait

In the NATS v6 beta line (`v6.0.0-beta.5`), `ack_wait` sets the time before JetStream redelivers an unacknowledged message. NATS v5 ignored this setting.

YAML uses a Go duration string, such as `ack_wait: 30s` or `ack_wait: 2m`. The default is `30s`. Use a duration longer than the expected queueing and processing time.

The `jobs.Declare` RPC payload instead uses integer seconds encoded as a string: `"ack_wait": "30"`. Do not send a duration string such as `"30s"` through RPC.
