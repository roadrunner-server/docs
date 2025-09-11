# Temporal worker
Unlike HTTP, Temporal uses a different way to configure a worker. Make sure to require the PHP SDK:

```
$ composer require temporal/sdk
```

The worker file will look as follows:

```php
<?php

declare(strict_types=1);

use Temporal\WorkerFactory;

ini_set('display_errors', 'stderr');
include "vendor/autoload.php";

// factory initiates and runs task queue specific activity and workflow workers
$factory = WorkerFactory::create();

// Worker that listens on a task queue and hosts both workflow and activity implementations.
$worker = $factory->newWorker(
    'taskQueue',
    \Temporal\Worker\WorkerOptions::new()->withMaxConcurrentActivityExecutionSize(10)
);

// Workflows are stateful. So you need a type to create instances.
$worker->registerWorkflowTypes(MyWorkflow::class);

// Activities are stateless and thread safe. So a shared instance is used.
$worker->registerActivityImplementations(new MyActivity());


// start primary loop
$factory->run();
```

Read more about Temporal configuration and usage on the [official website](https://docs.temporal.io/develop/php/core-application#run-a-dev-worker).

## Multi-worker environment
To serve both HTTP and Temporal from the same worker, use the `getMode()` option of `Environment`:

```php
use Spiral\RoadRunner;

$rrEnv = RoadRunner\Environment::fromGlobals();

if ($rrEnv->getMode() === RoadRunner\Environment\Mode::MODE_TEMPORAL) {
    // start Temporal worker
    return;
}

if ($rrEnv->getMode() === RoadRunner\Environment\Mode::MODE_HTTP) {
    // start HTTP worker
    return;
}
```

Or you may override the server command via:
```yaml
temporal:
  address: "127.0.0.1:7233"
  activities:
    num_workers: 10
    command: "php temporal.php"
```
