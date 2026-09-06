# v3 Migration

This guide compares the v5 plugins in [RoadRunner v2025.1.15](https://github.com/roadrunner-server/roadrunner/blob/v2025.1.15/go.mod) with the v6 beta versions selected by the [RoadRunner development snapshot](https://github.com/roadrunner-server/roadrunner/blob/b0cccd917f001b6584eafdc04ad6ba69a97cbb69/go.mod). It covers application behavior, configuration, and custom Go plugins.

{% hint style="warning" %}
The v6 plugin line is in beta. These notes do not describe a published RoadRunner v3 binary. Changes that are not in the selected beta versions are listed under [Development Changes](#development-changes).
{% endhint %}

Keep `version: "3"` in `.rr.yaml`. That value identifies the configuration format. The inspected RR Go module remains `github.com/roadrunner-server/roadrunner/v2025`; its plugins use `/v6`. Source builds require Go 1.27.

## Upgrade Checks

| Area | Required check |
| --- | --- |
| HTTP middleware | Requests now enter middleware from left to right. Reverse an existing v5 list to retain its previous execution order. See [Middleware order](../http/http.md#middleware-order). |
| H2C | HTTP/1.1 `Upgrade: h2c` no longer switches protocols. Configure clients for [HTTP/2 prior knowledge](../http/http.md#http2). |
| Logging | Replace `file_logger_options` with `output`. Built-in rotation is removed. Production JSON changes `ts` to `time` and uses uppercase levels. Review parsers and output management in [Logger](../lab/logger.md). |
| Tracing | Replace the removed RR `zipkin` exporter with [OTLP](../lab/otel.md). Middleware spans no longer measure the whole downstream request; use the server span for request latency. |
| RPC | Goridge v4 rejects MessagePack. Use JSON, protobuf, Gob, or raw bytes as appropriate for the method. RPC still uses Goridge and `net/rpc`, not Connect. See [RPC compatibility](../php/rpc.md#v6-compatibility). |
| TCP plugin | The default RR build no longer includes the [TCP plugin](../plugins/tcp.md). A `tcp:` section cannot add it. This does not remove `tcp://` RPC transport. |
| Centrifuge | Remove calls to `centrifuge.RateLimit`; the RPC has no replacement in the plugin. Check [Centrifuge](../plugins/centrifuge.md) and the DTO migration before updating direct clients. |
| Job headers | `pool` is now a routing header. Rename application headers that use this name. External producers must supply a valid pool when [named worker pools](../queues/overview-queues.md#named-worker-pools-v6-beta) are enabled. |
| Environment files | A configured root `envfile` is now loaded without experimental mode. Supply the file or remove an unused setting; a missing file fails startup. See [Environment](../php/environment.md). |
| gRPC reflection | Reflection is registered automatically. Unary interceptors do not protect its streams. Review network access and mTLS in [gRPC](../grpc/grpc.md#server-reflection). |

## New Features

- [Jobs worker pools](../queues/overview-queues.md#named-worker-pools-v6-beta): assign pipelines to separate named pools. The existing single `jobs.pool` format remains available; do not configure it together with `jobs.pools`.
- [AMQP version 2 configuration](../queues/amqp.md#version-2-configuration-recommended): separate exchange and queue settings, with controls for declaration and binding. Legacy flat configuration remains supported. Runtime `jobs.Declare` still uses flat keys.
- [NSQ](../queues/nsq.md): a new bundled Jobs driver with topics, channels, discovery, acknowledgements, and delayed delivery. Its retry limit and lack of a dead-letter handoff require application failure handling.
- [gRPC reflection](../grpc/grpc.md#server-reflection): v1 and v1alpha service listing. Full PHP-service descriptors require a [Protoreg plugin](../grpc/protoreg.md). Unary interceptors already existed in v5.3.0; they are not a new v6 feature.
- [Trusted proxy headers](../http/proxy.md): select and order the forwarding headers that a trusted proxy may supply. An empty list restores the defaults; it does not disable header trust.
- [Temporal](../workflow/temporal.md): configurable worker heartbeats and [dynamic workflows](../workflow/worker.md). Worker heartbeats are separate from activity heartbeats. Dynamic registration requires a compatible PHP SDK.
- [Centrifuge](../plugins/centrifuge.md): forwards `NotifyCacheEmpty` events to PHP. Enable this only with matching DTO and handler support.

## Bug Fixes

| Component | User-visible change |
| --- | --- |
| HTTP | Informational responses no longer corrupt the final response status or headers. Worker status 101 is ignored. Request-parsing errors return 400, and debug error text is HTML-escaped. Bracketed IPv6 HTTPS addresses and redirects work. |
| gRPC | `max_connection_age_grace` now uses its configured value instead of `max_connection_age`. Check connection-draining settings. Standard `google.rpc` error details are included in logs; check them for sensitive data. |
| X-Sendfile | Empty files no longer enter the read loop. File responses use `application/octet-stream`, even when the worker supplied another content type. |
| Fileserver | Initialization rejects missing addresses, empty route lists, and invalid prefixes. Listener failure and shutdown no longer retain the plugin lock. |
| Jobs | Pipeline restart no longer destroys the replacement pipeline. Empty-queue pollers and shutdown timeout handling no longer prevent clean shutdown. |
| NATS | Stopping a pipeline no longer purges its stream. Retry headers are retained. Configure retention and make workers tolerate redelivery. |
| SQS | FIFO retries use a fresh deduplication ID, so the broker does not discard a retry as the original message. The application job ID stays unchanged. Priority is read from message attributes. |
| Kafka | Explicit partition/offset consumption is now applied. Omit `consumer_options.topics` when using `consume_partitions`. Shutdown releases blocked rebalances. |
| AMQP | Private root CAs are used for server verification, including reconnects. Supported integer priority headers no longer cause type-assertion panics. |
| Beanstalk | Serialized jobs retain headers and trace context. Statistics now describe the pipeline's tube, not the entire server. Old messages cannot recover headers that were never stored. |
| Google Pub/Sub | Pause cancels receiving. Existing-topic startup and dynamically declared dead-letter settings are handled correctly. Existing subscription policies still require separate updates. |
| KV | Unknown drivers fail startup instead of being skipped. Memory deletion no longer blocks on duplicate timer cancellation. Redis expiration and BoltDB commit errors are returned instead of hidden. |
| BoltDB Jobs | Recovery removes stale in-flight records after returning jobs to the queue. Delayed jobs are dispatched after commit. Workers must still tolerate repeated deliveries. |
| Server | Invalid worker users fail initialization. `server.on_init.env` overrides inherited values. Use command sequences for arguments containing spaces; scalar commands do not parse shell quotes. |
| Service | A failed replacement removes the service entry and requests a stop for replacements already started. Fix the command and create the service again. Restart is not rolling or atomic. |
| Temporal | Polling stops before PHP pools are destroyed. An ordinary activity-worker exit no longer resets the whole activity pool. |
| Centrifuge | Missing gRPC metadata no longer causes a panic. A missing worker pool reports unavailable status. |
| Metrics and status | Implicit HTTP success is labeled `200`, not `-1`. Jobs distinguishes successful and requeued jobs, and totals use counters. Negative counter increments return errors. Shutdown readiness uses `unavailable_status_code`. |
| Locks and RPC | Lock wait paths release their mutexes correctly. Malformed RPC offsets and invalid response metadata return errors instead of causing slice or type-assertion panics. |

Review [Jobs metrics](../lab/metrics.md) before updating dashboards. The request-duration histograms still measure the full handler execution, unlike the shorter middleware spans.

## Custom Go Plugins

The API repositories have separate roles:

| Repository | Purpose |
| --- | --- |
| `api` | Protobuf schema source. It is no longer the Go module imported by plugins. |
| `api-go/v6` | Generated Go messages and gRPC bindings. Imports no longer contain `/build/`. |
| `api-plugins/v6` | Handwritten Go contracts for Jobs, KV, logging, locks, and status. |

Update custom plugins to `*slog.Logger`, the new import paths, and context-aware constructors and storage methods. The pool module is now `pool/v2`; Goridge is `goridge/v4`. Use the [import and signature migration table](../customization/plugin.md#v6-migration) and the updated [Jobs driver](../customization/jobs-driver.md) and [middleware](../customization/middleware.md) examples.

API relocation alone does not require a PHP worker-loop rewrite or change the Goridge frame version. Lock DTO namespaces and some Centrifugo messages have separate source-level changes; see [DTO compatibility](../customization/plugin.md#dto-compatibility). Do not use the Connect clients or general v2 DTO packages from intermediate betas.

RR still pins `endure/v2 v2.6.2` and `tcplisten v1.5.2`, as it did in v2025.1.15. There is no new Endure lifecycle, dependency-injection, or tcplisten caller migration in this comparison.

## Beta Limits

| Component | Limitation and action |
| --- | --- |
| `pool/v2 v2.0.0-beta.1` | Scale-up does not retry the request that triggered allocation. That request can fail even after new workers start. Size the base pool and allocation timeout for the expected load. The initial limit is 500 workers; combined base and dynamic capacity is 2048. See [Automatic scaling](../php/auto-scaling.md). |
| `jobs/v6 v6.0.0-beta.10` | An explicit `jobs.pool` with omitted or zero `num_workers` produces only two pollers. Set `num_workers` explicitly; setting `num_pollers` does not override the derived count. |
| BoltDB Jobs | `boltdb/v6 v6.0.0-beta.5` loses the `pool` header during storage, including for newly published jobs. Keep a single `jobs.pool` when consuming BoltDB pipelines with jobs beta.10. Named pools cannot route these jobs. |
| `memory/v6 v6.0.0-beta.5` | `MExpire` with a past deadline or less than one second remaining can replace a value and remove its expiration. Use `Delete` for immediate removal. See [Memory KV](../kv/memory.md). |
| `service/v6 v6.0.0-beta.8` | After an automatic restart, `service.Restart` can start replacements before the old processes finish. Do not rely on it for exclusive process replacement when `remain_after_exit` is enabled. |

## Development Changes

These changes are not in the beta dependencies selected by the RR snapshot above. Use a custom build containing the linked change, or wait for a version that includes it.

| Component | Pending change |
| --- | --- |
| Unix sockets | Optional [mode, owner, and group settings](config.md#unix-socket-attributes) for HTTP, FastCGI, RPC, gRPC, named TCP servers, Centrifuge proxy listeners, Fileserver, and worker relays. Requires the corresponding plugin changes with `tcplisten v1.6.0`. Existing defaults remain unchanged. |
| HTTP | [PROXY protocol support](https://github.com/roadrunner-server/http/commit/8bebd3b) for plain HTTP and HTTPS listeners. It requires trusted proxy addresses and changes how load balancers and readiness checks connect. See the [development configuration](../http/http.md#development-proxy-protocol). |
| Static | [Prefix and cache controls](https://github.com/roadrunner-server/static/commit/030052b), normalized path checks, and revised ETags. Positive hits still open and stat files; cached misses can delay newly created files. See [Static files](../http/static.md). |
| Pool | [Allocation and shutdown fixes](https://github.com/roadrunner-server/pool/commit/2ae6f57): retry acquisition after scale-up, cancel spawning during shutdown, reap failed or late workers, and correct supervisor state transitions. Do not assume beta.1 contains these fixes. |
| Temporal | [Shutdown deadlock fix](https://github.com/temporalio/roadrunner-temporal/commit/bcadca6) for concurrent activity-heartbeat RPCs. It is not in `v6.0.0-beta.1`. |
| Velox v3 | New module-based plugin configuration, replacements/exclusions, version-pin checks, and deterministic build inputs. Windows targets and the remote build server are removed. No v3 tag is published; the [build guide](../customization/build.md) pins the development revision. |
| Zstd | A separate [response-compression middleware](../http/zstd.md), not yet in the inspected default RR build. Include and register the plugin before selecting `http.middleware: ["zstd"]`. |
