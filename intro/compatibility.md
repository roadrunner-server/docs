# Upgrading

This section provides information about upgrading your RoadRunner configuration to the latest version.

## Compatibility matrix

The compatibility matrix provides information about the supported configuration versions for different RoadRunner
versions.

| RR version     | Configuration version                                                          |
|----------------|--------------------------------------------------------------------------------|
| **>=2023.x.x** | **3**                                                                          |
| **>=2.8**      | **2.7**                                                                        |
| **2.7.x**      | **2.7** `OR` Unversioned (treated as `v2.6.0`, will be auto-updated to `v2.7`) |
| **<=2.6.x**    | Doesn't support versions                                                       |

{% hint style="info" %}
*non-versioned: configuration used in the 2.0.x-2.6.x releases.
{% endhint %}

## Changelog

### v3.0 Configuration and RR v2023.x.x

#### Reload plugin Update

{% hint style="warning" %}
The `reload` plugin has been removed from the default plugins list. Please use `*.pool.debug=true` instead.
{% endhint %}

#### OpenTelemetry Middleware Update

Starting from version **v2023.1.0**, the OpenTelemetry (OTEL) middleware configuration has been moved out of the HTTP
plugin to support its usage across multiple plugins, including HTTP, gRPC, jobs, and temporal. The OTEL middleware is now
configured using a top-level YAML key.

**RoadRunner 2.x:**

{% code title=".rr.yaml" %}

```yaml
# HTTP plugin settings.
http:
  ...
  middleware: [ "otel" ]
  otel:
    insecure: true
    compress: false
    client: http
    exporter: otlp
    custom_url: ""
    service_name: "rr_test"
    service_version: "1.0.0"
    endpoint: "127.0.0.1:4318"
```

{% endcode %}

**RoadRunner v2023.x.x:**

{% code title=".rr.yaml" %}

```yaml
http:
  ...
  middleware: [ "otel" ]

otel:
  insecure: true
  compress: false
  client: http
  exporter: otlp
  custom_url: ""
  service_name: "rr_test"
  service_version: "1.0.0"
  endpoint: "127.0.0.1:4318"
```

{% endcode %}

## Updating from `version: 2.7` to `version: 3`

To update your configuration from version 2.7 to version 3, follow these steps:

1. **Update the version number:** Change the `version` value from `2.7` to `3`.
2. **Relocate the `otel` middleware configuration:** If your configuration uses the `otel` middleware configuration
   within the `http` plugin, move it to the configuration root by cutting it from the `http` plugin and pasting it at
   the root level.
3. **Remove** the `reload` plugin configuration and if needed, use the `*.pool.debug=true` option instead.

### Upgrading to RoadRunner v2024.1.x

**There are no breaking changes in the userland API.**

#### Configuration

1. **`server.relay_timeout`** was deprecated and replaced internally with the Golang context timeout. Starting from `v2024.1.0` this option is no-op.

#### ⚠️ HTTP plugin ⚠️

{% hint style="warning" %}
Starting from `v2024.1.0` RR uses the protobuf encoded messages to send the payloads to the PHP workers via pipes and our protocol called `goridge`.
That means that you'll need to install the `protobuf` PHP extension to have the increased performance. PHP userland API remains the same.
{% endhint %}

{% hint style="info" %}
RR uses `pipes` and our custom protocol called `goridge` to communicate with the PHP worker.
However, the user payload was usually encoded with a `JSON` codec.
This approach led to issues,
such as broken raw binary payloads due to `JSON's` limitations and fields reordering, even in raw mode.
With the new protobuf codec, all these problems have been resolved.
Now, by using the `http.raw_body=true` option, you'll receive the payload completely untouched as it is.
Additionally, encoded images or raw binary payloads will no longer be escaped
(which previously led to broken images in payloads not encoded in base64).
{% endhint %}

#### Compatibility with RoadRunner PHP packages

1. The `spiral/http and spiral/worker` packages should be upgraded to version **3.5**. Old RR versions up to `v2023.3.12` are also supported with the latest versions of the PHP packages.

### Upgrading to RoadRunner v2024.2.x

{% hint style="info" %}
**There are no breaking changes in the userland API and no configuration changes.**
{% endhint %}

#### Information for Plugin Developers

{% hint style="warning" %}

Starting from `v2024.2.0`, all plugins were updated to version 5 due to the deprecation of the RoadRunner [SDK](https://github.com/roadrunner-server/sdk).
Pool, Workers, and Context tools are now available separately:

- The `util.CreateListener` API was moved to the [TcpListen](https://github.com/roadrunner-server/tcplisten) package with the same functionality.
- `util.Context` (and various context keys) was moved to the [Context](https://github.com/roadrunner-server/context) package.
- `Pool`, `Workers API`, and all `IPC` related stuff was moved to the [Pool](https://github.com/roadrunner-server/pool) package. You only need to replace `sdk/v4` with `pool` in your imports. For example, `github.com/roadrunner-server/sdk/v4/pool/static_pool/config` -> `github.com/roadrunner-server/pool/static_pool/config`.
- `events` (events bus package from `github.com/roadrunner-server/sdk/v4/events`) was moved to the [Events](https://github.com/roadrunner-server/events) package.

{% endhint %}
