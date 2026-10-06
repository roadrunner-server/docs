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

## Dynamic Workflows

A dynamic workflow handles workflow types that have no named registration in that SDK worker. Register at most one dynamic workflow per worker. Multiple dynamic registrations cause worker initialization to fail.

PHP SDK support is pending in [temporalio/sdk-php#774](https://github.com/temporalio/sdk-php/pull/774). Use an SDK version that implements this support. The SDK must set the `dynamic` field to `true` in the workflow registration metadata. This is a worker registration option.

## Worker Recovery

After an activity worker exits, the pool replaces that worker without an explicit activity-pool reset. A workflow-worker exit still triggers a full reset and clears the sticky workflow cache so Temporal can replay workflow history.

## Graceful Shutdown

RoadRunner stops Temporal task polling before it destroys the PHP worker pools. Set the PHP SDK worker stop timeout to give running task handlers time to finish:

```php
$worker = $factory->newWorker(
    'taskQueue',
    \Temporal\Worker\WorkerOptions::new()
        ->withWorkerStopTimeout(new \DateInterval('PT20S'))
);
```

The plugin passes this timeout to the Go SDK. Set the RoadRunner [shutdown grace period](../app-server/cli.md#stopping-the-server) to allow time for SDK workers and PHP pools to stop. Activity heartbeats can continue while the plugin waits for running work.

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
