# NATS Driver

Support for NATS (JetStream only) has been available in RR since `v2.5.0`.

## Configuration

{% code title=".rr.yaml" %}

```yaml .rr.yaml
version: "3"

nats:
  addr: "demo.nats.io"

jobs:
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
        # RR does not apply this option to the consumer.
        deliver_new: true

        # Consumer rate-limiter in bytes https://docs.nats.io/jetstream/concepts/consumers#ratelimit
        # Default: 1000
        # RR does not apply this option to the consumer.
        rate_limit: 100

        # Delete the stream after the pipeline is stopped
        # Default: false
        delete_stream_on_stop: false

        # Delete message from the stream after successful acknowledge
        # Default: false
        delete_after_ack: false

        # Time before redelivery if no acknowledgment is received.
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

RR parses `deliver_new` but does not apply it to the consumer. It does not prevent replay after a consumer restart.

### Rate limit

RR parses `rate_limit` but does not apply it to the consumer.

### Delete stream on stop

`delete_stream_on_stop` deletes the whole stream when the pipeline stops. Default: `false`.

Stopping a pipeline with `delete_stream_on_stop: false` preserves the stream. Messages remain subject to the stream's retention policy. Setting this option to `true` deletes the stream and its messages.

RR creates a new consumer on each run or resume. Retained messages can be delivered again, including messages acknowledged by an earlier consumer when `delete_after_ack` is `false`. Review stream retention before upgrading. Make job handlers safe to process the same job more than once.

### Delete after ack

`delete_after_ack` - delete message after it successfully acknowledged.

### Ack wait

`ack_wait` sets the time before JetStream redelivers an unacknowledged message.

YAML uses a Go duration string, such as `ack_wait: 30s` or `ack_wait: 2m`. The default is `30s`. Use a duration longer than the expected queueing and processing time.

The `jobs.Declare` RPC payload instead uses integer seconds encoded as a string: `"ack_wait": "30"`. Do not send a duration string such as `"30s"` through RPC.

## Headers

RR preserves job headers when it republishes a job for retry. This includes application headers, trace context, and the `pool` header used by [named worker pools](overview-queues.md#named-worker-pools).
