# Health and Readiness checks

The RoadRunner Status Plugin provides a healthcheck status for various plugins such
as `http`, `grpc`, `temporal`, `jobs` and `centrifuge`. This plugin provides an easy way to check the condition of the
workers and ensure that they are ready to serve requests.

## Activation of the Status Plugin

To activate the health/readiness checks endpoint, include a `status` section in your configuration file.

Here is an example:

{% code title=".rr.yaml" %}

```yaml
version: "3"

status:
  address: 127.0.0.1:2114
```

{% endcode %}

The above configuration sets the address to `127.0.0.1:2114`. This is the address that the plugin will listen to. You
can change the address to any IP address and port number of your choice.

To access the health check, use the following URL: `http://127.0.0.1:2114/health`. This URL will return the health status of all plugins that are enabled and support health probes. To specify a particular plugin, you need to use the `plugin` query parameter: `http://127.0.0.1:2114/health?plugin=http`. In that case, the health status of the `http` plugin will be returned.

{% hint style="info" %}
Repeat the `plugin` query parameter to check multiple plugins: `http://127.0.0.1:2114/health?plugin=http&plugin=grpc`. Names that do not support the check are skipped.
{% endhint %}

The `/health` endpoint calls each selected plugin's health check. For worker-pool checks, an active worker can be busy with a request. Use `/ready` to check for idle workers.

If a checked plugin reports a status of `500` or higher, the HTTP response uses `unavailable_status_code` (`503` by default). The JSON response lists the checked plugins and their reported status or errors:

```json
[
    {
        "plugin_name": "http",
        "error_message": "",
        "status_code": 200
    },
    {
        "plugin_name": "grpc",
        "error_message": "some error message",
        "status_code": 404
    }
]
```

## Readiness Check

To access the readiness check, use the following URL: `http://127.0.0.1:2114/ready`.

For worker-pool checks, `/ready` returns `HTTP 200` when each checked plugin has at least one idle worker. If a checked pool has no ready workers, including when all workers are busy, the endpoint returns `unavailable_status_code` (`503` by default).

Like the health check, you can target a specific plugin using the `plugin` query parameter:

- `http://127.0.0.1:2114/ready?plugin=http`
- `http://127.0.0.1:2114/ready?plugin=grpc`

{% hint style="info" %}
Repeat the `plugin` query parameter to check multiple plugins: `http://127.0.0.1:2114/ready?plugin=http&plugin=grpc`.
{% endhint %}

The response format is the same JSON structure as the `/health` endpoint.

## Customizing the Not-Ready Status Code

By default, the Status Plugin uses a `503` status code. However, you can replace this status code with a custom one.

To achieve this, utilize the `unavailable_status_code` option:

{% code title=".rr.yaml" %}

```yaml
version: "3"

status:
  address: 127.0.0.1:2114
  unavailable_status_code: 501
```

{% endcode %}

## Graceful Shutdown

During graceful shutdown, `/health` returns `200`. The `/ready` and `/jobs` endpoints return `unavailable_status_code` (`503` by default). These responses contain the text `service is shutting down`, not the usual JSON report.

Use `/health` for liveness and `/ready` for readiness. This lets the process finish its current work after readiness checks stop new traffic.

## Check Timeout

Set `check_timeout` to an integer number of seconds. The default is `60`. This value sets the status server's HTTP request and header read timeouts. It does not set a deadline for `Status()`, `Ready()`, or `JobsState()` execution.

{% code title=".rr.yaml" %}

```yaml
version: "3"

status:
  address: 127.0.0.1:2114
  check_timeout: 30
```

{% endcode %}

## Jobs plugin pipelines check

In addition to checking the health status of the workers, you can also examine the pipelines in the Jobs plugin using
the following URL: http://127.0.0.1:2114/jobs

This endpoint returns a JSON array of pipeline states:

```json
[
    {
        "pipeline": "test-1",
        "priority": 13,
        "ready": true,
        "queue": "test-1",
        "active": 0,
        "delayed": 0,
        "reserved": 0,
        "driver": "amqp",
        "error_message": ""
    }
]
```

If the Jobs plugin is absent, `/jobs` returns `unavailable_status_code`. The handler passes the HTTP request context to `JobsState()` so the check can respond to request cancellation.

## Use cases

The health check endpoint serves the following purposes:

### Kubernetes Readiness and Liveness Probes

Configure the liveness probe to use `/health` and the readiness probe to use `/ready`. Busy workers can fail readiness without failing liveness. During shutdown, readiness fails while liveness remains successful.

**Read more [here](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)**

### AWS Target Group Health Checks

If you are using AWS Elastic Load Balancer, you can use it as a health check for your target group. You can configure
the target group to check the health check endpoint and take appropriate action if the endpoint returns an error.

**Read more [here](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html)**

### GCE Load Balancing Health Checks

If you are using Google Cloud Platform, you can use it as a health check for your load balancer. You can configure the
load balancer to check the health check endpoint and take appropriate action if the endpoint returns an error.

**Read more [here](https://cloud.google.com/load-balancing/docs/health-checks)**
