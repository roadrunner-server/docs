# Server

RoadRunner server plugin, is responsible for starting worker pools for plugins that use workers, such as `http`, `tcp`,
`jobs`, `centrifuge`, `temporal`, and `grpc`. The worker pools inherit all of RoadRunner's features, such as
supervising, state machine, and command handling.

## Configuration

The `server` section contains various options for configuring the plugin.

**Here is an example of a configuration:**

{% code title=".rr.yaml" %}

```yaml
server:
  on_init:
    # Command to execute before the main server's command
    #
    # This option is required if using on_init
    command: "any php or script here"

    # Username (not UID) of the user from whom the on_init command is executed. An empty value means to use the RR process user.
    #
    # Default: ""
    user: ""

    # Timeout after the command starts. Default: 60s. Include a unit.
    exec_timeout: 20s

    # Environment variables for the initialization command.
    #
    # Default: <empty map>
    env:
      SOME_KEY: "SOME_VALUE"
      SOME_KEY2: "SOME_VALUE2"

    # Exit RR if the `on_init` command fails or exceeds the `exec_timeout`.
    exit_on_error: false

  # Worker starting command, with any required arguments.
  #
  # This option is required.
  command: "php psr-worker.php"

  # Username (not UID) for the worker processes. An empty value means to use the RR process user.
  #
  # Default: ""
  user: ""

  # Environment variables for the worker processes.
  #
  # Default: <empty map>
  env:
    SOME_KEY: "SOME_VALUE"
    SOME_KEY2: "SOME_VALUE2"

  relay: pipes
```

{% endcode %}

{% hint style="info" %}
Worker relay can be: `pipes`, TCP (e.g.: `tcp://127.0.0.1:6002`), or socket (e.g.: `unix:///var/run/rr.sock`). But in
most cases, you should use the default `pipes` relay, it is the fastest communication transport.
It uses an inter-process communication mechanism that allows for fast and efficient communication between
processes. It does not require any network connections or external libraries, making it a lightweight and fast option.
{% endhint %}

{% hint style="info" %}
Use `on_init.user` option to execute the on_init command under a different user.
{% endhint %}

### Server initialization

The `on_init` section is used for application initialization or warming up before starting workers. It allows you to set
a command script that will be executed before starting the workers. You can also set environment variables to pass to
this script.

With server plugin v6, `on_init.env` values override inherited process environment values. In v5, inherited values took precedence for this command. Check for conflicting variable names when upgrading. `server.env` does not configure the initialization command.

The `on_init.exec_timeout` interval starts after the command starts successfully. Use a duration with a unit, such as `20s`.

{% hint style="info" %}
By default, RoadRunner logs an `on_init` command error and continues startup. Set `on_init.exit_on_error: true` to stop RoadRunner if the command fails or exceeds `exec_timeout`.
{% endhint %}

### Worker starting command

The `server.command` option is required and is used to start the worker pool for each configured section in the config.

{% hint style="info" %}
This option can be overridden by plugins with a pool section, such as the `http.pool.command`, or in general `<plugin>.pool.command`.
{% endhint %}

In v6, a scalar command or a one-element sequence is split at whitespace. Repeated spaces and tabs do not create empty arguments. This is not shell parsing: quotes inside a scalar do not keep words in one argument. Use a sequence with one element for the executable and one for each argument when an argument contains spaces:

{% code title=".rr.yaml" %}

```yaml
server:
  command: ["php", "worker.php", "--label", "my worker"]
```

{% endcode %}

The same argument rules apply to `server.on_init.command` and pool command overrides.

### Worker User

Set `server.user` to an account name, not a numeric UID. An empty value keeps the RoadRunner process user. The server plugin resolves the selected account's UID and GID during initialization. `server.group` does not override that GID.

In v6, a failed account lookup or an invalid numeric UID or GID stops initialization. Correct the account name before restarting RoadRunner. A nonempty `server.user` is not supported on Windows and also stops initialization; remove that setting on Windows.

{% hint style="warning" %}
On Unix, RoadRunner needs permission to change process credentials when `server.user` selects a different account.
{% endhint %}

Use `server.env` to set [worker environment variables](../php/environment.md#setting-environment-variables).

## PHP Client

There is a package that simplifies the process of integrating the RoadRunner worker pool with a PHP application. The
package contains a common codebase for all RoadRunner workers, making it easy to integrate and communicate with workers
created by RoadRunner.

When RoadRunner creates workers for any plugin that uses workers, it runs a PHP script and starts communicating with it
using a relay. The relay can be `pipes`, `TCP`, or a `socket`. The PHP client simplifies this process by providing a
convenient interface for sending and receiving payloads to and from the worker.

**Here is an example of simple PHP worker:**

{% code title="worker.php" %}

```php
<?php

require __DIR__ . '/vendor/autoload.php';

// Create a new Worker from global environment
$worker = \Spiral\RoadRunner\Worker::create();

while ($data = $worker->waitPayload()) {
    // Received Payload
    var_dump($data);

    // Respond Answer
    $worker->respond(new \Spiral\RoadRunner\Payload('DONE'));
}
```

{% endcode %}

The worker waits for incoming payloads, for example `HTTP request`, `TCP request`, `Queue task` or`Centrifuge message`,
processes them, and sends a response back to the server. Once a payload is received, the worker processes it and sends a
response using the `respond` method.

## What's Next?

1. [PHP Workers](../php/worker.md) - Read more about PHP workers.
