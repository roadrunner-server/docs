# Logger

The logger plugin writes logs from RoadRunner plugins and PHP workers to standard error, standard output, or files. The `logs` section controls the format, minimum level, and output destinations.

{% hint style="info" %}
PHP worker standard error is logged at `info` level through the `server` channel. Set that channel to `info` or `debug` to retain worker output when the root level is higher:

{% code title=".rr.yaml" %}

```yaml
version: "3"

logs:
  mode: production
  level: error
  channels:
    server:
      mode: production
      level: info
```

{% endcode %}

{% endhint %}

## Configuration

### Modes

{% code title=".rr.yaml" %}

```yaml
logs:
  mode: production
```

{% endcode %}

| Mode | Output |
| --- | --- |
| `production` | JSON records with `time`, `level`, `msg`, and structured attributes. |
| `development` | Key/value text without console colors. This is the default mode. |
| `raw` | The message only. Attributes and groups are discarded. |
| `off`, `none` | No output from this logger. Channel overrides can still enable their own output. |

Unknown modes use the same text format as `development`.

Logger plugin v6 production records use a `time` string in RFC3339 format with up to nanosecond precision. Level names use uppercase, such as `INFO`. In v5, production records used a numeric `ts` value in epoch nanoseconds and lowercase levels. Update log parsers for these changes. Development output also changes from the v5 colored console format to key/value text.

Example v6 production record:

```json
{"time":"2026-08-17T12:00:00.123Z","level":"INFO","msg":"worker output","logger":"server"}
```

### Encoding

V6 ignores the `encoding` setting. Remove it from the root and channel configurations. Use `mode: production` for JSON or `mode: development` for text.

### Custom Format

Use `format` to select a custom text format. It takes precedence over `mode`, except that `off` and `none` still disable the logger. The `time_format` setting uses a [Go time layout](https://pkg.go.dev/time#Layout). Its default is RFC3339, and it applies only to `%time%` in a custom format.

{% code title=".rr.yaml" %}

```yaml
logs:
  level: info
  format: "%time% [%level%] %logger% %message% %attrs%"
  time_format: "2006-01-02 15:04:05"
```

{% endcode %}

| Placeholder | Value |
| --- | --- |
| `%time%` | Record time, using `time_format`. |
| `%level%` | Level name, such as `INFO`. |
| `%message%` | Log message. |
| `%logger%` | Logger name. |
| `%attrs%` | Attributes as space-separated `key=value` pairs. |
| `%source_file%` | Go source file path. |
| `%source_line%` | Go source line number. |
| `%source_func%` | Go function name. |

Unknown placeholders remain unchanged. When `%logger%` is present, `%attrs%` omits the `logger` attribute. Custom formats do not escape message or attribute text. Use production mode when you need JSON encoding.

### Level

The `level` setting is the minimum severity to emit. Supported values are `debug`, `info`, `warn`, and `error`. The value `warning` also selects `warn`. Values are case-insensitive.

{% code title=".rr.yaml" %}

```yaml
logs:
  level: info
```

{% endcode %}

{% hint style="info" %}
In v6, an empty or unknown level selects `debug`, including in production and raw channels. Values such as `panic`, `dpanic`, and `fatal` are not supported. Replace these v5 values with a supported level; they do not disable logging or fail configuration validation.
{% endhint %}

### Output

The logger writes to standard error by default. The `output` list accepts `stderr`, `stdout`, and file paths. The logger writes each enabled record to every destination in the list. Paths are file paths, not URLs.

{% code title=".rr.yaml" %}

```yaml
logs:
  output: [ stdout ]
```

{% endcode %}

### Error Output

All enabled levels of a logger use the same output destinations. V6 ignores `err_output`. In v5, this setting controlled internal logger errors, not error-level records. Remove it from your configuration. The `error_output` key is not supported.

### Line Endings

Custom formats append `\n` by default. Set `line_ending` to change it. To append nothing, set `skip_line_ending: true`. This takes precedence over `line_ending`. An empty `line_ending` without `skip_line_ending` still selects `\n`.

{% code title=".rr.yaml" %}

```yaml
logs:
  format: "%message%"
  line_ending: "\r\n"
```

{% endcode %}

These settings require a nonempty `format`. Standard production and development handlers append `\n`. Standard raw mode appends `\n` only when the message does not already end with it.

### Channels

Use `channels` to configure a plugin logger separately. A channel configuration replaces the root settings for that plugin. Omitted channel settings use their own defaults, not the root values. Plugins without a channel override use the root logger.

{% code title=".rr.yaml" %}

```yaml
version: "3"

logs:
  mode: none
  channels:
    http:
      mode: production
      level: info
      output: [ http.log ]
```

{% endcode %}

If a channel output cannot be opened, the logger reports the error through the root logger and uses the root settings for that channel. A disabled root logger hides this error. Check the root configuration when a channel file is missing. A root-output open error fails initialization.

## File Output

Use a file path in `output` to write logs to a file. The plugin creates the file if needed and appends to an existing file. Create its parent directory before starting RoadRunner.

{% code title=".rr.yaml" %}

```yaml
logs:
  mode: production
  level: info
  output: ["rr.log"]
```

{% endcode %}

V6 removes `file_logger_options`. Replace the v5 `file_logger_options.log_output` setting with an entry in `output`. For a channel, use `logs.channels.<name>.output` as shown above.

There is no built-in log rotation, backup retention, or compression in v6. The v5 `max_size`, `max_age`, `max_backups`, and `compress` settings have no effect. The plugin does not reopen files after external rotation. Send logs to stdout or stderr and let a service manager, container runtime, or log collector manage rotation.

## Startup Logs

{% code %}

```log
[INFO] RoadRunner server started; version: 2024.3.0, buildtime: 2024-12-05T18:39:32+0000
[INFO] sdnotify: not notified
```

{% endcode %}

These logs are not controlled by the logs configuration section. They are emitted directly by the RoadRunner core and can be turned off using the `-s` or `--silent` CLI option.

## Go Loggers

V6 uses Go's [log/slog](https://pkg.go.dev/log/slog) instead of Zap. Named loggers return `*slog.Logger`. Use a `slog.Handler` for custom output.

The plugin closes its root and channel file outputs during `Stop`. If you call `Config.BuildLogger()` directly, close each resource in the returned `BuildResult.Closers`. If you construct a `Log` with `NewLogger`, call `Log.Close()` to close its channel outputs.
