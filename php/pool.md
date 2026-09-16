# Worker pool

RoadRunner uses a worker pool to start PHP CLI processes, assign work, and replace stopped workers. The pool also handles worker reset and shutdown.

The worker pool is not used in every RoadRunner plugin but only in the `http`, `gRPC`, `tcp`, `roadrunner-temporal`, `jobs`, and `centrifuge` plugins.

Additionally, the worker pool contains an internal `supervisor` to control the execution TTL of the workers, overall TTL, and execution time limits.

## Worker pool configuration

The v6 plugin beta uses [pool/v2 v2.0.0-beta.1](https://github.com/roadrunner-server/pool/tree/v2.0.0-beta.1). The configuration below describes that version.

{% code title=".rr.yaml" %}

```yaml
  # Workers pool settings.
  pool:
    # Start a fresh worker for each request. Do not pre-allocate workers.
    # In debug mode, num_workers is ignored.
    #
    # Default: false
    debug: false

    # Override server's command
    #
    # Default: empty
    command: "php my-super-app.php"

    # Initial worker count, at most 500. Zero means the number of logical CPUs.
    #
    # Default: 0
    num_workers: 0

    # Maximal count of worker executions. Zero (or nothing) means no limit.
    #
    # Default: 0
    max_jobs: 0

    # [2023.3.10]
    # Request admission limit. Concurrent requests can exceed this value.
    #
    # Default: 0 (no limit)
    max_queue_size: 0

    # Timeout for worker allocation. Zero means 60s.
    #
    # Default: 60s
    allocate_timeout: 60s

    # Wait for active work before stopping workers during reset. Zero means 60s.
    #
    # Default: 60s
    reset_timeout: 60s

    # Timeout for the stream cancellation. Zero means 60s.
    #
    # Default: 60s
    stream_timeout: 60s

    # Wait for active work before stopping workers during shutdown. Zero means 60s.
    #
    # Default: 60s
    destroy_timeout: 60s

    # Dynamic allocator settings. Base and additional workers share a limit of 2048.
    #
    # Default: empty
    dynamic_allocator:
        max_workers: 25
        spawn_rate: 10
        idle_timeout: 10s

    # Supervisor is used to control HTTP workers (previous name was "limit", video: https://www.youtube.com/watch?v=NdrlZhyFqyQ).
    # "Soft" limits will not interrupt current request processing. "Hard"
    # limit on the contrary - interrupts the execution of the request.
    supervisor:
      # How often to check the state of the workers.
      #
      # Default: 5s
      watch_tick: 5s

      # How long a worker can live (soft limit). Zero means no limit.
      #
      # Default: 0s
      ttl: 0s

      # How long a worker can spend in IDLE mode after first use (soft limit). Zero means no limit.
      #
      # Default: 0s
      idle_ttl: 10s

      # Maximal worker memory usage in megabytes (soft limit). Zero means no limit.
      #
      # Default: 0
      max_worker_memory: 128

      # Maximal job lifetime (hard limit). Zero means no limit.
      #
      # Default: 0s
      exec_ttl: 60s
```

{% endcode %}

## Timeouts and admission

`allocate_timeout` limits worker allocation and the wait for a free worker. It does not limit PHP request execution. Set `supervisor.exec_ttl` to limit execution time. Without `exec_ttl`, canceling the caller's context does not stop normal pool execution in PHP.

For a stream, `exec_ttl` applies separately to each read, not to the entire stream. `stream_timeout` applies to stream cancellation.

`reset_timeout` and `destroy_timeout` limit the wait for active work before the pool starts stopping workers. They are not strict limits on the entire operation. A worker stop has its own ten-second grace period.

`max_queue_size` checks the number of active pool `Exec` calls before another call is registered. This includes calls waiting for a worker and calls executing a request. Concurrent calls can pass the check together, so the value is not a strict queue-capacity guarantee. Zero disables the check.

Tips and tricks:

{% hint style="info" %}
See [automatic worker scaling](auto-scaling.md) for allocation batches, capacity limits, and timeout behavior.
{% endhint %}

{% hint style="info" %}
If you need to control the worker's memory, use `supervisor.max_worker_memory` option rather than `pool.max_jobs`.
{% endhint %}

{% hint style="info" %}
Use `pool.debug = true` locally for development purposes. See [additional info](developer.md).
{% endhint %}
