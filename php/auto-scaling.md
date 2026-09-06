# Automatic worker scaling (beta)

## Beta notice

Automatic scaling is in beta. Do not use it in production environments. This page describes [pool/v2 v2.0.0-beta.1](https://github.com/roadrunner-server/pool/tree/v2.0.0-beta.1), which is used by the v6 plugin beta.

## Introduction

Automatic scaling has been available since RoadRunner `2024.3`. It adds workers when the pool cannot supply a free worker before the allocation timeout. It removes extra workers when allocation pressure stops.

### Supported plugins

- All plugins that support a worker pool are supported.

### Limitations

- This feature is unavailable when running RoadRunner in debug mode (`*.pool.debug=true`).
- This feature does not scale Temporal workflow workers; only activity workers are scaled.
- The initial `num_workers` value cannot exceed 500. The combined base and additional worker count cannot exceed 2048.

### How it works

If no worker becomes free within `pool.allocate_timeout`, RoadRunner attempts to add up to `spawn_rate` workers. It does not exceed `max_workers` additional workers or the combined pool limit.

Only one allocation batch can run at a time. Concurrent triggers do not each start a batch. The next batch can start after a one-second cooldown once the previous batch finishes.

{% hint style="warning" %}
In beta.1, allocation does not retry the request that triggered it. That request can still fail with `NoFreeWorkers` after new workers are added. Applications must handle this failure.
{% endhint %}

After an `idle_timeout` interval without allocation triggers, the allocator removes up to `spawn_rate` extra workers per tick. Recent allocation triggers postpone removal, including triggers rejected by the cooldown. This is not a separate idle timer for each worker.

A removal attempt waits up to 500 ms for a free worker. If none becomes free, the allocator stops that removal batch and tries again on a later tick. It does not interrupt a busy worker to scale down. Stopping a selected worker can take longer than the 500 ms wait.

### Usage

Configure the allocator in the plugin's `pool` section:

{% code title=".rr.yaml" %}

```yaml
version: '3'

rpc:
  listen: tcp://127.0.0.1:6002

server:
  command: "php worker.php"

http:
  address: 127.0.0.1:10085
  pool:
    num_workers: 2
    allocate_timeout: 60s
    destroy_timeout: 1s
    dynamic_allocator:
        max_workers: 25
        spawn_rate: 10
        idle_timeout: 10s

logs:
  mode: development
  level: debug
```

{% endcode %}

### Configuration

| Option | Meaning | Default and limit |
| --- | --- | --- |
| `max_workers` | Maximum additional worker count, not the total pool size. | Default: 10. Reduced if necessary so `num_workers + max_workers` does not exceed 2048. |
| `spawn_rate` | Maximum number of workers added per allocation batch or removed per idle tick. | Default: 5. Maximum: 100. |
| `idle_timeout` | Interval for idle checks and minimum time without allocation triggers before removal. | Default: `1m`. Values below `1s` use the default. |
