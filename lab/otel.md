# OpenTelemetry

The RoadRunner OpenTelemetry (OTEL) plugin exports tracing data to an OTLP receiver, standard output, or standard error.

{% hint style="warning" %}
The v6 plugin rejects `exporter: zipkin`. The native Jaeger exporter is also unavailable. Use `exporter: otlp` and an OTLP receiver instead. A Zipkin `/api/v2/spans` endpoint cannot receive OTLP data.
{% endhint %}

![OpenTelemetry](https://user-images.githubusercontent.com/773481/213914208-cd944ca8-f218-4baf-8a54-5a4e42a1ed40.jpg)

{% hint style="info" %}
Read more about OpenTelemetry on the [official site](https://opentelemetry.io/).
{% endhint %}

This page describes trace export. For Prometheus metrics, see [Metrics](metrics.md).

## Configuration

Here is an example configuration file:

{% code title=".rr.yaml" %}

```yaml
version: "3"

otel:
  resource:
    service_name: "rr_test"
    service_version: "1.0.0"
    service_namespace: "RR-Shop"
    service_instance_id: "UUID"
  insecure: true
  compress: false
  exporter: otlp
  client: grpc
  endpoint: 127.0.0.1:4317
```

{% endcode %}

{% hint style="info" %}
You can use environment variables in the configuration with [shell parameter expansion](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html). This example leaves `client` and `endpoint` unset so the OTLP exporter can use its environment configuration.

{% code title=".rr.yaml" %}

```yaml
version: "3"

otel:
  resource:
    service_name: "${OTEL_SERVICE_NAME:-rr_test}"
    service_version: "${OTEL_SERVICE_VERSION:-1.0.0}"
  insecure: "${OTEL_EXPORTER_OTLP_INSECURE:-true}"
  exporter: "${OTEL_TRACES_EXPORTER:-otlp}"
```

{% endcode %}

When `client` is unset, `OTEL_EXPORTER_OTLP_TRACES_PROTOCOL` takes precedence over `OTEL_EXPORTER_OTLP_PROTOCOL`. Supported values are `grpc` and `http/protobuf`. When `endpoint` is unset, the SDK reads its OTLP endpoint environment variables. For example, use `OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf` with `OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318`.

{% endhint %}

The `grpc` and `jobs` plugins use the configured trace provider. Temporal collects the `otel` worker interceptor. The `http` plugin requires the `otel` middleware in its middleware list.

**Here is an example:**

{% code title=".rr.yaml" %}

```yaml .rr.yaml
http:
  address: 127.0.0.1:15389
  middleware: [ otel, gzip ]
```

{% endcode %}

Requests pass through the middleware list from left to right. Put `otel` before the middleware you want to trace. In this example, the HTTP server span starts before `gzip` runs.

The `headers`, `gzip`, `zstd`, `proxy_ip_parser`, `rate_limiter`, and `http_metrics` spans end before the next handler starts. They measure work before that call. Compression spans do not measure the full response compression time.

The `static` span includes file serving when it handles the request. When it forwards the request, its span ends before the next handler starts. The HTTP plugin's `http` span has kind `Internal` and includes worker and response processing. The outer HTTP `Server` span covers the handlers that follow `otel`.

The `sendfile:post` and `http_metrics:post` spans measure work after the downstream handler returns. The Prometheus duration histogram includes downstream request execution. See [HTTP tracing](../http/http.md#tracing) for these span boundaries.

For Jobs, pass the producer's `traceparent` and optional `tracestate` values in the task headers to continue its trace. Without valid trace context, the push RPC starts a new trace. See [Jobs trace context](../queues/overview-queues.md#trace-context) for batch behavior and driver storage requirements.

**The `otel` section of the configuration file contains the following options:**

| Option | Description |
| -------- | ------------- |
| **insecure** | Use an OTLP connection without TLS. The configuration default is `false`. |
| **compress** | Compress exported spans with gzip. The configuration default is `false`. |
| **exporter** | Trace exporter: `otlp`, `stdout`, or `stderr`. The default is `otlp`. |
| **custom_url** | Override the HTTP request path, for example `/v1/traces`. This option does not set the receiver address. |
| **client** | OTLP transport: `http` or `grpc`. If unset, the plugin checks the protocol environment variables and defaults to `http`. |
| **endpoint** | OTLP receiver address as `host:port`, without a scheme or path. If unset, the SDK uses its environment configuration or transport default. |
| **service_name** | Deprecated. Use `resource.service_name`. The default resource value is `RoadRunner`. |
| **service_version** | Deprecated. Use `resource.service_version`. The default resource value is `1.0.0`. |
| **headers** | Headers sent to the OTLP receiver, such as an `api-key` header. |
| **resource** | Service attributes: `service_name`, `service_version`, `service_namespace`, and `service_instance_id`. |

Values in `resource` take precedence over environment resource attributes. For service name and version, the deprecated top-level settings take precedence over the environment when their `resource` values are empty. The plugin also reads `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES`.

{% hint style="warning" %}
Match `client` to the receiver protocol. OTLP normally uses port `4317` for `grpc` and port `4318` for `http`. For example, `client: http` requires an HTTP receiver such as `endpoint: 127.0.0.1:4318`. See the [OTLP exporter configuration](https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/).
{% endhint %}

## Collector

The OpenTelemetry Collector offers a vendor-agnostic implementation of how to receive, process and export telemetry
data. It removes the need to run, operate, and maintain multiple agents/collectors. This works with improved scalability
and supports open-source observability data formats (e.g. Jaeger, Prometheus, Fluent Bit, etc.) sending to one or more
open-source or commercial back-ends. The local Collector agent is the default location to which instrumentation
libraries export their telemetry data.

To start the collector, you can use the official Docker container `otel/opentelemetry-collector-contrib`.

**Here is an example `docker-compose.yaml`:**

{% code title="docker-compose.yaml" %}

```yaml
version: "3.6"

services:
  collector:
    image: otel/opentelemetry-collector-contrib
    command: [ "--config=/etc/otel-collector-config.yml" ]
    volumes:
      - ./otel-collector-config.yml:/etc/otel-collector-config.yml
    ports:
      - "4318:4318"
      - "4317:4317"
```

{% endcode %}

{% hint style="info" %}
Read more about the OpenTelemetry Collector on the [official site](https://opentelemetry.io/docs/collector/).
{% endhint %}

### Collector Configuration

The collector is started with the `otel-collector-config.yml` configuration file, which specifies how the collector
should receive, process, and export the tracing data.

RoadRunner sends OTLP data to the Collector. This Collector configuration prints received traces with the `debug` exporter. Replace it with your backend exporter to store traces.

{% code title="otel-collector-config.yml" %}

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s

exporters:
  debug:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [ otlp ]
      processors: [ batch ]
      exporters: [ debug ]
```

{% endcode %}

| Option | Description |
| --- | --- |
| **receivers** | Protocols that receive trace data. |
| **processors** | Processing steps for trace data. This example uses `batch` to group traces before export. |
| **exporters** | Output destinations. This example uses `debug` to print traces. |
| **service** | Enabled pipelines. This example connects the OTLP receiver, batch processor, and debug exporter. |

{% hint style="info" %}
Read more about the OpenTelemetry Collector configuration on
the [official site](https://opentelemetry.io/docs/collector/configuration/).
{% endhint %}

## PHP Client

The official [PHP SDK](https://github.com/open-telemetry/opentelemetry-php) for OpenTelemetry provides support for the
OpenTelemetry standard on the PHP side.

To configure your PHP application to use the RoadRunner OpenTelemetry plugin, you need to use environment variables.

**Here is an example of the required environment variables:**

{% code title=".env" %}

```sh
# OpenTelemetry
OTEL_SERVICE_NAME=php-blog
OTEL_TRACES_EXPORTER=otlp
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318 # Collector address
OTEL_PHP_TRACES_PROCESSOR=simple
```

{% endcode %}

You can pass these environment variables to your PHP application from the RoadRunner configuration using
the `server.env` option:

{% code title=".rr.yaml" %}

```yaml
server:
  command: "php otel_worker.php"
  env:
    - OTEL_SERVICE_NAME: php
    - OTEL_TRACES_EXPORTER: otlp
    - OTEL_EXPORTER_OTLP_PROTOCOL: http/protobuf
    - OTEL_EXPORTER_OTLP_ENDPOINT: http://127.0.0.1:4318
    - OTEL_PHP_TRACES_PROCESSOR: simple
  relay: pipes
```

{% endcode %}

## Supported plugins

Here is the list of currently supported plugin

| Plugin/Driver | Description                                                                                  |
|---------------|----------------------------------------------------------------------------------------------|
| **Redis**     | Redis driver, e.g.: [link](https://redis.uptrace.dev/guide/go-redis-monitoring.html#uptrace) |
| **Memcached** | Memcached driver. Native OTEL integration.                                                   |
| **In-Memory** | In-Memory KV/Jobs driver.                                                                    |
| **BoltDB**    | Jobs and KV drivers.                                                                         |
| **KV**        | KV PRC layer.                                                                                |
| **HTTP**      | HTTP plugin with all HTTP middleware (gzip, http_tracing, headers, etc).                     |
| **JOBS**      | JOBS plugin RPC layer.                                                                       |
| **AMQP**      | AMQP driver.                                                                                 |
| **SQS**       | SQS driver.                                                                                  |
| **Kafka**     | Kafka driver.                                                                                |
| **NATS**      | NATS driver.                                                                                 |
| **Beanstalk** | Beanstalk driver.                                                                            |
| **Temporal**  | Workflow and activity tracing through the `otel` worker interceptor.                         |

{% hint style="info" %}
Thanks to [Brett McBride](https://github.com/brettmc), he created a rr-otel [PHP demo](https://github.com/brettmc/rr-otel-demo).
{% endhint %}

## Original issue

* [link](https://github.com/roadrunner-server/roadrunner/issues/1027)

## Troubleshooting:

* If you have problems with setting endpoints via OTEL envs, read this comment: [link](https://github.com/roadrunner-server/roadrunner/issues/1848#issuecomment-1938860185)
