# Environment variables

Environment variables allow you to separate configuration data from your application code, making it more maintainable
and portable.

RoadRunner supports the expansion of environment variables using the `${VARIABLE}` or `$VARIABLE` syntax in a
configuration file and CLI commands. You can use this feature to dynamically set values based on the current
environment, such as database connection strings, API keys, and other sensitive information.

You can specify a default value for an environment variable using the `${VARIABLE:-DEFAULT_VALUE}` syntax. For example,
if you want to use a default value of `8080` for the `HTTP_PORT` environment variable if it is not defined or is empty,
you can use the following configuration:

{% code title=".rr.yaml" %}

```yaml
http:
  address: 127.0.0.1:${HTTP_PORT:-8080}
```

{% endcode %}

{% hint style="info" %}
You can find more information on Bash Environment Variable Defaults in
the [Bash Reference Manual](https://www.gnu.org/software/bash/manual/bash.html#Shell-Parameter-Expansion).
{% endhint %}

This allows you to easily customize the configuration based on your specific environment without changing the
configuration file itself.

Here is an example of a `docker-compose.yaml` file that redefines the `HTTP_PORT` for an RR service:

{% code title="docker-compose.yaml" %}

```yaml
version: '3.8'

services:
  app:
    image: xxx
    environment:
      - HTTP_PORT=8081
```

{% endcode %}

## Setting environment variables

You can set environment variables for PHP workers by defining them in the `server.env` section of the RoadRunner
configuration file. These variables will be applied to all workers when they are started by the server.

**Here is an example:**

{% code title=".rr.yaml" %}

```yaml
server:
  command: "php worker.php"
  env:
    APP_RUNTIME: "${APP_RUNTIME:-prod}"
```

{% endcode %}

This example uses the RoadRunner process value of `APP_RUNTIME`. If it is unset or empty, the worker receives `prod`.

Use a mapping for `server.env`, as shown above. Environment defaults are expanded in map values. A sequence of one-key maps does not receive the same expansion.

Values in `server.env` override inherited environment values in the worker processes. They do not change the RoadRunner process environment. Use `server.on_init.env` separately for the [initialization command](../plugins/server.md#server-initialization).

{% hint style="warning" %}
Keys in `server.env` are automatically converted to uppercase.
{% endhint %}

## Dotenv

Use the root `envfile` setting to load a file before environment variable expansion in the main configuration and included files:

{% code title=".rr.yaml" %}

```yaml
version: "3"

envfile: env/.env

logs:
  level: ${RR_LOG_LEVEL:-info}
```

{% endcode %}

With config plugin v6, `envfile` no longer requires experimental mode. Its path is relative to the main configuration file's directory, not the process working directory. In this example, a main configuration at `/var/www/.rr.yaml` loads `/var/www/env/.env`.

The file supplies variables that are not already present in the RoadRunner process environment. It does not replace existing values. For example, an exported `RR_LOG_LEVEL=error` takes precedence over `RR_LOG_LEVEL=info` in the file. A missing or unreadable file stops startup.

The `reset`, `workers`, and `jobs` CLI commands use their own configuration reader. It does not load the root `envfile` setting. Use `--dotenv` for those commands when their RPC address depends on variables from that file:

{% code %}

```bash
./rr serve --dotenv /var/www/config/.env
./rr workers --dotenv /var/www/config/.env
```

{% endcode %}

The `DOTENV_PATH` environment variable also selects a dotenv file and takes precedence over `--dotenv`. Relative CLI dotenv paths use the effective working directory.

## Default environment variables in PHP workers

RoadRunner comes with a set of default environment (ENV) values that facilitate proper communication between the PHP
process and the server. These values are automatically available to workers and can be used to configure and manage
various aspects of the worker's operation.

**Here is a list of the default environment variables provided by RoadRunner:**

| Key            | Description                                                                                                  |
|----------------|--------------------------------------------------------------------------------------------------------------|
| **RR_MODE**    | Identifies the mode the worker should run in (`http`, `temporal`, `grpc`, `jobs`, `tcp`, `centrifuge`, etc.) |
| **RR_RPC**     | Contains RPC connection address when enabled.                                                                |
| **RR_RELAY**   | `pipes` or `tcp://...`, depends on server relay configuration.                                               |
| **RR_VERSION** | RoadRunner version that started the PHP worker. Available since RoadRunner `v2023.1.0`.                      |

These default environment values can be used within your PHP worker to configure various settings and adapt the worker's
behavior according to the specific requirements of your application.

{% hint style="info" %}

See how these variables are used in the [spiral/roadrunner-worker](https://github.com/roadrunner-php/worker/blob/3.x/src/Environment.php) to determine the Environment.

{% endhint %}

## What's Next?

1. [Environment variables](../intro/config.md) - Learn how to use environment variables in your RoadRunner
   configuration.
