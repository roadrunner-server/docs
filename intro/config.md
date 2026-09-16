# RoadRunner — Configuration

RoadRunner supports both **YAML** and **JSON** configuration formats. The examples in our documentation use YAML, but
you can use JSON as well.

{% hint style="info" %}
To convert a YAML configuration file to JSON, you can use an online tool such
as <https://onlineyamltools.com/convert-yaml-to-json>.
{% endhint %}

## Configuration reference

Use the configuration reference for the RoadRunner revision you build. The following file matches the snapshot used by the [installation guide](install.md):

- [**.rr.yaml**](https://github.com/roadrunner-server/roadrunner/blob/b0cccd917f001b6584eafdc04ad6ba69a97cbb69/.rr.yaml)

{% hint style="warning" %}
We use dots as level separators, e.g.: `http.pool`, you can't use dots in section names, queue names, etc. You can find out more about it [here](https://github.com/roadrunner-server/roadrunner/issues/1529).
{% endhint %}

## Unix Socket Attributes

{% hint style="warning" %}
RoadRunner v3 supports these options through its v6 plugins. RoadRunner `v2025.1.15` does not include them. See [New Features](v3-migration.md#new-features).
{% endhint %}

Configure each filesystem Unix listener separately. Omit its options object to keep the existing socket defaults.

The configuration provider can discard an empty options object (`{}`). It then behaves like omission, including on TCP addresses or the `pipes` worker relay.

| Options object | Listener address |
| --- | --- |
| `http.unix_socket` | `http.address`, including H2C |
| `http.fcgi.unix_socket` | `http.fcgi.address` |
| `rpc.unix_socket` | `rpc.listen` |
| `grpc.unix_socket` | `grpc.listen` |
| `tcp.servers.<name>.unix_socket` | `tcp.servers.<name>.addr` |
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

See [Nginx group access](../app-server/nginx-with-rr.md#development-unix-socket) for a FastCGI example.

## Configuration file

The RoadRunner looks for a configuration file named `.rr.yaml` in the same directory as the server binary.

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

## Support for the nested configurations: `[>=2024.2.0]`, [issue](https://github.com/roadrunner-server/roadrunner/issues/935)

Using the following syntax, you may include other configuration files into the main one:

{% code title=".rr.yaml" %}

```yaml
version: "3"

include:
  - .rr.include1-sub1.yaml
  - ../foo/bar/rr.include1-sub2.yaml

reload:
  interval: 1s
  patterns: [".php"]
```

{% endcode %}

{% hint style="warning" %}

Starting from version `v2024.2.0`, the `includes` configuration option no longer has restrictions on where the included config can be placed. However, please note that the path for the included configurations is calculated based on the working directory of the RoadRunner process.

{% endhint %}

Includes override the main configuration file. For example, if you have the following nested configuration:

{% code title=".rr.include1-sub1.yaml" %}

```yaml
version: "3"

server:
  command: "php php_test_files/psr-worker-bench.php"
  relay: pipes

http:
  address: 127.0.0.1:15389
  middleware:
    - "sendfile"
  pool:
    allocate_timeout: 10s
    num_workers: 2
```

{% endcode %}

It will override the `server` and `http` sections of the main configuration file.
You may use env variables in the included configuration files, but you can't use overrides for the nested configuration. For example:

{% hint style="info" %}

The next 'include' will override values set by the previous 'include'. Values in the root `.rr.yaml` will also be overwritten by the includes. Feel free to send us feedback on this feature.

{% endhint %}

{% code title=".rr.include1-sub1.yaml" %}

```yaml
version: "3"

server:
  command: "${PHP_COMMAND:-php_test_files/psr-worker-bench.php}"
  relay: pipes
```

{% endcode %}

You may use any number of the included configuration files via CLI command, in quotas and separated by whitespace. For example:

{% code %}

```bash
./rr serve -e -c .rr.yaml -o include=".rr.yaml .rr2.yaml"
```

{% endcode %}

## What's Next?

1. [Server Commands](../app-server/cli.md) - learn how to start the server.
2. [PHP Workers — Environment variables](../php/environment.md) - learn how to configure PHP workers environment.
3. [Config plugin](../plugins/config.md) - learn more about the Config plugin.
