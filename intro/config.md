# RoadRunner Configuration

RoadRunner supports both **YAML** and **JSON** configuration formats. The examples in our documentation use YAML, but
you can use JSON as well.

{% hint style="info" %}
To convert a YAML configuration file to JSON, you can use an online tool such
as <https://onlineyamltools.com/convert-yaml-to-json>.
{% endhint %}

## Configuration reference

Use the configuration reference for the RoadRunner revision you build:

- [**.rr.yaml for v3.0.0**](https://github.com/roadrunner-server/roadrunner/blob/v3.0.0/.rr.yaml)

{% hint style="warning" %}
We use dots as level separators, e.g.: `http.pool`, you can't use dots in section names, queue names, etc. You can find out more about it [here](https://github.com/roadrunner-server/roadrunner/issues/1529).
{% endhint %}

## Unix Socket Attributes

{% hint style="info" %}
RoadRunner v3 supports these options through its v6 plugins. See the [v3 release notes](../releases/v3-0-0.md).
{% endhint %}

Configure each filesystem Unix listener separately. Omit its options object to keep the existing socket defaults.

The configuration provider can discard an empty options object (`{}`). It then behaves like omission, including on TCP addresses or the `pipes` worker relay.

| Options object | Listener address |
| --- | --- |
| `http.unix_socket` | `http.address`, including H2C |
| `http.fcgi.unix_socket` | `http.fcgi.address` |
| `rpc.unix_socket` | `rpc.listen` |
| `grpc.unix_socket` | `grpc.listen` |
| `centrifuge.proxy_socket` | `centrifuge.proxy_address`, the incoming proxy listener |
| `fileserver.unix_socket` | `fileserver.address` |
| `server.relay_socket` | `server.relay`, the worker communication listener |

All objects use the same optional fields:

| Field | Meaning |
| --- | --- |
| `mode` | Quoted octal string from `"0000"` through `"0777"`. Omitted or empty means no mode change. |
| `uid` | Numeric socket owner ID. Omit it to keep the default owner. |
| `gid` | Numeric socket group ID. Omit it to keep the default group. |

Use numeric UID and GID values from `0` through `4294967294` that fit the platform's Go `int` type. Zero is an explicit ID, not an omitted value. Account names are not supported.

The configuration provider converts values before socket validation. Viper can convert booleans and fractional numbers to integer IDs. An unset or empty environment variable can become ID `0`, rather than cause an error. Set ID variables explicitly, or omit the fields to keep ownership unchanged.

These options require a filesystem `unix://` address on Linux, macOS, or FreeBSD. They do not apply to TCP addresses such as `0.0.0.0:8000`, the `pipes` worker relay, Windows, or Linux abstract sockets. They do not change RoadRunner or PHP worker credentials, the process umask, or application file permissions.

The plugin validates the decoded socket options during initialization, before it opens that listener. Invalid modes, out-of-range IDs, and incompatible listener addresses fail initialization. This validation does not check filesystem permissions.

Create the parent directory before startup. RoadRunner needs permission to create the socket there. Clients need search (`x`) permission on every parent directory. The process also needs permission to apply the requested ownership changes. An unprivileged socket owner can change the socket group only to a group to which its process belongs.

Ownership is applied before mode, after the socket starts listening. Clients can connect before these operations finish. The settings specify final attributes, not access control during startup. Existing directory permissions and umask must restrict initial access. Parent directories must prevent untrusted path replacement. If an attribute change fails, RoadRunner closes that listener and reports the error.

See [Nginx group access](../app-server/nginx-with-rr.md#unix-socket) for a FastCGI example.

## Configuration file

RoadRunner looks for `.rr.yaml` in the current working directory.

If your configuration file and other application files are located in a different directory than the binary, you can use
the `-w` option to specify the working directory.

{% code %}

```bash
./rr serve -w /path/to/project
```

{% endcode %}

You can also use the `-c` option to specify the path to the configuration file if you don't want to specify the working
directory.

{% code %}

```bash
./rr serve -c /path/to/project/.rr-dev.yaml
```

{% endcode %}

Or you can combine the `-c` and `-w` options to specify both the configuration file and the working directory:

{% code %}

```bash
./rr serve -c .rr-dev.yaml -w /path/to/project
```

{% endcode %}

Without `-w`, RoadRunner changes its working directory to the configuration file's directory. With `-w`, it uses that directory and resolves a relative `-c` path from there.

{% hint style="info" %}

Read more about starting the server in the [**Server Commands**](../app-server/cli.md) section.

{% endhint %}

### CLI command and arguments

You can also use environment variables in CLI commands to customize the behavior of your RR server. This is especially
useful when you need to pass configuration values that are environment-specific or sensitive, such as secrets or API
keys.

{% code %}

```bash
set -a
source /var/www/config/.env
set +a

exec /var/www/rr \
  -c /var/www/.rr.yaml \
  -w /var/www \
  -o http.pool.num_workers=${RR_NUM_WORKERS:-8} \
  -o http.pool.max_jobs=${RR_MAX_JOBS:-16} \
  -o http.pool.supervisor.max_worker_memory=${RR_MAX_WORKER_MEMORY:-512} \
  serve
```

{% endcode %}

The `set -a` enables automatic exporting of variables. Any variables that are defined in `/var/www/config/.env` will be
automatically exported to the environment, making them available to any child processes that are executed from the
current shell. The final `set +a` command disables automatic exporting of variables, ensuring that only the variables
that were defined in `/var/www/config/.env` are exported, and preventing any unintended variables from leaking into the
environment.

In this example, the following options are used:

| Option                   | Description                                                                                                                          |
|--------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| **-c**                   | Specifies the configuration file.                                                                                                    |
| **-w**                   | Specifies the working directory.                                                                                                     |
| **-o**                   | Overwrites specific configuration options.                                                                                           |
| **/var/www/config/.env** | File that contains the required environment variables.                                                                               |
| **${RR_NUM_WORKERS:-8}** | Sets the number of workers to `RR_NUM_WORKERS` from the `.env` file or uses the default value of `8` if the variable is not present. |

## Include configuration files

List additional configuration files under `include`. RoadRunner reads the list in order after it reads the main file and applies CLI overrides. Includes do not require experimental mode.

{% code title=".rr.yaml" %}

```yaml
version: "3"

include:
  - config/http.yaml
  - config/local.yaml
```

{% endcode %}

Paths are relative to the RoadRunner process working directory. Each included file must contain the same string `version` as the main file. A missing file, invalid YAML, or a different version stops initialization.

For example, put the HTTP settings in this file:

{% code title="config/http.yaml" %}

```yaml
version: "3"

server:
  command: "php worker.php"
  relay: pipes

http:
  address: 127.0.0.1:8080
  pool:
    num_workers: 2
```

{% endcode %}

An included value overrides a matching value from the main file or an earlier include. Values found only in the main file remain available. Included sections can also override values supplied with `-o`. Use complete sections when replacing an earlier include. Lists are replaced, not appended. Includes have no delete operation and do not process another `include` list recursively.

Environment expansion also applies to included files:

{% code title="config/local.yaml" %}

```yaml
version: "3"

logs:
  level: "${RR_LOG_LEVEL:-debug}"
```

{% endcode %}

To set the include list from the CLI, pass a quoted, space-separated list:

{% code %}

```bash
./rr serve -c .rr.yaml -o 'include=config/http.yaml config/local.yaml'
```

{% endcode %}

### Local environment values

Every include is required. The `include` setting accepts file paths, not objects with `file` and `optional` fields. For developer-specific values, use [environment defaults and `envfile`](../php/environment.md#dotenv). An `envfile` must also exist when configured.

Use `version: "3"` in the main file and all includes. The configuration version is separate from the RoadRunner release number. See the [Config plugin](../plugins/config.md#configuration-file-structure).

## What's Next?

1. [Server Commands](../app-server/cli.md) - learn how to start the server.
2. [PHP worker environment variables](../php/environment.md) - learn how to configure the worker environment.
3. [Config plugin](../plugins/config.md) - learn more about the Config plugin.
