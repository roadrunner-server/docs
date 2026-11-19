# Redis Jobs driver

Use `driver: redis` to publish and consume jobs with Redis 8.0 or later. RoadRunner stores ready jobs in a [Redis Stream](https://redis.io/docs/latest/develop/data-types/streams/). Instances that use the same Redis database and queue name share the work.

## Configuration

{% code title=".rr.yaml" %}

```yaml
version: '3'
rpc:
  listen: tcp://127.0.0.1:6001
server:
  command: php consumer.php
redis:
  addrs: ["127.0.0.1:6379"]
  db: 1
jobs:
  pipelines:
    mail:
      driver: redis
      config:
        queue: outbound
        priority: 10
        prefetch: 10
        visibility_timeout: 5m
  consume: [mail]
```

{% endcode %}

The top-level `redis` section supplies the [Redis connection settings](../kv/redis.md), including authentication, TLS, and Sentinel. Jobs opens a connection for each pipeline. It does not require a KV storage.

| Option | Default | Meaning |
| --- | --- | --- |
| `queue` | Pipeline name | Name of the Redis queue. Braces are reserved for Redis Cluster key routing. |
| `priority` | `10` | Priority for a job with priority zero. RoadRunner dispatches lower numbers first among jobs in its local priority queue. Redis reads new stream entries in insertion order. |
| `prefetch` | `10` | Maximum number of jobs this pipeline can hold while they wait for or run in a worker. |
| `visibility_timeout` | `5m` | Minimum idle time before a consumer can reclaim a pending job. Allow time for both the local queue and the worker. |

Omit a pipeline from `jobs.consume` to publish before consumption starts. Call `jobs.Resume` to start reading. `jobs.Pause` stops new reads. Jobs already in the local queue can finish. During shutdown, active workers can acknowledge or requeue deliveries until the shutdown deadline. You can also declare a pipeline with `jobs.Declare`.

{% hint style="warning" %}
If a KV storage uses `kv.Clear`, give it a separate Redis database. The Redis KV driver runs [`FLUSHDB`](https://redis.io/docs/latest/commands/flushdb/), which also deletes queue keys in that database. [Redis Cluster supports only database 0](https://redis.io/docs/latest/commands/select/). Use separate deployments if a KV storage on Cluster needs `Clear`.
{% endhint %}

## Delivery and retries

With manual acknowledgment, Redis keeps a delivery pending until PHP acknowledges it or requests a terminal rejection. If RoadRunner stops before acknowledgment, a consumer can reclaim the delivery after `visibility_timeout`, even if no new jobs arrive. The job keeps its ID and headers. A late response from a previous delivery cannot settle the current one.

If PHP finishes an operation but RoadRunner stops before the acknowledgment reaches Redis, PHP can receive the job again. Use the job ID to make operations safe to repeat. Survival of a Redis restart or failover depends on the [Redis persistence and replication settings](https://redis.io/docs/latest/develop/data-types/streams/#persistence-replication-and-message-safety). If Redis loses the consumer group, RoadRunner recreates it and reads from the start of the retained stream.

A worker transport error leaves a manual delivery pending for recovery. A terminal `nack` discards it. An explicit requeue can update headers and set a delay. Redis stores delayed pushes and retries in a sorted set. A consumer moves due jobs to the stream. If Redis has too little memory for this step, the consumer continues to read ready jobs.

With `auto_ack` enabled, RoadRunner acknowledges the job when it prepares the payload for PHP. A job waiting in the local queue remains pending in Redis. After auto-ack, an explicit retry returns an error and publishes no replacement. Use manual acknowledgment when the worker needs retries.
