# Upgrade from v2025 to v3

This guide lists the major upgrade steps from RoadRunner v2025 with v5 plugins to RoadRunner v3 with v6 plugins. The linked reference pages contain the configuration examples and full behavior descriptions. See the [v3 release notes](../releases/v3-0-0.md) for new features.

Keep `version: "3"` at the root of `.rr.yaml`. Source builds require Go 1.27 or later.

## Application and Configuration Changes

| Area | Upgrade action |
| --- | --- |
| HTTP middleware | Requests enter middleware from left to right. Reverse an existing v5 list to keep its execution order. See [Middleware order](../http/http.md#middleware-order). |
| HTTP/2 | Use HTTP/2 prior knowledge for cleartext connections. HTTP/1.1 `Upgrade: h2c` is no longer supported. See [HTTP/2](../http/http.md#http2). |
| AMQP | Move broker settings to named connections. Set `config.connection` for each pipeline. Move exchange and queue settings into their nested sections and remove AMQP `config.version`. See [RabbitMQ](../queues/amqp.md). |
| Logging | Replace `file_logger_options` with `output`. Use an external tool for log rotation. Update JSON parsers for `time` and uppercase levels. Output is asynchronous; a full queue drops new records. See [Logger](../lab/logger.md). |
| Tracing | Replace the removed Zipkin exporter with OTLP. Use the server span to measure request latency; middleware spans end before the next handler starts. See [OpenTelemetry](../lab/otel.md). |
| RPC | Replace MessagePack with a supported codec. Goridge v4 supports JSON, protobuf, Gob, and raw payloads. See [RPC](../php/rpc.md). |
| Jobs | Reserve the `pool` job header for named-pool routing. Configure either `jobs.pool` or `jobs.pools`. See [Jobs](../queues/overview-queues.md). |
| NATS | Review stream retention and repeated-job handling. Stopping a pipeline now preserves messages unless stream deletion is enabled. A new consumer can receive retained messages again after restart. See [NATS retention](../queues/nats.md#delete-stream-on-stop). |
| Centrifuge | Remove calls to `centrifuge.RateLimit`. Update direct clients for the supported DTOs and events. See [Centrifuge](../plugins/centrifuge.md). |
| Environment files | A root `envfile` is loaded without experimental mode. Supply the configured file or remove the setting. See [Environment](../php/environment.md). |
| Server initialization | Values in `server.on_init.env` override inherited environment values. Use a command sequence for arguments that contain spaces. See [Server](../plugins/server.md). |
| Metrics | Update Jobs dashboards for counter types and separate requeue totals. See [Metrics](../lab/metrics.md). |
| TCP plugin | Add TCP to a custom build if the application uses the `tcp:` plugin. It is no longer in the default container. See [TCP](../plugins/tcp.md). |

## Custom Go Plugins

- Update RoadRunner library imports to `github.com/roadrunner-server/roadrunner/v3`.
- Replace imports from `api/v4` with generated bindings from `api-go/v6` or contracts from `api-plugins/v6`. The `api` repository contains protobuf schemas.
- Update plugin loggers to `*slog.Logger` and use the context-aware Jobs and KV contracts.
- Update worker pool imports to `pool/v2` and Goridge imports to `goridge/v4`.
- Update custom builds to the Velox v3 module configuration.

See [Writing a Plugin](../customization/plugin.md), [Writing a Jobs Driver](../customization/jobs-driver.md), and [Building RR with a Custom Plugin](../customization/build.md) for imports, signatures, and build examples.
