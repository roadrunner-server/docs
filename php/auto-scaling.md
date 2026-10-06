# Automatic worker scaling

This page describes automatic scaling in the upcoming RoadRunner v3 release, using [pool/v2](https://github.com/roadrunner-server/pool).

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

After a batch adds workers, the request retries worker acquisition. A concurrent request that cannot start a batch because of the cooldown also retries. Each retry waits up to `allocate_timeout`, subject to the caller's context deadline. The total wait can include the initial timeout, worker startup, and the retry timeout.

If a batch adds no workers because the pool is at its limit or worker startup fails, the request returns `NoFreeWorkers`. A batch that adds at least one worker still permits the retry, even if a later worker fails to start. Choose `allocate_timeout` to allow both worker startup and waiting for a free worker.

Idle checks start with the first automatic allocation attempt. After an `idle_timeout` interval without allocation triggers, the allocator removes up to `spawn_rate` extra workers per tick. Recent allocation triggers postpone removal, including triggers rejected by the cooldown. This is not a separate idle timer for each worker.

A removal batch is limited to the number of workers that are free at the start of the check. Each removal attempt waits up to 500 ms for a free worker. If none becomes free, the allocator stops that batch. It does not interrupt a busy worker to scale down. Stopping a selected worker can take longer than the 500 ms wait.

The allocator counts all workers above the base `num_workers` value. This includes workers added through [manual scaling](manual-scaling.md). Worker resets preserve the current pool size; extra workers remain subject to idle removal.

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
