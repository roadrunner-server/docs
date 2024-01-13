<p align="center">
 <a href="https://roadrunner.dev" target="_blank">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://github.com/roadrunner-server/.github/assets/8040338/cf1bfcf2-b787-426d-80f5-2862bb2a39b2">
    <img align="center" src="https://github.com/roadrunner-server/.github/assets/8040338/c4b971fd-b84f-406d-b850-0a4f072a5885">
  </picture>
</a>
</p>
<p align="center">
 <a href="https://packagist.org/packages/spiral/roadrunner"><img src="https://poser.pugx.org/spiral/roadrunner/version"></a>
 <a href="https://pkg.go.dev/github.com/roadrunner-server/roadrunner/v2023?tab=doc"><img src="https://godoc.org/github.com/roadrunner-server/roadrunner/v2023?status.svg"></a>
    <a href="https://twitter.com/spiralphp"><img src="https://img.shields.io/twitter/follow/spiralphp?style=social"></a>
    <a href="https://codecov.io/gh/roadrunner-server/roadrunner/"><img src="https://codecov.io/gh/roadrunner-server/roadrunner/branch/master/graph/badge.svg"></a>
 <a href="https://github.com/roadrunner-server/roadrunner/actions"><img src="https://github.com/roadrunner-server/roadrunner/workflows/rr_cli_tests/badge.svg" alt=""></a>
 <a href="https://goreportcard.com/report/github.com/roadrunner-server/roadrunner/v2"><img src="https://goreportcard.com/badge/github.com/roadrunner-server/roadrunner/v2"></a>
 <a href="https://discord.gg/TFeEmCs"><img src="https://img.shields.io/badge/discord-chat-magenta.svg"></a>
 <a href="https://packagist.org/packages/spiral/roadrunner"><img src="https://img.shields.io/packagist/dd/spiral/roadrunner?style=flat-square"></a>
    <img alt="All releases" src="https://img.shields.io/github/downloads/roadrunner-server/roadrunner/total">
</p>

RoadRunner is an open-source (MIT licensed) high-performance PHP application server, process manager written in Go and powered with plugins ❤️.
It supports running as a service with the ability to extend its functionality on a per-project basis with plugins.

RoadRunner includes PSR-7/PSR-17 compatible HTTP and `HTTP(S)/2/3` servers and can be used to replace classic Nginx+FPM setup
with much greater performance and flexibility. `HTTP(S)/2/3` servers as just one of its many available plugins, but its capabilities extend far beyond:

- Queue drivers: RabbitMQ, Kafka, SQS, Beanstalk, NATS, In-Memory.
- KV drivers: Redis, Memcached, BoltDB, In-Memory.
- OpenTelemetry protocol support (`gRPC`, `http`, `jaeger`).
- Workflows manager via [Temporal](https://temporal.io)
- `gRPC` server. For increased speed, the `protobuf` extension can be used.
- `HTTP(S)/2/3` server features **automatic TLS management**, **103 Early Hints** support and middleware like: Static, Headers, gzip, prometheus (metrics), send (x-sendfile), OTEL, proxy_ip_parser, etc.
- Embedded distribute lock plugin which manages access to shared resources.
- Metrics server (you might easily expose your own).
- WebSockets and Broadcast via [Centrifugo](https://centrifugal.dev) server.
- Systemd-like services manager with auto-restarts, execution time limiter, etc.
- And more 😉

# Join our discord server: [Link](https://discord.gg/TFeEmCs)
