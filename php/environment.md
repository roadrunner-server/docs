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
     APP_RUNTIME: prod
```

{% endcode %}

In this example, when RoadRunner starts a PHP worker, it will set the `APP_RUNTIME` environment variable to `prod`.

{% hint style="warning" %}
All environment variable keys will be automatically converted to uppercase.
{% endhint %}

## Dotenv

RoadRunner supports reading environment variables from `.env` files, which are typically used to store sensitive or
environment-specific variables outside your codebase.

To read environment variables from an `.env` file, you can use the `--dotenv` CLI option when starting RoadRunner.

{% code %}

```bash
./rr serve --dotenv /var/www/config/.env
```

{% endcode %}

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
| **RR_VERSION** | RoadRunner version that started the PHP worker (minimum `2023.1.0`)                                          |

These default environment values can be used within your PHP worker to configure various settings and adapt the worker's
behavior according to the specific requirements of your application.

{% hint style="info" %}

See how these variables are used in the [spiral/roadrunner-worker](https://github.com/roadrunner-php/worker/blob/3.x/src/Environment.php) to determine the Environment.

{% endhint %}

## What's Next?

1. [Environment variables](../intro/config.md) - Learn how to use environment variables in your RoadRunner
   configuration.
