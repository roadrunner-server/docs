# Dynamic Worker Scaling

## Introduction

Manual scaling has been available since RoadRunner `2023.3`. Use `Spiral\RoadRunner\WorkerPool` to add or remove workers through [Goridge RPC](rpc.md). The v6 plugin beta retains these PHP calls.

### Limitations

- This feature is not available when running RoadRunner in debug mode (`pool.debug=true`).
- With `pool/v2 v2.0.0-beta.1`, a pool can hold at most 2048 workers. The initial `num_workers` value cannot exceed 500.
- A removal selects a free worker. It does not interrupt an active request or remove the last worker.

### Usage

Add or remove a worker from the HTTP pool:

{% code title="worker.php" %}

```php
use Spiral\RoadRunner\WorkerPool;
use Spiral\Goridge\RPC\RPC;

$rpc = RPC::create('tcp://127.0.0.1:6001');
$pool = new WorkerPool($rpc);

// Add a worker to the pool.
$pool->addWorker('http');

// Remove a worker from the pool.
$pool->removeWorker('http');
```

{% endcode %}

### List of supported plugins

- `http`, `grpc`, `temporal`, `centrifuge`, `tcp`, `jobs`.

For automatic allocation, see [automatic worker scaling](auto-scaling.md).
