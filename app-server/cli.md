# CLI Commands

RoadRunner offers a convenient CLI to start and manage the server.

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

By default, RoadRunner looks for a `.rr.yaml` file in the current working directory where the binary is placed. However,
you can also specify the configuration file from a different location using the `-c` option:

{% code %}

```terminal
./rr serve -c /var/www/.rr.yaml
```

{% endcode %}

You can also specify the working directory using the -w option:

{% code %}

```terminal
./rr serve -w /var/www
```

{% endcode %}

RoadRunner also supports `.env` files. To read environment variables from the `.env` file, use the `--dotenv` CLI
command:

{% code %}

```terminal
./rr serve --dotenv .env -c .rr.yaml
```

{% endcode %}

## Background mode

Additionally, you can start the server in background mode using the `-p` option. In this mode, RoadRunner creates
a `.pid` file:

{% code %}

```terminal
./rr serve -p
```

{% endcode %}

## Debug mode

RoadRunner also supports running the Golang pprof server in debug mode using the `-d` option. This option enables the
debug mode, which starts the pprof server and listens for incoming requests on the configured port.

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

- `-c` - specifies the path to the configuration file. By default, RoadRunner looks for a `.rr.yaml` file in the current
  working directory. However, you can specify a different file using this option.
- `-w` - sets the working directory for the server. By default, RoadRunner uses the current working directory.
- `--dotenv` - populates the process with environment variables from the `.dotenv` file. This can be useful when you
  want to set environment variables for your application.
- `-d` - starts a Golang profiling server. This allows you to analyze the performance of your application and
  identify potential bottlenecks. (Read more [here](https://pkg.go.dev/net/http/pprof).)
- `-s` - enables silent mode. In this mode, RoadRunner does not display any output in the console.
- `-o` - allows you to override configuration keys with your values. For example, `-o=http.address=8080` will override
  the `http.address` configuration key from the `.rr.yaml` file.
- `-p` - creates a `.pid` file to use with the rr stop command later. This can be useful when you want to run RoadRunner
  in the background mode.
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
By default, the grace period is 30 seconds. You can change it in the configuration file using `endure.grace_period`
setting. You can find an example [here](https://github.com/roadrunner-server/roadrunner/blob/master/.rr.yaml#L2115).
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

RoadRunner supports graceful restarts using the `SIGUSR2` signal on Unix-like systems. This allows you to restart the RoadRunner process without downtime, which is particularly useful in deployment scenarios.

To restart RoadRunner, send the `SIGUSR2` signal to the main RoadRunner process:

{% code %}

```bash
kill -USR2 <pid>
```

{% endcode %}

Where `<pid>` is the process ID of the main RoadRunner process.

When RoadRunner receives the `SIGUSR2` signal, it performs the following steps:

1. Validates the executable path
2. Notifies systemd (if running under systemd)
3. Gracefully stops all workers and plugins
4. Replaces the current process with a new one using the same executable path

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

This command reloads all RoadRunner workers, including HTTP, gRPC, and others, and starts them with a new worker pool.
This can be useful when you make changes to your application code and want to see the changes reflected in the running
server.

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
RoadRunner will wait for any active requests to finish processing before reloading the worker pool. This ensures that
all active requests are completed before the new worker pool is started.
Any new incoming requests that arrive during the reloading process will be queued and processed once the new worker
pool is started. This means that there may be a temporary delay in processing new requests while the worker pool is
being reloaded, but no requests should be lost.
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
- `-w` - sets the working directory for the server. By default, RoadRunner uses the current working directory.
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
rpc:
  listen: "127.0.0.1:6001"
```

{% endcode %}
