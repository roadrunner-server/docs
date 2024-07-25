# Google Pub/Sub driver [Beta]

{% hint style="warning" %}
Driver is still in beta. Please report any issues you encounter. Configuration might be updated due to the received feedback.
{% endhint %}

Pub/Sub is an asynchronous and scalable messaging service that decouples services producing messages from services processing those messages.

Pub/Sub allows services to communicate asynchronously, with latencies on the order of 100 milliseconds.

Pub/Sub is used for streaming analytics and data integration pipelines to load and distribute data. It's equally effective as a messaging-oriented middleware for service integration or as a queue to parallelize tasks.

Pub/Sub lets you create systems of event producers and consumers, called publishers and subscribers. Publishers communicate with subscribers asynchronously by broadcasting events, rather than by synchronous remote procedure calls (RPCs).

Publishers send events to the Pub/Sub service, without regard to how or when these events are to be processed. Pub/Sub then delivers events to all the services that react to them. In systems communicating through RPCs, publishers must wait for subscribers to receive the data. However, the asynchronous integration in Pub/Sub increases the flexibility and robustness of the overall system.

{% hint style="info" %}
Read more about Google Pub/Sub [here](https://cloud.google.com/pubsub/docs/overview).
{% endhint %}

## Configuration

{% code title=".rr.yaml" %}

```yaml
version: "3"

google_pub_sub:
  insecure: true
  host: 127.0.0.1:8085

jobs:
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
        max_delivery_attempts: 3
```

{% endcode %}

## Configuration options

**Here is a detailed description of each of the amqp-specific options:**

### Priority

`priority`- job priority, integer. A lower value corresponds to a higher priority. For instance, consider two pipelines: `pipe1`
with a priority of `1` and `pipe10` with a priority of `10`. Workers will only take jobs from `pipe10` if all the jobs
from `pipe1` have been processed.

### Project ID

`project_id` - required, string. Google Cloud project ID. You can use a special value `*detect-project-id*` (with asterics) to detect the project ID from the credentials. More info: [link](https://developers.google.com/accounts/docs/application-default-credentials)

### Topic

`topic` - required, string. Topic is a named resource that represents a feed of messages. The specified topic ID must start with a letter, and contain only letters `([A-Za-z])`, numbers `([0-9])`, dashes `(-)`, underscores `(_)`, periods `(.)`, tildes `(~)`, plus `(+)` or percent signs `(%)`. It must be between 3 and 255 characters in length, and must not start with "goog". For more information, see: [link](https://cloud.google.com/pubsub/docs/admin#resource_names)

https://cloud.google.com/pubsub/docs/handling-failures#dead_letter_topic

### Dead letter topic

`dead_letter_topic` - optional, string. Should be used with `max_delivery_attempts`. If the Pub/Sub service attempts to deliver a message but the subscriber can't acknowledge it, Pub/Sub can forward the undeliverable message to a dead-letter topic. For more information, see: [link](https://cloud.google.com/pubsub/docs/handling-failures#dead_letter_topic)

### Max delivery attempts

`max_delivery_attempts` - optional, int. Should be used with `dead_letter_topic`. The maximum number of delivery attempts for a message. For more information, see: [link](https://cloud.google.com/pubsub/docs/handling-failures#dead_letter_topic)

## Limitations

1. TLS is not supported at the moment.
2. Metrics are not supported at the moment.
3. Telemetry and Authentification are not supported at the moment. Use `insecure: true` to test this driver.
