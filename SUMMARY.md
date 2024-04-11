# Table of contents

## 😃 General

* [Intro into RoadRunner](intro/about.md)
* [Features](README.md)
* [Quick Start](intro/quick-start.md)
* [Installation](intro/install.md)
* [Configuration](intro/config.md)
* [Contributing](intro/contributing.md)
* [Upgrade and Compatibility](intro/compatibility.md)

## 👷 PHP Worker

* [Dynamic Workers Scaling](php/scaling.md)
* [Developer mode](php/developer.md)
* [Environment](php/environment.md)
* [Debugging](php/debugging.md)
* [Worker](php/worker.md)
* [Workers pool](php/pool.md)
* [RPC](php/rpc.md)

## 🚀 Customization

* [Building RR with a custom plugin](customization/build.md)
* [Integrating with Golang Apps](customization/embedding.md)
* [Writing a Middleware](customization/middleware.md)
* [Writing a Plugin](customization/plugin.md)
* [Events Bus](customization/events-bus.md)

## 🔌 Plugins

* [Intro into Plugins](plugins/intro.md)
* [Centrifuge (WebSockets)](plugins/centrifuge.md)
* [Service (Systemd)](plugins/service.md)
* [Configuration](plugins/config.md)
* [Server](plugins/server.md)
* [Locks](plugins/locks.md)
* [gRPC](plugins/grpc.md)
* [TCP](plugins/tcp.md)

## 📡 App Server

* [Production Usage](app-server/production.md)
* [RoadRunner with NGINX](app-server/nginx-with-rr.md)
* [RR as AWS Lambda](app-server/aws-lambda.md)
* [Docker Images](app-server/docker.md)
* [CLI Commands](app-server/cli.md)
* [Systemd](app-server/systemd.md)

## 🔐 Key-Value

* [Intro into KV](kv/overview-kv.md)
* [Memcached](kv/memcached.md)
* [In-Memory](kv/memory.md)
* [BoltDG](kv/boltdb.md)
* [Redis](kv/redis.md)

## 📦 Queues and Jobs

* [Intro into Jobs](queues/overview-queues.md)
* [Beanstalk](queues/beanstalk.md)
* [In-Memory](queues/memory.md)
* [RabbitMQ](queues/amqp.md)
* [BoltDB](queues/boltdb.md)
* [Kafka](queues/kafka.md)
* [NATS](queues/nats.md)
* [SQS](queues/sqs.md)

## 🕸️ HTTP

* [Intro into HTTP](http/http.md)
* [Headers and CORS](http/headers.md)
* [Proxy IP parser](http/proxy.md)
* [RFC 7234 cache](http/cache.md)
* [Static files](http/static.md)
* [X-Sendfile](http/sendfile.md)
* [Streaming](http/resp-streaming.md)
* [gzip](http/gzip.md)

## 📈 Logging and Observability

* [OpenTelemetry](lab/otel.md)
* [HealthChecks](lab/health.md)
* [Access Logs](lab/accesslogs.md)
* [AppLogger](lab/applogger.md)
* [Metrics](lab/metrics.md)
* [Grafana](lab/dashboards/dashboards.md)
* [Logger](lab/logger.md)

## 🔀 Workflow Engine

* [Temporal.io](workflow/temporal.md)
* [Worker](workflow/worker.md)

## 🧩 Integrations

* [Migration from RRv1 to RRv2](integration/migration.md)
* [Spiral Framework](integration/spiral.md)
* [Yii](integration/yii.md)
* [Symfony](integration/symfony.md)
* [Laravel](integration/laravel.md)
* [ChubbyPHP](integration/chubbyphp.md)

## 🧪 Experimental Features

* [List of the Experimental Features](experimental/experimental.md)

## 🚨 Error codes

* [CRC validation failed](known-issues/stdout-crc.md)
* [Allocate Timeout](known-issues/allocate-timeout.md)

## 📚 Releases

* [v2024.1.0](releases/v2024-1-0.md)
