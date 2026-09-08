# v3 Migration

This guide covers migration from [RoadRunner v2025.1.15](https://github.com/roadrunner-server/roadrunner/blob/v2025.1.15/go.mod) with v5 plugins to RoadRunner v3 with v6 plugins. It covers application behavior, configuration, and custom Go plugins.

Keep `version: "3"` in `.rr.yaml`. That value identifies the configuration format, not the RoadRunner release. The plugins use `/v6` Go module paths. Source builds require Go 1.27.

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
| Job headers | `pool` is now a routing header. Rename application headers that use this name. External producers must supply a valid pool when [named worker pools](../queues/overview-queues.md#named-worker-pools) are enabled. |
| AMQP (development/unreleased) | Move global broker settings to named connections. Set `config.connection` on every AMQP pipeline. Use nested exchange and queue settings without `config.version`. See [AMQP migration](#amqp-configuration-development). |
| Environment files | A configured root `envfile` is now loaded without experimental mode. Supply the file or remove an unused setting; a missing file fails startup. See [Environment](../php/environment.md). |
| gRPC reflection | Reflection is registered automatically. Unary interceptors do not protect its streams. Review network access and mTLS in [gRPC](../grpc/grpc.md#server-reflection). |

## New Features

- [Jobs worker pools](../queues/overview-queues.md#named-worker-pools): assign pipelines to separate named pools. The existing single `jobs.pool` format remains available; do not configure it together with `jobs.pools`.
- [AMQP pipeline configuration](../queues/amqp.md#pipeline-configuration): separate exchange and queue settings, with controls for declaration and binding. The development configuration requires nested sections and named connections. Runtime `jobs.Declare` remains a flat string map.
- [NSQ](../queues/nsq.md): a new bundled Jobs driver with topics, channels, discovery, acknowledgements, and delayed delivery. Its retry limit and lack of a dead-letter handoff require application failure handling.
- [gRPC reflection](../grpc/grpc.md#server-reflection): v1 and v1alpha service listing. Full PHP-service descriptors require a [Protoreg plugin](../grpc/protoreg.md). Unary interceptors already existed in v5.3.0; they are not a new v6 feature.
- [Trusted proxy headers](../http/proxy.md): select and order the forwarding headers that a trusted proxy may supply. An empty list restores the defaults; it does not disable header trust.
- [Temporal](../workflow/temporal.md): configurable worker heartbeats and [dynamic workflows](../workflow/worker.md). Worker heartbeats are separate from activity heartbeats. Dynamic registration requires a compatible PHP SDK.
- [Centrifuge](../plugins/centrifuge.md): forwards `NotifyCacheEmpty` events to PHP. Enable this only with matching DTO and handler support.
- [Unix socket attributes](config.md#unix-socket-attributes): configure mode, owner, and group independently for HTTP, FastCGI, RPC, gRPC, named TCP servers, Centrifuge proxy listeners, Fileserver, and worker relays. Omitted settings keep the existing defaults.
- [PROXY protocol](../http/http.md#development-proxy-protocol): accept trusted proxy addresses on plain HTTP and HTTPS listeners. Configure load balancers and readiness checks to send the required PROXY header.
- [Static file controls](../http/static.md): configure URL prefixes, cache lifetimes, and cache limits. The middleware normalizes paths before access checks and uses revised ETags. Positive cache hits still open and stat files. Cached misses can delay newly created files.
- [Zstd middleware](../http/zstd.md): add response compression with Zstandard. Include and register the plugin before selecting `http.middleware: ["zstd"]`.
- [HTTP rate limiting](../http/rate-limiter.md): upcoming bundled middleware with global, IP, or header keys, bounded process-local state, and `429` responses with `Retry-After`. The pinned source build does not include it.

## AMQP Configuration (Development)

Named connections and nested-only static configuration are development/unreleased changes for the next major release. The pinned [RR source build](install.md) at `b0cccd9` uses AMQP `v6.0.0-beta.9` and includes neither change. That beta allowed both flat and nested static configuration. Select an AMQP dependency with both changes before using the new configuration.

Move `amqp.addr` to `amqp.<name>.addr`. Move optional `amqp.tls` to `amqp.<name>.tls`. Each connection requires an explicit address. Every YAML AMQP pipeline requires `config.connection` with a configured name. Top-level `amqp.addr` and `amqp.tls` are not supported. There is no implicit default connection or localhost fallback.

Keep root `version: "3"`. Remove AMQP `config.version`. Static AMQP configuration uses only nested `exchange` and `queue` sections. Scalar `exchange` or `queue` values fail to decode. Flat flags do not set nested values. Move old flat entity settings with the [AMQP migration table](../queues/amqp.md#migration).

Runtime `jobs.Declare` stays a flat string map. Send `connection`, or use `queueHeaders: ['rr_connection' => 'brokerB']` with the existing PHP `AMQPCreateInfo` API. No PHP package change is required. See [AMQP runtime declarations](../queues/amqp.md#runtime--rpc-jobsdeclare) for precedence and reserved-key removal, and [named connections](../queues/amqp.md#named-connections-development) for separate consume and publish pipelines.

## Bug Fixes

| Component | User-visible change |
| --- | --- |
| HTTP | Informational responses no longer corrupt the final response status or headers. Worker status 101 is ignored. Request-parsing errors return 400, and debug error text is HTML-escaped. Bracketed IPv6 HTTPS addresses and redirects work. |
| gRPC | `max_connection_age_grace` now uses its configured value instead of `max_connection_age`. Check connection-draining settings. Standard `google.rpc` error details are included in logs; check them for sensitive data. |
| X-Sendfile | Empty files no longer enter the read loop. File responses use `application/octet-stream`, even when the worker supplied another content type. |
| Fileserver | Initialization rejects missing addresses, empty route lists, and invalid prefixes. Listener failure and shutdown no longer retain the plugin lock. |
| Pool | Worker acquisition retries after dynamic scale-up. Shutdown cancels worker allocation. Allocation cleanup reaps failed and late workers. Supervisor state transitions are atomic. |
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
| Temporal | Polling stops before PHP pools are destroyed. An ordinary activity-worker exit no longer resets the whole activity pool. Activity-heartbeat RPCs no longer deadlock with shutdown. |
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

API relocation alone does not require a PHP worker-loop rewrite or change the Goridge frame version. Lock DTO namespaces and some Centrifugo messages have separate source-level changes; see [DTO compatibility](../customization/plugin.md#dto-compatibility). Use the v1 DTO packages listed in the migration table.

The Endure lifecycle and dependency-injection contracts are unchanged. Existing `tcplisten.CreateListener` callers need no changes. Use `tcplisten.CreateListenerWithOptions` from `tcplisten v1.6.0` or later to configure filesystem Unix socket attributes.

Velox v3 uses [module-based build configuration](../customization/build.md) with replacements, exclusions, version-pin checks, and deterministic build inputs. Windows targets and the remote build server are removed.

## Known Limits

| Component | Limitation and action |
| --- | --- |
| Pool | The initial `num_workers` limit is 500 per pool. Base and additional workers cannot exceed 2048 in total. `dynamic_allocator.max_workers` counts additional workers. |
| Jobs | An explicit `jobs.pool` with omitted or zero `num_workers` produces only two pollers. Set a positive `num_workers` value. Setting `num_pollers` does not override the derived count. |
| BoltDB Jobs | Storage does not preserve the `pool` header, including for newly published jobs. Use a single `jobs.pool` with BoltDB pipelines. Named pools cannot route these jobs. |
| Memory KV | `kv.MExpire` with a past deadline or less than one second remaining can replace a value and remove its expiration. Use `kv.Delete` for immediate removal. See [Memory KV](../kv/memory.md). |
| Service | After an automatic restart, `service.Restart` can start replacements before the old processes finish. Do not rely on it for exclusive process replacement when `remain_after_exit` is enabled. |
