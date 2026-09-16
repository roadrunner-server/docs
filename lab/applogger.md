# Application logger

The app-logger plugin accepts log messages over RPC. Level-based methods use the `app` logger channel. The `log()` method writes directly to RoadRunner's standard error.

## Configuration

Configure the `app` channel in the `logs` section to select the output format and minimum level:

{% code title=".rr.yaml" %}

```yaml
rpc:
  listen: tcp://127.0.0.1:6001

logs:
  channels:
    app:
      mode: production
      level: info
```

{% endcode %}

{% hint style="warning" %}
Configure [RPC](../php/rpc.md#configuration) to use app-logger.
{% endhint %}

This example emits JSON records at `info` level or higher and filters out `debug`. These settings do not control raw log calls.

{% hint style="info" %}
See [Logger](./logger.md) for v6 formats, levels, and output destinations.
{% endhint %}

## PHP client

The RoadRunner `app-logger` plugin comes with a convenient PHP package that simplifies the process of integrating the
plugin with your PHP application.

### Installation

To get started, you can install the package via Composer using the following command:

{% code %}

```bash
composer require roadrunner-php/app-logger
```

{% endcode %}

### Usage

After the installation, you can create an instance of the `RoadRunner\Logger\Logger` class, which will allow you to use
the available class methods.

**Here is an example:**

{% code title="logger.php" %}

```php
use Spiral\Goridge\RPC\RPC;
use RoadRunner\Logger\Logger;

$rpc = RPC::create('tcp://127.0.0.1:6001');

$logger = new Logger($rpc);

$logger->info('Hello, RoadRunner!');
$logger->warning('Something might be wrong...');
$logger->error('Houston, we have a problem!');
```

{% endcode %}

{% hint style="info" %}
See [RPC connections](../php/rpc.md) for connection setup.
{% endhint %}

### Available methods

- `debug(string): void`: Sends a debug log message to the server
- `error(string): void`: Sends an error log message to the server
- `info(string): void`: Sends an info log message to the server
- `warning(string): void`: Sends a warning log message to the server
- `log(string): void`: Sends a log message directly to the `STDERR` of the server

### Raw Output

The raw RPC methods `app.Log` and `app.LogWithContext` bypass logger levels, formats, and output settings, including `logs.channels.app`. This differs from selecting the logger's `raw` mode, which still uses the configured level and destinations.

With app-logger v6, `app.Log` appends LF (`\n`) only when the message does not already end with LF. `app.LogWithContext` uses the same rule when there are no attributes. V5 wrote these messages without adding a line ending. Update consumers that depended on concatenated messages without line endings.

With attributes, `app.LogWithContext` writes the message, a space, comma-separated `key:value` pairs, and LF. It preserves newlines inside the message. Send the raw RPC message without a trailing newline if you need the attributes on the same line. V6 also retains the complete final attribute value instead of removing its last byte as v5 did.

## API

### RPC API

The string methods accept a message and a boolean reply placeholder. Call them by their registered RPC names:

| Method | Output |
| --- | --- |
| `app.Error` | Error-level record through the `app` logger. |
| `app.Info` | Info-level record through the `app` logger. |
| `app.Warning` | Warning-level record through the `app` logger. |
| `app.Debug` | Debug-level record through the `app` logger. |
| `app.Log` | Raw standard error output. |

Each method also has a `WithContext` variant, such as `app.InfoWithContext`. These methods accept `LogEntry` and a `Response` placeholder from `github.com/roadrunner-server/api-go/v6/applogger/v1`. The entry carries the message and `LogAttrs` key/value pairs.

The plugin's Go RPC receiver is no longer exported as `app.RPC`. Custom containers obtain it through `Plugin.RPC()`; PHP RPC method names are unchanged.
