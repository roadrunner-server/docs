# Table of contents

## 😃 General

- [What is RoadRunner?](intro/about.md)
- [Features](README.md)
- [Quick Start](intro/quick-start.md)
- [Installation](intro/install.md)
- [Configuration](intro/config.md)
- [Contributing](intro/contributing.md)
- [Upgrade and Compatibility](intro/compatibility.md)

## 👷 PHP Worker

- [Worker](php/worker.md)
- [Workers pool](php/pool.md)
- [Developer mode](php/developer.md)
- [Code Coverage](php/codecoverage.md)
- [Debugging](php/debugging.md)
- [Environment](php/environment.md)
- [Manual workers scaling](php/manual-scaling.md)
- [Auto workers scaling](php/auto-scaling.md)
- [RPC](php/rpc.md)

## 🚀 Customization

- [Building RR with a custom plugin](customization/build.md)
- [Integrating with Golang Apps](customization/embedding.md)
- [Writing a Middleware](customization/middleware.md)
- [Writing a Jobs Driver](customization/jobs-driver.md)
- [Writing a Plugin](customization/plugin.md)
- [Events Bus](customization/events-bus.md)

## 🔌 Plugins

- [Intro into Plugins](plugins/intro.md)
- [Centrifuge (WebSockets)](plugins/centrifuge.md)
- [Service (Systemd)](plugins/service.md)
- [Configuration](plugins/config.md)
- [Server](plugins/server.md)
- [Locks](plugins/locks.md)
- [TCP](plugins/tcp.md)

## 🌐 Community Plugins

- [Intro into Community Plugins](community-plugins/intro.md)
- [Circuit Breaker](community-plugins/circuit-breaker.md)
- [SendRemoteFile](community-plugins/sendremotefile.md)
- [RFC 7234 Cache](community-plugins/cache.md)

## 📡 App Server

- [Production Usage](app-server/production.md)
- [RoadRunner with NGINX](app-server/nginx-with-rr.md)
- [RR as AWS Lambda](app-server/aws-lambda.md)
- [Docker Images](app-server/docker.md)
- [Kubernetes + Argo CD](app-server/kubernetes-argocd.md)
- [CLI Commands](app-server/cli.md)
- [Systemd](app-server/systemd.md)

## 🔐 Key-Value

- [Intro into KV](kv/overview-kv.md)
- [Memcached](kv/memcached.md)
- [In-Memory](kv/memory.md)
- [BoltDB](kv/boltdb.md)
- [Redis](kv/redis.md)

## 📦 Queues and Jobs

- [Intro into Jobs](queues/overview-queues.md)
- [Google Pub/Sub](queues/google-pub-sub.md)
- [Beanstalk](queues/beanstalk.md)
- [In-Memory](queues/memory.md)
- [RabbitMQ](queues/amqp.md)
- [BoltDB](queues/boltdb.md)
- [Kafka](queues/kafka.md)
- [NATS](queues/nats.md)
- [SQS](queues/sqs.md)

## 🕸️ HTTP

- [Intro into HTTP](http/http.md)
- [Headers and CORS](http/headers.md)
- [Proxy IP parser](http/proxy.md)
- [Static files](http/static.md)
- [X-Sendfile](http/sendfile.md)
- [Streaming](http/resp-streaming.md)
- [gzip](http/gzip.md)

## ⚡️ gRPC

- [Intro into gRPC](grpc/grpc.md)
- [Interceptors](grpc/interceptors.md)
- [Protoreg](grpc/protoreg.md)

## 📈 Logging and Observability

- [OpenTelemetry](lab/otel.md)
- [HealthChecks](lab/health.md)
- [Access Logs](lab/accesslogs.md)
- [AppLogger](lab/applogger.md)
- [Metrics](lab/metrics.md)
- [Grafana](lab/dashboards/dashboards.md)
- [Logger](lab/logger.md)

## 🔀 Workflow Engine

- [Temporal.io](workflow/temporal.md)
- [Worker](workflow/worker.md)

## 🧩 Integrations

- [Migration from RRv1 to RRv2](integration/migration.md)
- [Spiral Framework](integration/spiral.md)
- [Yii](integration/yii.md)
- [Symfony](integration/symfony.md)
- [Laravel](integration/laravel.md)
- [ChubbyPHP](integration/chubbyphp.md)

## 🧪 Experimental Features

- [List of the Experimental Features](experimental/experimental.md)

## 🚨 Error codes

- [CRC validation failed](known-issues/stdout-crc.md)
- [Allocate Timeout](known-issues/allocate-timeout.md)

## 📚 Releases

- [v2025.1.13](releases/v2025-1-13.md)
- [v2025.1.12](releases/v2025-1-12.md)
- [v2025.1.11](releases/v2025-1-11.md)
- [v2025.1.9](releases/v2025-1-9.md)
- [v2025.1.8](releases/v2025-1-8.md)
- [v2025.1.7](releases/v2025-1-7.md)
- [v2025.1.5](releases/v2025-1-5.md)
- [v2025.1.4](releases/v2025-1-4.md)
- [v2025.1.3](releases/v2025-1-3.md)
- [v2025.1.2](releases/v2025-1-2.md)
- [v2025.1.1](releases/v2025-1-1.md)
- [v2025.1.0](releases/v2025-1-0.md)
