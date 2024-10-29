# SQS Driver

[Amazon Simple Queue Service (SQS)](https://aws.amazon.com/sqs/) is an alternative
queue server developed by Amazon, provided as part of the AWS Cloud infrastructure. You can find the documentation
[here](https://docs.aws.amazon.com/sqs/).

While this service is usually deployed on AWS, you can also self-host any compatible version, such as the
community-developed [ElasticMQ](https://github.com/softwaremill/elasticmq).

## Configuration

Specify the following connection settings in the `sqs` configuration settings.

{% code title=".rr.yaml" %}

```yaml
sqs:
  # AWS AccessKey ID.
  # Default: empty
  key: access-key

  # AWS Access Key Secret.
  # Default: empty
  secret: api-secret

  # AWS region.
  # Default: empty
  region: us-west-1

  # AWS SQS endpoint.
  # This parameter *is only required* if your SQS endpoint is *not* hosted on the AWS infrastructure.
  # Always leave this empty if your queue is on AWS SQS.
  # For ElasticMQ, the default is http://127.0.0.1:9324
  # Default: empty
  endpoint: ""
```

{% endcode %}

### Examples

Because the options can be mixed in ways that don't make sense, here are a few examples of how to use the driver
in various scenarios. In all examples, the **queue name** is provided in the `jobs.pipelines.pipeline_name.config.queue`
property below.

#### AWS SQS with dynamic IAM in same region

If you access your AWS SQS queue from inside AWS via a service that supports dynamic IAM roles (such as EC2), and the
queue is in the same region.

{% code title=".rr.yaml" %}

```yaml
sqs:
# No parameters are required. Normally, if your IAM role has the required permissions and your service endpoint
# is in the same region, the environment variables associated with your instance/task will resolve all the 
# configuration for you. You must provide the parent sqs key to enable SQS, even if you want to use the defaults.
```

{% endcode %}

#### AWS SQS in different region with dynamic IAM

If you access your AWS SQS queue from inside AWS via a service that supports dynamic IAM roles (such as EC2), and the
queue is in a different region.

{% code title=".rr.yaml" %}

```yaml
sqs:
  # Only region is required
  region: eu-central-1
```

{% endcode %}

#### AWS SQS without dynamic IAM

If you access your AWS SQS queue from outside AWS, or from a service that does not support IAM, you must provide
an access key and secret. If you access the service from outside AWS, you must also provide `region`.
You may also use this option if your service *does* support dynamic IAM, but you want to use explicit access keys.

{% code title=".rr.yaml" %}

```yaml
sqs:
  key: ASIAIOSFODNN7EXAMPLE
  secret: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  # If in a different region *or* if outside AWS, provide the correct region.
  region: eu-west-1 
```

{% endcode %}

#### Self-hosted (ElasticMQ)

If you self-host your SQS-compatible queue, only the endpoint is required.

{% code title=".rr.yaml" %}

```yaml
sqs:
  endpoint: http://127.0.0.1:9324
```

{% endcode %}

After you have configured the driver, you should configure the queue that will use this connection.

{% code title=".rr.yaml" %}

```yaml
version: "3"

sqs:
# SQS connection configuration. See above example.

jobs:
  pipelines:
    test-sqs-pipeline:
      # Required section.
      # Should be "sqs" for the Amazon SQS driver.
      driver: sqs

      config:
        # Optional section.
        # Default: 10
        prefetch: 10

        # Get queue URL only
        # Default: false
        skip_queue_declaration: false

        # Optional section.
        # Default: 0
        visibility_timeout: 0

        # Optional section.
        # Default: 0
        wait_time_seconds: 0

        # Optional section.
        # Default: default
        queue: default

        # Message group ID: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessage.html#SQS-SendMessage-request-MessageGroupId
        # Default: empty, should be set if FIFO queue is used
        message_group_id: "test"

        # Optional section.
        # Default: empty
        attributes:
          DelaySeconds: 42
          # etc... see https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SetQueueAttributes.html

        # Optional section.
        # Default: empty
        tags:
          test: "tag"
```

{% endcode %}

## FIFO Queues

AWS supports [FIFO](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html)
queues, which guarantee order of delivery. In order for RoadRunner to correctly handle this scenario, some
configuration is required.

1. You should set `retain_failed_jobs` to `true` on the pipeline, so a failed job does not get re-queued at the back of
the queue. This would break the FIFO order.
2. You must have either `num_workers` set to `1` for the worker pool **or** `prefetch` set to `1` for the pipeline. If
you have multiple workers and fetch multiple messages at a time (`prefetch > 1`), RoadRunner may not process the jobs in
correct order.

## Configuration Options

### Prefetch

`prefetch` - The number of jobs to request from SQS at a time. This sets the `MaxNumberOfMessages` parameter on the
[`ReceiveMessage`](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ReceiveMessage.html) call
to the queue. Additionally, it limits the number of queues that may be retained on the priority queue in RoadRunner for
the pipeline. If you set this to zero or omit it, the default value of 1 will be used.

Default: `1`
Maximum: `10`

### Visibility Timeout

`visibility_timeout` - The duration (in seconds) that received messages are hidden from subsequent retrieve requests
after being retrieved by a consumer. This value should be **longer** than the time you expect each job to take, or the
job may run multiple times. If you do not provide a value for this parameter or set it to 0, the `VisibilityTimeout`
attribute of the queue is used, which defaults to 30 seconds. For more information, see the documentation on
[visibility timeouts](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html).

Default: `0`
Maximum: `43200` (12 hours)

### Error Visibility Timeout

`error_visibility_timeout` - If `retain_failed_jobs` is enabled, this is the duration (in seconds) that `NACK`'ed (or
failed) jobs will have their visibility timeout changed to. Note that you cannot have a message remain invisible for
more than the maximum value in cases where it fails multiple times, so setting this value to the maximum is error-prone.
This parameter is ignored if `retain_failed_jobs` is `false`, and it must be larger than zero to apply.

Default: `0`
Maximum: `43200` (12 hours)

### Retain Failed Jobs

`retain_failed_jobs` - If enabled, jobs will **not** be deleted and re-queued if they fail. Instead, RoadRunner will
simply let them be processed again after `visibility_timeout` has passed. If you set `error_visibility_timeout` and
enable this feature, RoadRunner will change the timeout of the job to the value of `error_visibility_timeout`. This lets
you customize the timeout for errors specifically. If you enable this feature, you can configure SQS to automatically
move jobs that fail multiple times to a
[dead-letter queue](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html).
Always enable this feature if you consume from a FIFO queue.

Default: `false`

### Wait Time

`wait_time_seconds` - The duration (in seconds) for which the call waits for a message to arrive in the queue before
returning. If a message is available, the call returns sooner than `wait_time_seconds`. If no messages are available and
the wait time expires, the call returns successfully with an empty list of messages. Please note that this parameter
cannot be explicitly configured to use zero, as zero will apply the queue defaults. In most cases, you should set this
to a non-zero value.

Default: `0`
Maximum: `20`

{% hint style="warning" %}
### Short vs. Long Polling

By default, SQS and RoadRunner is configured to use **short polling**. Please review the documentation on the
differences between
[short and long polling](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html)
and make sure that your queue and RoadRunner is configured correctly. Long polling will *usually* be a cost-saving
feature with no practical impact on performance or functionality. Remember that not providing a value (or zero) for
`wait_time_seconds` will cause your queue wait time to be based on the `ReceiveMessageWaitTimeSeconds` attribute
configured on the queue, which also defaults to zero.
{% endhint %}

### Queue

`queue` - The name of the queue. May only contain alphanumeric characters, hyphens (`-`), and underscores (`_`). On AWS,
this will form the last part of the queue URL, i.e. `https://sqs.<region>.amazonaws.com/<account id>/<queue>`.

Default: `default`

{% hint style="info" %}
Please note that [FIFO](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html)
queue names **must** end with `.fifo`.
{% endhint %}

### Message Group ID

`message_group_id` - Message group ID is required for FIFO queues. Messages that belong to the same message group are
processed in a FIFO manner.
More info:
[link](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessage.html#SQS-SendMessage-request-MessageGroupId)

### Skip Queue Declaration

`skip_queue_declaration` - By default, RR tries to create the queue (using the `queue` name) if it does not exist. Set
this option to `true` if the queue already exists.

Default: `false`

{% hint style="info" %}
For queue creation to work, the credentials or the IAM role used must have the
[required permissions](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-api-permissions-reference.html),
such as `sqs:CreateQueue` and `sqs:SetQueueAttributes`.
{% endhint %}

### Attributes

`attributes` - A list of
[AWS SQS attributes](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SetQueueAttributes.html)
to configure for the queue. 

{% code title=".rr.yaml" %}

```yaml
attributes:
  DelaySeconds: 0
  MaximumMessageSize: 262144
  MessageRetentionPeriod: 345600
  ReceiveMessageWaitTimeSeconds: 0
  VisibilityTimeout: 30
```

{% endcode %}

{% hint style="info" %}
Attributes are only set if RoadRunner creates the queue. Attributes of existing queues are **not modified**.
{% endhint %}

### Tags

`tags` - Tags don't have any semantic meaning. Amazon SQS interprets tags as character. Tags are only set if RoadRunner
creates the queue. Existing queues are **not** modified.

{% hint style="info" %}
This functionality is rarely used and slows down queues. Please see
[this link](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-queue-tags.html).
{% endhint %}
