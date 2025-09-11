# Auto worker scaling [BETA]

## Beta notice
This feature is still in beta, errors are expected. Do not use it in production environments.

## Introduction

This feature became available starting with the RoadRunner `2024.3` release.
Users can now scale their RoadRunner workers automatically, up to an additional 100 workers.

### Supported plugins

- All plugins that support a worker pool are supported.

### Limitations

- This feature is unavailable when running RoadRunner in debug mode (`*.pool.debug=true`).
- This feature does not scale Temporal workflow workers; only activity workers are scaled.

### How it works
RoadRunner uses the `pool.allocate_timeout` option to determine when to start spawning additional workers. If no workers are available to handle the request before the timeout expires, RoadRunner begins dynamically allocating additional workers according to the `spawn_rate`.

### Usage

Below is a configuration example demonstrating how to use this new feature:

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

The new `dynamic_allocator` section has been added to the `*.pool` configuration. It contains the following parameters:
- `max_workers` - the maximum number of workers that can be additionally spawned.
- `spawn_rate` - the number of workers that can be spawned per NoFreeWorkers error (but up to `max_workers`).
- `idle_timeout` - the time after which dynamically allocated workers are considered not needed and will be deallocated.
