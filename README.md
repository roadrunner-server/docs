<figure>
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://github.com/roadrunner-server/.github/assets/8040338/cf1bfcf2-b787-426d-80f5-2862bb2a39b2">
    <img align="center" src="https://github.com/roadrunner-server/.github/assets/8040338/c4b971fd-b84f-406d-b850-0a4f072a5885">
  </picture>
</figure>

<p align="center">
 <a href="https://packagist.org/packages/spiral/roadrunner"><img src="https://poser.pugx.org/spiral/roadrunner/version"></a>
 <a href="https://pkg.go.dev/github.com/roadrunner-server/roadrunner/v2024?tab=doc"><img src="https://godoc.org/github.com/roadrunner-server/roadrunner/v2023?status.svg"></a>
    <a href="https://twitter.com/spiralphp"><img src="https://img.shields.io/twitter/follow/spiralphp?style=social"></a>
    <a href="https://codecov.io/gh/roadrunner-server/roadrunner/"><img src="https://codecov.io/gh/roadrunner-server/roadrunner/branch/master/graph/badge.svg"></a>
 <a href="https://github.com/roadrunner-server/roadrunner/actions"><img src="https://github.com/roadrunner-server/roadrunner/workflows/rr_cli_tests/badge.svg" alt=""></a>
 <a href="https://goreportcard.com/report/github.com/roadrunner-server/roadrunner/v2"><img src="https://goreportcard.com/badge/github.com/roadrunner-server/roadrunner/v2"></a>
 <a href="https://discord.gg/TFeEmCs"><img src="https://img.shields.io/badge/discord-chat-magenta.svg"></a>
 <a href="https://packagist.org/packages/spiral/roadrunner"><img src="https://img.shields.io/packagist/dd/spiral/roadrunner?style=flat-square"></a>
    <img alt="All releases" src="https://img.shields.io/github/downloads/roadrunner-server/roadrunner/total">
</p>

# Join our discord server: [Link](https://discord.gg/TFeEmCs)

RoadRunner is an open-source (MIT licensed) high-performance PHP application server and process manager written in Go and powered with plugins ❤️.
It supports running as a service with the ability to extend its functionality on a per-project basis with plugins and is designed to replace traditional Nginx+FPM setups with enhanced performance and versatility. With its wide range of features, RoadRunner is a production-ready solution that is PCI DSS compliant, ensuring the security and safety
of your web applications.

RoadRunner includes [PSR-7](https://www.php-fig.org/psr/psr-7), [PSR-17](https://www.php-fig.org/psr/psr-17) compatible HTTP and `HTTP(S)/2/3/fCGI` servers and can be used to replace classic Nginx+FPM setups with much greater performance and flexibility. `HTTP(S)/2/3/fCGI` servers as just one of its many available plugins, but its capabilities extend far beyond:

- Queue drivers: RabbitMQ, Kafka, SQS, Beanstalk, NATS, In-Memory.
- KV drivers: Redis, Memcached, BoltDB, In-Memory.
- OpenTelemetry protocol support (`gRPC`, `http`, `jaeger`).
- Workflows engine via [Temporal](https://temporal.io)
- `gRPC` server. For increased speed, the `protobuf` extension can be used.
- `HTTP(S)/2/3` and `fCGI` server features **automatic TLS management**, **103 Early Hints** support and middleware like: Static, Headers, gzip, prometheus (metrics), send (x-sendfile), OTEL, proxy_ip_parser, etc.
- Embedded distribute lock plugin which manages access to shared resources.
- Metrics server (you might easily expose your own).
- WebSockets and Broadcast via [Centrifugo](https://centrifugal.dev) server.
- Systemd-like services manager with auto-restarts, execution time limiter, etc.
- Embedded supervisor for your PHP workers with execution, max working time, and memory TTLs.
- Compatible with Windows, WSL2, FreeBSD, GNU/Linux, etc.
- And more 😉

If you have a feature request in mind, you can check
out [GitHub issues](https://github.com/roadrunner-server/roadrunner/issues) page. Here you'll find a list of open
feature requests. The RoadRunner community is active and responsive, so feel free to join the discussion on
our [Discord channel](https://discord.gg/spiralphp) or [contribute](intro/contributing.md) to the project.

<a href="https://discord.gg/spiralphp"><img src="https://img.shields.io/badge/discord-chat-magenta.svg"></a>

With the support of the community, RoadRunner will continue to grow and evolve to meet the needs of modern web
development.

## Useful articles
- [Scaling PHP applications with RoadRunner](https://betterstack.com/community/guides/scaling-php/php-roadrunner)
- [RoadRunner for PHP applications](https://butschster.medium.com/roadrunner-an-underrated-powerhouse-for-php-applications-46410b0abc)
- [Turbo charge for your PHP applications](https://davegebler.com/post/php/turbo-charge-your-php-applications-with-roadrunner)
