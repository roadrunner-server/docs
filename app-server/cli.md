# CLI Commands

RoadRunner offers a convenient CLI to start and manage the server.

| Command | Purpose |
| --- | --- |
| `rr serve` | Start RoadRunner. |
| `rr stop` | Signal the process recorded in `.pid`. |
| `rr reset [plugin...]` | Reset worker pools through RPC. |
| `rr workers [plugin...]` | Show worker state through RPC. |
| `rr jobs --list` | List Jobs pipelines through RPC. |
| `rr jobs --pause`, `--resume`, or `--destroy` | Manage Jobs pipelines through RPC. |

Use `rr --help` or `rr <command> --help` to list the options in your binary.

## Version

To display the version of RoadRunner, you can use the `-v` or `--version` option:

{% code %}

```bash
./rr -v
```

{% endcode %}

This command will display the version of RoadRunner, as well as information about the build time, operating system, and
architecture.

{% code %}

```output
rr version 2023.1.2 (build time: 2023-05-04T13:19:13+0000, go1.20.4), OS: linux, arch: amd64
```

{% endcode %}

## Starting the Server

To start the server, you can use the following command:

{% code %}

```terminal
./rr serve
```

{% endcode %}

RoadRunner looks for `.rr.yaml` in the current working directory. Use `-c` to select a different configuration file:

{% code %}

```terminal
./rr serve -c /var/www/.rr.yaml
```

{% endcode %}

Use `-w` to select a working directory. Without `-w`, RoadRunner uses the directory that contains the selected configuration file:

{% code %}

```terminal
./rr serve -w /var/www
```

{% endcode %}

Use `--dotenv` to load an environment file before the command reads configuration. `DOTENV_PATH` takes precedence over this option:

{% code %}

```terminal
./rr serve --dotenv .env -c .rr.yaml
```

{% endcode %}

## PID file

Use `-p` to write the process ID to `.pid` in the working directory. The process stays in the foreground. Use a process manager such as [systemd](systemd.md) to run it as a service.

{% code %}

```terminal
./rr serve -p
```

{% endcode %}

## Debug mode

RoadRunner also supports running the Golang pprof server in debug mode using the `-d` option. This option enables the
debug mode, which starts the pprof server and listens for incoming requests on port 6061.

{% code %}

```terminal
./rr serve -d -c .rr.yaml
```

{% endcode %}

## Experimental features

RoadRunner also supports experimental features.
To enable experimental features, use the `--enable-experimental` or `-e` option:

{% code %}

```terminal
./rr serve --enable-experimental -c .rr.yaml
```

{% endcode %}

{% code %}

```terminal
./rr serve -e -c .rr.yaml
```

{% endcode %}

## Options

- `-c` - selects the configuration file. The default is `.rr.yaml` in the current working directory.
- `-w` - sets the working directory. Without this option, RoadRunner uses the configuration file's directory.
- `--dotenv` - loads an environment file. Existing process environment values take precedence over values in the file.
- `-d` - starts a Go profiling server. See [pprof](https://pkg.go.dev/net/http/pprof).
- `-s` - suppresses command status messages. Configure plugin logs through the `logs` section.
- `-o` - overrides a configuration key, such as `-o http.address=127.0.0.1:8080`. Included files can override these values.
- `-p` - creates a `.pid` file for `rr stop`.
- `-e` or `--enable-experimental` - enables experimental features. This option is useful when you want to test new
  features that are not yet available in the stable release.

## Stopping the Server

To stop RoadRunner, you have a few options:

- You can send a `SIGINT`, `SIGTERM`, or `SIGQUIT` signal to the main RoadRunner process. Inside a Kubernetes environment, this
  is done automatically when you're stopping the pod. All these signals trigger a graceful shutdown.
- If you want to stop RoadRunner manually, you can hit `ctrl+c` for a graceful stop or hit `ctrl+c` one more time to
  force stop.

{% hint style="info" %}
`SIGQUIT` is only available on Unix-like systems (Linux, macOS, FreeBSD, etc.) and is not supported on Windows.
{% endhint %}

{% hint style="info" %}
The default grace period is 30 seconds. Set `endure.grace_period` to change it. See [plugin shutdown](../customization/plugin.md#serving).
{% endhint %}

You can also use the following command to stop the server:

{% code %}

```terminal
./rr stop
```

{% endcode %}

If you want to force stop the server, you can use the `-f` option:

{% code %}

```terminal
./rr stop -f
```

{% endcode %}

{% hint style="info" %}
The `rr stop` command can only be used to stop a RoadRunner server that was started with the `-p` option and has a `.pid` file.
{% endhint %}

### Options

- `-f` - force stop.
- `-s` - silent mode.

## Restarting the Server

Use `SIGUSR2` on Unix-like systems to reload RoadRunner configuration. RoadRunner stops its plugins, then starts the same executable with the same arguments and environment. Listeners close during this restart, so requests need another available instance if continuous service is required.

To restart RoadRunner, send the `SIGUSR2` signal to the main RoadRunner process:

{% code %}

```bash
kill -USR2 1234
```

{% endcode %}

Replace `1234` with the process ID of the main RoadRunner process.

When RoadRunner receives the `SIGUSR2` signal, it performs the following steps:

1. Resolves the executable path.
2. Stops workers and plugins.
3. Replaces the process with the same executable, arguments, and environment.
4. Reads the configuration during startup.

{% hint style="info" %}
If the configuration file is a symlink, RoadRunner will properly re-read the configuration from the symlink target after restart. This allows you to update the symlink to point to a new configuration file for deployment workflows.
{% endhint %}

{% hint style="warning" %}
`SIGUSR2` is only available on Unix-like systems (Linux, macOS, FreeBSD, etc.) and is not supported on Windows.
{% endhint %}

## Reloading workers

RoadRunner allows you to reload all workers.

{% code %}

```terminal
./rr reset
```

{% endcode %}

This command calls `resetter.Reset` for plugins that support worker reset. It loads new PHP code with the running server's configuration. Use [SIGUSR2](#restarting-the-server) to apply changes to `.rr.yaml`.

By default, this command displays output in the console. However, if you want to reload the workers silently, you can
use the `--silent` option:

{% code %}

```terminal
./rr reset --silent
```

{% endcode %}

{% hint style="info" %}
You can attach this command as a file watcher in your IDE.
{% endhint %}

Additionally, you can reload only particular plugins by specifying their names:

{% code %}

```terminal
./rr reset http
```

{% endcode %}

{% hint style="info" %}
The pool waits up to `pool.reset_timeout` for active work before it starts stopping workers. Requests waiting for a worker can still reach their allocation timeout. See [worker pool timeouts](../php/pool.md#timeouts-and-admission).
{% endhint %}

### Options

- `-c` - specifies the path to the configuration file. By default, RoadRunner looks for a .rr.yaml file in the current
  working directory. However, you can specify a different file using this option.

## Workers status (OS metrics)

RoadRunner provides a command to view the status of active workers. You can use this command to monitor the health and
performance of your workers and diagnose any issues.

{% code %}

```terminal
./rr workers
```

{% endcode %}

This command displays a table with the status of all workers, including their PID, status, number of executions, memory
usage, and creation time.

{% code %}

```output
Workers of [jobs]:
+---------+-----------+---------+---------+---------+--------------------+
|   PID   |  STATUS   |  EXECS  | MEMORY  |  CPU%   |      CREATED       |
+---------+-----------+---------+---------+---------+--------------------+
|    3458 | ready     |       0 | 45 MB   |    0.00 | 1 day ago          |
|    3460 | ready     |       0 | 45 MB   |    0.00 | 1 day ago          |
|    3461 | ready     |       0 | 46 MB   |    0.00 | 1 day ago          |
|    3462 | ready     |       0 | 46 MB   |    0.00 | 1 day ago          |
+---------+-----------+---------+---------+---------+--------------------+

Workers of [http]:
+---------+-----------+---------+---------+---------+--------------------+
|   PID   |  STATUS   |  EXECS  | MEMORY  |  CPU%   |      CREATED       |
+---------+-----------+---------+---------+---------+--------------------+
|    3454 | ready     |     396 | 79 MB   |    0.04 | 1 day ago          |
+---------+-----------+---------+---------+---------+--------------------+
```

{% endcode %}

You can also specify plugin names to view the status of workers for a particular plugin:

{% code %}

```terminal
./rr workers http
```

{% endcode %}

Use the `-i` option to enable interactive mode. In this mode, the command will update the statistics every second:

{% code %}

```terminal
./rr workers -i
```

{% endcode %}

This command is useful for debugging performance issues and identifying workers that are not performing as expected.
Additionally, you can use this command to monitor the health of your workers and identify any potential problems before
they become critical.

{% hint style="info" %}
When the `pool.debug` configuration option is set to `true`, the `rr workers` command will not display any workers in
the pool. This is because workers are only created when incoming requests arrive, so there may not be any active
workers at the time the command is executed.
{% endhint %}

### Options

- `-c` - specifies the path to the configuration file. By default, RoadRunner looks for a `.rr.yaml` file in the current
  working directory. However, you can specify a different file using this option.
- `-w` - sets the working directory. Without this option, RoadRunner uses the configuration file's directory.
- `-i` - interactive mode (updates statistics every second).

## Jobs commands

RoadRunner provides several CLI commands to manage jobs and pipelines.

{% tabs %}

{% tab title="Pause" %}

This command allows you to specify one or more pipelines to `pause`. For example, to pause pipelines `pipeline1` and
`pipeline2`, you can use the following command:

```bash
./rr jobs --pause pipeline1,pipeline2
```

{% endtab %}

{% tab title="Resume" %}

This command allows you to specify one or more pipelines to `resume`. For example, to resume pipelines `pipeline1` and
`pipeline2`, you can use the following command:

```bash
./rr jobs --resume pipeline1,pipeline2
```

{% endtab %}

{% tab title="Destroy" %}

This command allows you to specify one or more pipelines to `destroy`. For example, to destroy pipelines `pipeline1` and
`pipeline2`, you can use the following command:

```bash
./rr jobs --destroy pipeline1,pipeline2
```

{% endtab %}

{% endtabs %}

To list all running pipelines, you can use the following command:

{% code %}

```terminal
./rr jobs --list
```

{% endcode %}

### Options

- `-c` - specifies the path to the configuration file. By default, RoadRunner looks for a `.rr.yaml` file in the current
  working directory. However, you can specify a different file using this option.

## RoadRunner client

The `reset`, `workers`, and `jobs` commands use RoadRunner's RPC interface to interact with the server, and the
connection information is obtained from the configuration file.

This means that you can use a local RoadRunner binary as a client to connect to a remote server, as long as the server's
host and port information is specified in the local configuration. This can be useful in situations where you want to
manage a remote RoadRunner instance from your local machine.

To specify the server `host` and `port` information, you can use the `rpc` section of the `.rr.yaml` file.

For example:

{% code %}

```yaml
version: "3"

rpc:
  listen: "tcp://127.0.0.1:6001"
```

{% endcode %}

If the RPC address uses values from a dotenv file, pass `--dotenv` to the client command. These commands do not load the root `envfile` setting. See [environment files](../php/environment.md#dotenv).
