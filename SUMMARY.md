# Table of contents

## 😃 General

* [Intro into RoadRunner](intro/about.md)
* [Quick Start](intro/quick-start.md)
* [Features](README.md)
* [Installation](intro/install.md)
* [Configuration](intro/config.md)
* [Contributing](intro/contributing.md)

## 👷 PHP Worker

* [Dynamic Workers Scaling](php/scaling.md)
* [Developer mode](php/developer.md)
* [Environment](php/environment.md)
* [Debugging](php/debugging.md)
* [Worker](php/worker.md)
* [RPC](php/rpc.md)

## 🛠️ Customization

* [Building RR with a custom plugin](customization/build.md)
* [Integrating with Golang Apps](customization/embedding.md)
* [Writing a Middleware](customization/middleware.md)
* [Writing a Plugin](customization/plugin.md)
* [Events Bus](customization/events-bus.md)

## 📡 App Server

* [Production Usage](app-server/production.md)
* [RoadRunner with NGINX](app-server/nginx-with-rr.md)
* [RR as AWS Lambda](app-server/aws-lambda.md)
* [Docker Images](app-server/images.md)
* [CLI Commands](app-server/cli.md)
* [Systemd](app-server/systemd.md)

## 🔌 Plugins

* [Intro into Plugins](plugins/intro.md)
* [Centrifuge (WebSockets)](plugins/centrifuge.md)
* [Service (Systemd)](plugins/service.md)
* [Queues and Jobs](queues/overview-queues.md)
* [Configuration](plugins/config.md)
* [Server](plugins/server.md)
* [Locks](plugins/locks.md)
* [gRPC](plugins/grpc.md)
* [TCP](plugins/tcp.md)
* [KV](kv/overview-kv.md)

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
* [Laravel](integration/laravel.md)
* [Symfony](integration/symfony.md)
* [CakePHP](integration/cake.md)
* [ChubbyPHP](integration/chubbyphp.md)
* [CodeIgniter](integration/codeigniter.md)
* [Mezzio](integration/mezzio.md)
* [Phalcon](integration/phalcon.md)
* [Slim](integration/slim.md)
* [Symlex](integration/symlex.md)
* [Ubiquity](integration/ubiquity.md)
* [Zend Expressive](integration/zend.md)

## 🧪 Experimental Features

* [List of the Experimental Features](experimental/experimental.md)

## 🚨 Error codes

* [CRC valication failed](known-issues/stdout-crc.md)
* [Allocate Timeout](known-issues/allocate-timeout.md)

## 📚 Releases

* [v2023.3.9](releases/v2023-3-9.md)
* [v2023.3.8](releases/v2023-3-8.md)
* [v2023.3.7](releases/v2023-3-7.md)
* [v2023.3.6](releases/v2023-3-6.md)
* [v2023.3.5](releases/v2023-3-5.md)
* [v2023.3.4](releases/v2023-3-4.md)
* [v2023.3.3](releases/v2023-3-3.md)
* [v2023.3.2](releases/v2023-3-2.md)
* [v2023.3.1](releases/v2023-3-1.md)
* [v2023.3.0](releases/v2023-3-0.md)
