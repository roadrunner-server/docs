# Google Pub/Sub Driver

RoadRunner publishes jobs to Google Pub/Sub topics and receives them through subscriptions.

{% hint style="info" %}
Read more about Google Pub/Sub [here](https://cloud.google.com/pubsub/docs/overview).
{% endhint %}

## Configuration

The global `google_pub_sub` section is required. This example connects to a local Pub/Sub emulator:

{% code title=".rr.yaml" %}

```yaml
version: "3"

google_pub_sub:
  insecure: true
  endpoint: 127.0.0.1:8085

jobs:
  consume: ["test-1"]
  pool:
    num_workers: 10

  pipelines:
    test-1:
      driver: google_pub_sub
      config:
        priority: 1
        project_id: test
        topic: rrTopic1
        dead_letter_topic: "dead-letter-topic"
        max_delivery_attempts: 10
```

{% endcode %}

### Google Cloud Connection

For Google Cloud, set `insecure: false` and an explicit service endpoint. The client uses TLS and [Application Default Credentials](https://docs.cloud.google.com/docs/authentication/provide-credentials-adc).

{% code title=".rr.yaml" %}

```yaml
google_pub_sub:
  insecure: false
  endpoint: pubsub.googleapis.com:443
```

{% endcode %}

Set each pipeline's `project_id` to your Google Cloud project ID. Without an explicit `endpoint`, RR uses `127.0.0.1:8085`. Use `insecure: true` for an emulator. This setting disables TLS, authentication, and Google client telemetry.

## Configuration options

**Here is a detailed description of each of the Google Pub/Sub-specific options:**

### Priority

`priority`- job priority, integer. A lower value corresponds to a higher priority. For instance, consider two pipelines: `pipe1`
with a priority of `1` and `pipe10` with a priority of `10`. Workers will only take jobs from `pipe10` if all the jobs
from `pipe1` have been processed.

### Project ID

`project_id` - required, string. Google Cloud project ID. You can use a special value `*detect-project-id*` (with asterisks) to detect the project ID from the credentials. More info: [link](https://developers.google.com/accounts/docs/application-default-credentials)

### Topic

`topic` - required, string. Topic is a named resource that represents a feed of messages. The specified topic ID must start with a letter, and contain only letters `([A-Za-z])`, numbers `([0-9])`, dashes `(-)`, underscores `(_)`, periods `(.)`, tildes `(~)`, plus `(+)` or percent signs `(%)`. It must be between 3 and 255 characters in length, and must not start with "goog". For more information, see: [link](https://cloud.google.com/pubsub/docs/admin#resource_names)

### Dead letter topic

`dead_letter_topic`: Optional topic ID. Use it with `max_delivery_attempts` to configure forwarding of messages that cannot be acknowledged. See [Pub/Sub dead-letter topics](https://cloud.google.com/pubsub/docs/handling-failures#dead_letter_topic).

### Max delivery attempts

`max_delivery_attempts`: Optional delivery-attempt setting for dead-letter forwarding. RoadRunner defaults to `10` when `dead_letter_topic` is set. See [Pub/Sub dead-letter configuration](https://cloud.google.com/pubsub/docs/handling-failures#dead_letter_topic).

## Subscriptions

RR reuses existing topics and subscriptions. The pipeline name is the subscription ID. Keep the project, topic, and pipeline names unchanged during an upgrade to use the same subscription.

RR applies `dead_letter_topic` and `max_delivery_attempts` when it creates subscriptions for both YAML pipelines and dynamically declared pipelines. These settings do not update an existing subscription. Use Pub/Sub administration to change an existing subscription's dead-letter policy. Do not delete a subscription with pending messages just to apply new settings.

## Pause and Resume

Pausing a pipeline cancels its receive operation and stops new message pulls. Publishing remains available while paused. Jobs already received by RoadRunner can still complete. Resume starts receiving from the same subscription again.

## Statistics

Use Google Pub/Sub monitoring for queue counts. RR statistics return pipeline identity only. Zero counts and `ready: false` do not report the broker's actual state.
