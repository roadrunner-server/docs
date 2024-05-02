# Workers pool

RoadRunner uses a worker pool to manage the PHP workers (PHP CLI processes). Internally, the worker pool consists of a Workers Watcher, which is used to control the workers—releasing, allocating, preventing zombie processes, resetting, destroying, and an internal stack responsible for manipulating (popping and pushing) already allocated workers.

The worker pool is not used in every RoadRunner plugin but only in the `http`, `gRPC`, `tcp`, `roadrunner-temporal`, `jobs`, and `centrifuge` plugins.

Additionally, the worker pool contains an internal `supervisor` to control the execution TTL of the workers, overall TTL, and limit execution time.

## Workers pool configuration:

{% code title=".rr.yaml" %}

```yaml
  # Workers pool settings.
  pool:
    # Debug mode for the pool. In this mode, pool will not pre-allocate the worker. Worker (only 1, num_workers ignored) will be allocated right after the request arrived.
    #
    # Default: false
    debug: false

    # Override server's command
    #
    # Default: empty
    command: "php my-super-app.php"

    # How many worker processes will be started. Zero (or nothing) means the number of logical CPUs.
    #
    # Default: 0
    num_workers: 0

    # Maximal count of worker executions. Zero (or nothing) means no limit.
    #
    # Default: 0
    max_jobs: 0

    # [2023.3.10] 
    # Maximum size of the internal requests queue. After reaching the limit, all additional requests would be rejected with error.
    #
    # Default: 0 (no limit)
    max_queue_size: 0

    # Timeout for worker allocation. Zero means 60s.
    #
    # Default: 60s
    allocate_timeout: 60s

    # Timeout for the reset timeout. Zero means 60s.
    #
    # Default: 60s
    reset_timeout: 60s

    # Timeout for the stream cancellation. Zero means 60s.
    #
    # Default: 60s
    stream_timeout: 60s

    # Timeout for worker destroying before process killing. Zero means 60s.
    #
    # Default: 60s
    destroy_timeout: 60s

    # Supervisor is used to control http workers (previous name was "limit", video: https://www.youtube.com/watch?v=NdrlZhyFqyQ).
    # "Soft" limits will not interrupt current request processing. "Hard"
    # limit on the contrary - interrupts the execution of the request.
    supervisor:
      # How often to check the state of the workers.
      #
      # Default: 1s
      watch_tick: 1s

      # How long worker can live (soft limit). Zero means no limit.
      #
      # Default: 0s
      ttl: 0s

      # How long worker can spend in IDLE mode after first using (soft limit). Zero means no limit.
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

Tips and tricks:

{% hint style="info" %}
The worker pool has an internal queue that holds requests waiting for execution, which can be limited with the `pool.max_queue_size` option.
{% endhint %}

{% hint style="info" %}
The Workers pool can be dynamically scaled: [link](scaling.md)
{% endhint %}

{% hint style="info" %}
If you need to control the worker's memory, use `supervisor.max_worker_memory` option rather than `pool.max_jobs`.
{% endhint %}

{% hint style="info" %}
Use `pool.debug = true` locally, for the development purposes, [additional info](developer.md)
{% endhint %}
