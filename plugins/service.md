# Service

RoadRunner Service Plugin provides a simple API to monitor and control processes. It is often used to manage background
processes, daemons, or services that need to run continuously. The Service Plugin allows you to start, stop, and manage
any number of processes, including PHP scripts, binaries, and bash/powershell scripts.

## Manage services using RoadRunner config

The Service Plugin is configured using a `.rr.yaml`.

Here is an example configuration:

{% code title=".rr.yaml" %}

```yaml
version: "3"

service:
  meilisearch:
    service_name_in_log: true
    timeout_stop_sec: 10
    remain_after_exit: true
    restart_sec: 1
    command: "./bin/meilisearch"
    user: "www-data"
  centrifuge:
    service_name_in_log: true
    timeout_stop_sec: 10
    remain_after_exit: true
    restart_sec: 1
    command: "./bin/centrifugo --config=centrifugo.json"
  some_service_1:
    command: "php loop.php"
    process_num: 10
    timeout_stop_sec: 10
    exec_timeout: 0s
    remain_after_exit: true
    service_name_in_log: false
    env:
      foo: "BAR"
    restart_sec: 1
```

{% endcode %}

The `service` section is where you define your services, with each service having its own configuration settings.

### Configuration Settings

The following are the available configuration settings for each service:

| Setting               | Description                                                                                                                                                                                                                                                                                                                    |
|-----------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **command**           | The command to execute. There are no restrictions on commands. It could be a binary, a PHP file, a script, etc.                                                                                                                                                                                                                |
| **process_num**       | The number of processes for the command to fire. The default value is `1`.                                                                                                                                                                                                                                                     |
| **exec_timeout**      | The maximum execution time as a YAML duration, such as `1h`, `1m`, or `1s`. The default is `0s` (unlimited). RPC and PHP use integer seconds. |
| **timeout_stop_sec**  | The stop timeout in integer seconds. The default is `5`; `0` selects the default. |
| **remain_after_exit** | If `true`, automatically restarts the process after any exit code. The time between starts includes execution time and `restart_sec`. |
| restart_sec           | The delay after process exit, in integer seconds. The default is `30`; `0` selects the default. |
| service_name_in_log   | If `true`, adds a `service` log attribute with the service name. The logger name remains `service`. The default is `false`.                                                                                                                                                                                                    |
| env                   | Environment variables to pass to the underlying process from the config.                                                                                                                                                                                                                                                       |
| user                  | Username (not UID) for the Service process. An empty value means to use the RR process user.                                                                                                                                                                                                                                   |

Services will be started when RoadRunner starts and will be stopped when RoadRunner stops.

Service plugin v5 used `service_name_in_log` to change the logger name to `service.NAME`. In v6, update log filters to use the separate `service` attribute. Use production mode or a custom [logger format](../lab/logger.md#custom-format) with `%attrs%` to retain that attribute; raw mode discards it.

## PHP client

The RoadRunner Service Plugin PHP Client Library allows you to manage processes in PHP application using the Service
Plugin. You can use the library to create, start, stop, restart, and manage any number of services.

### Installation

You can install the package via composer:

{% code %}

```bash
composer require spiral/roadrunner-services
```

{% endcode %}

### Usage

To use the library, you need to create an instance of `Spiral\RoadRunner\Services\Manager`:

{% code title="app.php" %}

```php
use Spiral\RoadRunner\Services\Manager;
use Spiral\Goridge\RPC\RPC;

require __DIR__ . '/vendor/autoload.php';

$manager = new Manager(RPC::create('tcp://127.0.0.1:6001'));
```

{% endcode %}

### Create a service

To create a new service, use the `create` method:

{% code title="app.php" %}

```php
use Spiral\RoadRunner\Services\Exception\ServiceException;

try {
    $result = $manager->create(
        name: 'listen-jobs', 
        command: 'php app.php queue:listen',
        processNum: 3,
        execTimeout: 0,
        remainAfterExit: false,
        env: ['APP_ENV' => 'production'],
        restartSec: 30
    );
    
    if (!$result) {
        throw new ServiceException('Service creation failed.');
    }
} catch (ServiceException $e) {
    // handle exception
}
```

{% endcode %}

#### Update service configuration

{% hint style="warning" %}
`service.Update` is a development/unreleased feature. Use a RoadRunner development build with the updated service plugin. The [pinned source build](../intro/install.md) at `b0cccd9` does not include it. The PHP examples require the upcoming client release with `Manager::update()` and the upcoming DTO release with `Update` and `Environment`.

API `v6.0.0-beta.6` and `api-go/v6 v6.0.0-beta.15` are published prereleases of the schema and generated Go types. The service plugin and PHP updates are still unreleased.
{% endhint %}

Use `update()` to change the desired configuration of an existing service. A `true` result confirms acceptance. A name-only call succeeds. Omitted arguments and `null` keep stored values. Explicit `false`, `0`, and `[]` are sent.

{% code title="app.php" %}

```php
$result = $manager->update(
    name: 'listen-jobs',
    processNum: 2,
    execTimeout: 60,
    remainAfterExit: true,
    env: ['APP_ENV' => 'production', 'QUEUE' => 'priority'],
    restartSec: 5,
    serviceNameInLogs: true,
    stopTimeout: 10
);
```

{% endcode %}

| RPC field | PHP argument | Behavior |
| --- | --- | --- |
| `process_num` | `processNum` | Target process count, at least `1`. |
| `exec_timeout` | `execTimeout` | Integer seconds; `0` means unlimited. A running execution keeps its deadline. |
| `remain_after_exit` | `remainAfterExit` | Automatic restart after any exit code. The latest value controls exit handling and queued starts. `false` cancels queued automatic starts. |
| `env` | `env` | Replaces the complete service override map for new executions. `[]` clears overrides. |
| `restart_sec` | `restartSec` | Integer seconds; `0` selects `30`. A queued timer keeps its delay. The next scheduled restart uses the new delay. |
| `service_name_in_logs` | `serviceNameInLogs` | Adds the separate `service` log attribute for new executions. Current output keeps its logger. YAML uses `service_name_in_log`. |
| `timeout_stop_sec` | `stopTimeout` | Integer seconds; `0` selects `5`. New executions use this stop timeout. Current executions keep theirs. |

Updates keep current PIDs. When automatic restart is enabled, the next exit or queued restart adjusts the process count. Excess processes finish without replacement. An increase starts the missing processes at the next restart opportunity.

Processes inherit the RoadRunner environment. Override keys become uppercase. Values expand from the RoadRunner environment. Empty strings and the string `'0'` are preserved.

For example, send explicit values to clear settings and disable automatic restart:

{% code title="app.php" %}

```php
$result = $manager->update(
    name: 'listen-jobs',
    execTimeout: 0,
    remainAfterExit: false,
    env: [],
    serviceNameInLogs: false
);
```

{% endcode %}

A later explicit `Restart` or `Reset` uses unlimited execution time, cleared environment overrides, and no separate service log attribute. Automatic restart stays disabled.

Server validation rejects the whole invalid request. An unknown service returns an RPC error. `Manager` converts these RPC errors to `Spiral\RoadRunner\Services\Exception\ServiceException`.

Updates stay in memory. RoadRunner process startup reloads file configuration. Explicit `Restart` and `Reset` use the latest settings and count. An inactive service with no queued starts stays inactive until `Restart` or `Reset`.

#### Checking Service Status

To check the status of a service, use the `statuses` method:

{% code title="app.php" %}

```php
use Spiral\RoadRunner\Services\Exception\ServiceException;

try {
    $status = $manager->statuses(name: 'listen-jobs');
    
    // Will return an array with statuses of every run process
} catch (ServiceException $e) {
    // handle exception
}
```

{% endcode %}

#### Restarting a Service

To restart a service, use the `restart` method:

In v6, restart calls stop for each old process before starting replacements. It is not a rolling or atomic restart. Plan for an interval with no running process in that service.

{% hint style="warning" %}
In `v6.0.0-beta.8`, a service with `remain_after_exit: true` can start replacements before old processes finish if an automatic restart occurred earlier. Do not rely on `service.Restart` for exclusive process replacement in this case.
{% endhint %}

{% code title="app.php" %}

```php
use Spiral\RoadRunner\Services\Exception\ServiceException;

try {
    $result = $manager->restart(name: 'listen-jobs');
    
    if (!$result) {
        throw new ServiceException('Service restart failed.');
    }
} catch (ServiceException $e) {
    // handle exception
}
```

{% endcode %}

If a replacement fails to start, RoadRunner requests a stop for replacements that already started and removes the service from its registry. It does not restore the old processes. Fix the reported startup error. Then call `create` with the required service settings. Another `restart` call cannot recover a service that is no longer registered.

#### Terminating a Service

To terminate a service, use the `terminate` method:

{% code title="app.php" %}

```php
use Spiral\RoadRunner\Services\Exception\ServiceException;

try {
    $result = $manager->terminate(name: 'listen-jobs');
    
    if (!$result) {
        throw new ServiceException('Service termination failed.');
    }
} catch (ServiceException $e) {
    // handle exception
}
```

{% endcode %}

{% hint style="info" %}
When you're terminating the service, RR sends the `SIGINT` signal to the underlying process(ses).
{% endhint %}

#### Listing All Services

To get a list of all services, use the `list` method:

{% code title="app.php" %}

```php
use Spiral\RoadRunner\Services\Exception\ServiceException;

try {
    $services = $manager->list();
    
    // Will return an array with services names
    // ['listen-jobs', 'websocket-connection'] 
} catch (ServiceException $e) {
    // handle exception
}
```

{% endcode %}

## API

### Protobuf API

To make it easy to use the Service proto API in PHP, we provide
a [GitHub repository](https://github.com/roadrunner-php/roadrunner-api-dto), that contains all the generated
PHP DTO classes proto files, making it easy to work with these files in your PHP application.

- [Service protobuf API](https://github.com/roadrunner-server/api/blob/v6.0.0-beta.6/roadrunner/api/service/v1/service.proto)

### Update with Goridge

Use Goridge RPC with `ProtobufCodec` for `service.Update`. It takes `Update` and returns `Response`. The development build and upcoming PHP DTO release described [above](#update-service-configuration) are required.

{% code title="app.php" %}

```php
use RoadRunner\Service\DTO\V1\Environment;
use RoadRunner\Service\DTO\V1\Response;
use RoadRunner\Service\DTO\V1\Update;
use Spiral\Goridge\RPC\Codec\ProtobufCodec;
use Spiral\Goridge\RPC\RPC;

require __DIR__ . '/vendor/autoload.php';

$rpc = RPC::create('tcp://127.0.0.1:6001')->withCodec(new ProtobufCodec());
$update = (new Update())
    ->setName('listen-jobs')
    ->setEnv(new Environment());

$response = $rpc->call('service.Update', $update, Response::class);
$result = $response->getOk();
```

{% endcode %}

Call setters only for fields to change. Omitting `setEnv()` keeps the override map. A present empty `Environment` clears it. To replace the map, use `setEnv((new Environment())->setValues(['APP_ENV' => 'production']))`.
