# gRPC

RoadRunner gRPC plugin enables PHP applications to communicate with gRPC clients.

It consists of two main parts:

1. **protoc-plugin `protoc-gen-php-grpc`:** This is a plugin for the protoc compiler that generates PHP code from a gRPC service definition file (`.proto`). It generates PHP classes that correspond to the service definition and message types. These classes provide an interface for handling incoming gRPC requests and sending responses back to the client.
2. **gRPC server:** This is a server that starts PHP workers and listens for incoming gRPC requests. It receives requests from gRPC clients, proxies them to the PHP workers, and sends the responses back to the client. The server is responsible for managing the lifecycle of the PHP workers and ensuring that they are available to handle requests.

For custom gRPC interceptor plugins, see [Interceptors](./interceptors.md).

## Protoc-plugin

The first step is to define a `.proto` file that describes the gRPC service and messages that your PHP application will handle.

In our documentation, we will use the following example of a `.proto` file that is stored in the `<app>/proto` directory:

{% code title="proto/helloworld.proto" %}

```proto
syntax = "proto3";

option go_package = "proto/greeter";
option php_namespace = "GRPC\\Greeter";
option php_metadata_namespace = "GRPC\\GPBMetadata";

package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

{% endcode %}

It defines a simple gRPC service called `Greeter` that takes a name as input and returns a greeting.

The `php_namespace` and `php_metadata_namespace` options allow you to specify the namespaces to use in the generated DTO and Service interface.

### Generating PHP code

After defining the proto file, you need to generate the PHP files using the `protoc` compiler and the `protoc-gen-php-grpc` plugin. You can install the plugin binary using Composer or download a pre-built binary from the GitHub releases page.

{% tabs %}

{% tab title="Prebuilt Binary" %}

The simplest way to get the latest version of `protoc-gen-php-grpc` plugin is to download one of the pre-built release binaries on the GitHub [releases page](https://github.com/roadrunner-server/roadrunner/releases).

Just download the appropriate archive from the release page and extract it into your desired application directory.

{% endtab %}

{% tab title="Composer" %}

If you use Composer to manage your PHP dependencies, you can install the `spiral/roadrunner-cli` package to download the latest version of `protoc-gen-php-grpc` plugin to your project's root directory.

**Install the package**

{% code %}

```bash
composer require spiral/roadrunner-cli
```

{% endcode %}

And run the following command to download the latest version of the plugin

{% code %}

```bash
./vendor/bin/rr download-protoc-binary
```

{% endcode %}

Server binary will be available at the root of your project.

{% hint style="warning" %}
PHP's extensions `php-curl` and `php-zip` are required. Check with `php --modules` your installed extensions.
{% endhint %}

{% endtab %}

{% endtabs %}

Once the plugin is installed, you can use the `protoc` command to compile the proto file into PHP files.

**Here's an example command:**

{% code %}

```bash
protoc --plugin=protoc-gen-php-grpc \
       --php_out=./generated \
       --php-grpc_out=./generated \
       proto/helloworld.proto
```

{% endcode %}

{% hint style="info" %}
Make sure that the `generated` directory exists and is writable.
{% endhint %}

After running the command, you can find the generated DTO and `GreeterInterface` files in the `<app>/generated/GRPC/Greeter` directory.

We recommend also registering the GRPC namespace in the `composer.json` file:

{% code title="composer.json" %}

```json
{
  "autoload": {
    "psr-4": {
      ...
      "GRPC\\": "generated/GRPC"
    }
  }
}
```

{% endcode %}

By doing this, you can easily use the generated PHP classes in your application.

### Using BUF plugin

You can also use the [BUF](https://buf.build/community/roadrunner-server-php-grpc) RoadRunner gRPC plugin to generate PHP code from the `.proto` file instead of using the `protoc` compiler.

To use BUF to generate PHP gRPC services, you'll need to create two files: `buf.yaml` and `buf.gen.yaml`.

Here's an example of a `buf.yaml` file:

{% code title="buf.yaml" %}

```yaml
version: v2
deps:
    - buf.build/googleapis/googleapis:fb98f92554c17ec159a0b35ea8ffca71bac14385

name: buf.build/<your_org_name>/<you_project_name>
lint:
  use:
    - DEFAULT
  except:
    - FIELD_NOT_REQUIRED
    - PACKAGE_NO_IMPORT_CYCLE
breaking:
  use:
    - FILE
  except:
    - EXTENSION_NO_DELETE
    - FIELD_SAME_DEFAULT
```

{% endcode %}

Note that you need to optionally replace `<your_org_name>` and `<your_project_name>` with your organization and project names created on the [BUF](https://login.buf.build/u/signup) website.

In the `deps` section, you can specify the dependencies that your `.proto` file relies on. In this example, we're using a dependency from the Google APIs repository.

Also, you may configure the linting and breaking changes rules.

Here's an example of a `buf.gen.yaml` file:

{% code title="buf.gen.yaml" %}

```yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/php:v26.1
    out: generated/php
  - remote: buf.build/community/roadrunner-server-php-grpc:v4.8.0
    out: generated/php
  - remote: buf.build/protocolbuffers/go:v1.32.0
    out: generated/go
    opt: paths=source_relative
  - remote: buf.build/grpc/go:v1.3.0
    out: generated/go
    opt:
      - paths=source_relative
      - require_unimplemented_servers=false
  - remote: buf.build/grpc/python:v1.63.0
    out: generated/python
  - remote: buf.build/protocolbuffers/python
    out: generated/python
  - remote: buf.build/protocolbuffers/pyi
    out: generated/python
```

{% endcode %}

As you can see, the `buf.gen.yaml` file specifies the plugins that will be used to generate the code. In this example, we're using the `buf.build/community/roadrunner-server-php-grpc` plugin to generate PHP gRPC services.

Also, you may generate code for other languages like Go and Python.

## PHP Client

The RoadRunner gRPC plugin comes with a convenient PHP package that simplifies the process of integrating the plugin with your PHP application.

### Installation

You can install the package via Composer using the following command:

{% code %}

```bash
composer require spiral/roadrunner-grpc
```

{% endcode %}

### Implement Service

Next, you will need to create a PHP class that implements the `Greeter` service defined in the `.proto` file. This class should implement the `GRPC/Greeter/GreeterInterface`.

Here's an example:

{% code title="Greeter.php" %}

```php
<?php

use Spiral\RoadRunner\GRPC;
use GRPC\Greeter\GreeterInterface;
use GRPC\Greeter\HelloRequest;
use GRPC\Greeter\HelloReply;

final class Greeter implements GreeterInterface
{
    public function SayHello(GRPC\ContextInterface $ctx, HelloRequest $in): HelloReply
    {
        $greeting = "Hello " . $in->getName() . "!";

        return new HelloReply([
            'message' => $greeting
        ]);
    }
}
```

{% endcode %}

### Usage

To use the `Greeter` service, you can create a PHP worker that registers the service with the gRPC server.

Here's an example of how to do this:

{% code title="grpc-worker.php" %}

```php
<?php

use GRPC\Greeter\GreeterInterface;
use Spiral\RoadRunner\GRPC\Invoker;
use Spiral\RoadRunner\GRPC\Server;
use Spiral\RoadRunner\Worker;

require __DIR__ . '/vendor/autoload.php';
require 'Greeter.php';

$server = new Server(new Invoker(), [
    'debug' => false, // optional (default: false)
]);

$server->registerService(GreeterInterface::class, new Greeter());

$server->serve(Worker::create());
```

{% endcode %}

After creating the worker, you need to configure RoadRunner to register the `proto/helloworld.proto` service.

Here's an example configuration:

{% tabs %}

{% tab title="Server command" %}

{% code title=".rr.yaml" %}

```yaml .rr.yaml
version: "3"

server:
  command: "php grpc-worker.php"

grpc:
  listen: "tcp://127.0.0.1:9001"

  proto:
    - "proto/helloworld.proto"
```

{% endcode %}

{% hint style="info" %}
You can define command to start server in the `server.command` section. It will be used to start PHP workers for all registered plugins, such as `grpc`, `http`, `jobs`, etc.
{% endhint %}

{% endtab %}

{% tab title="Worker command" %}

You can also define command to start server in the `grpc.pool.command` section to separate server and grpc workers.

{% code title=".rr.yaml" %}

```yaml
version: "3"

server:
  command: "php worker.php"

grpc:
  listen: "tcp://127.0.0.1:9001"

  pool:
    command: "php grpc-worker.php"

  proto:
    - "proto/helloworld.proto"
```

{% endcode %}

{% endtab %}

{% endtabs %}

{% hint style="info" %}
You can define multiple proto files in the `proto` section.
{% endhint %}

After configuring the server, you can start it using the following command:

{% code %}

```bash
./rr serve
```

{% endcode %}

This will start the gRPC server and make the `Greeter` service available for remote clients to call. You can use any gRPC client library in any language that supports gRPC to call the `SayHello` method.

To quickly verify your code's correctness, you can install [grpc-client-cli](https://github.com/vadimi/grpc-client-cli) which provides an interactive CLI tool for talking to gRPC services.

To access the `Greeter` service with `grpc-client-cli` run the following in a terminal:

`grpc-client-cli --proto proto/helloworld.proto 127.0.0.1:9001`

It will provide some interactive prompts asking you what service and method you want to call

```shell
? Choose a service:  [Use arrows to move, type to filter]
→ helloworld.Greeter

? Choose a method:  [Use arrows to move, type to filter]
  [..]
→ SayHello
```

You will then be prompted to manually form the JSON payload it expects and upon pressing enter, your `Greeter` service should respond in an amicable manner:

```
Message json (type ? to see defaults): {"name":"Roadrunner"}
{
  "message": "Hello Roadrunner!"
}
```

On the PHP side, you should see that the successful request has been logged:

```
2025-03-28T10:04:06+0000        DEBUG   server          req-resp mode   {"pid": 81551}
2025-03-28T10:04:06+0000        DEBUG   grpc            method was called successfully  {"method": "/helloworld.Greeter/SayHello", "start": "2025-03-28T10:04:06+0000", "elapsed": 0}
```

Congratulations on being able to communicate over gRPC to PHP using Roadrunner!

## Metrics

RoadRunner has a [metrics plugin](../lab/metrics.md) that provides metrics for the gRPC server, which can be used with Prometheus and a preconfigured [Grafana dashboard](../lab/dashboards/grpc.md)

![grpc-metrics](https://user-images.githubusercontent.com/773481/235685443-05cf8af0-9e43-4aed-8801-da6595ca7d19.png)

## mTLS

To enable [mTLS](https://www.cloudflare.com/en-gb/learning/access-management/what-is-mutual-tls/) use the following configuration:

{% code title=".rr.yaml" %}

```yaml
version: "3"

grpc:
  listen: "tcp://127.0.0.1:9001"

  proto:
    - "first.proto"
    - "second.proto"

  tls:
    key: "server-key.pem"
    cert: "server-cert.pem"
    root_ca: "rootCA.pem"
    client_auth_type: request_client_cert
```

{% endcode %}

Options for the `client_auth_type` are:

- `request_client_cert`
- `require_any_client_cert`
- `verify_client_cert_if_given`
- `require_and_verify_client_cert`
- `no_client_certs`

## Full example of Configuration

{% code title=".rr.yaml" %}

```yaml
version: "3"

grpc:
  # GRPC address to listen
  #
  # This option is required
  listen: "tcp://127.0.0.1:9001"

  # Proto file to use, multiply files supported [SINCE 2.6]. As of [2023.1.4], wildcards are allowed in the proto field.
  #
  # This option is required
  proto:
    - "*.proto"
    - "first.proto"
    - "second.proto"

  # GRPC TLS configuration
  #
  # This section is optional
  tls:
    # Path to the key file
    #
    # This option is required
    key: ""

    # Path to the certificate
    #
    # This option is required
    cert: ""

    # Path to the CA certificate, defines the set of root certificate authorities that servers use if required to verify a client certificate. Used with the `client_auth_type` option.
    #
    # This option is optional
    root_ca: ""

    # Client auth type.
    #
    # This option is optional. Default value: no_client_certs. Possible values: request_client_cert, require_any_client_cert, verify_client_cert_if_given, require_and_verify_client_cert, no_client_certs
    client_auth_type: no_client_certs

  # Maximum send message size
  #
  # This option is optional. Default value: 50 (MB)
  max_send_msg_size: 50

  # Maximum receive message size
  #
  # This option is optional. Default value: 50 (MB)
  max_recv_msg_size: 50

  # MaxConnectionIdle is a duration for the amount of time after which an
  #	idle connection would be closed by sending a GoAway. Idleness duration is
  #	defined since the most recent time the number of outstanding RPCs became
  #	zero or the connection establishment.
  #
  # This option is optional. Default value: infinity.
  max_connection_idle: 0s

  # MaxConnectionAge is a duration for the maximum amount of time a
  #	connection may exist before it will be closed by sending a GoAway. A
  #	random jitter of +/-10% will be added to MaxConnectionAge to spread out
  #	connection storms.
  #
  # This option is optional. Default value: infinity.
  max_connection_age: 0s

  # MaxConnectionAgeGrace is an additive period after MaxConnectionAge after
  #	which the connection will be forcibly closed.
  max_connection_age_grace: 0s8h

  # Maximal concurrent streams count.
  #
  # This option is optional. Default value: 10
  max_concurrent_streams: 10

  # After a duration of this time if the server doesn't see any activity it
  #	pings the client to see if the transport is still alive.
  #	If set below 1s, a minimum value of 1s will be used instead.
  #
  # This option is optional. Default value: 2h
  ping_time: 1s

  # After having pinged for keepalive check, the server waits for a duration
  #	of Timeout and if no activity is seen even after that the connection is
  #	closed.
  #
  # This option is optional. Default value: 20s
  timeout: 200s

  # Usual workers pool configuration
  pool:
    # Debug mode for the pool. In this mode, pool will not pre-allocate the worker. Worker (only 1, num_workers ignored) will be allocated right after the request arrived.
    #
    # Default: false
    debug: false

    # Override server's command
    #
    # Default: empty
    command: "php my-super-app.php"

    # How many worker processes will be started. Zero (or nothing) means the number of logical CPUs.
    #
    # Default: 0
    num_workers: 0

    # Maximal count of worker executions. Zero (or nothing) means no limit.
    #
    # Default: 0
    max_jobs: 0

    # Timeout for worker allocation. Zero means 60s.
    #
    # Default: 60s
    allocate_timeout: 60s

    # Timeout for the reset timeout. Zero means 60s.
    #
    # Default: 60s
    reset_timeout: 60s

    # Timeout for worker destroying before process killing. Zero means 60s.
    #
    # Default: 60s
    destroy_timeout: 60s
```

{% endcode %}


## OTLP support in the `gRPC` plugin: `[>=2023.3.8]`

In the `v2023.3.8` we added experimental support for the `OTLP` protocol in the `gRPC` plugin. Stable since `v2024.1.1`. To enable it, you need to activate `otel` plugin by adding the following lines to the `.rr.yaml` file:

{% code title=".rr.yaml" %}

```yaml
otel: # <- activate otel plugin
  resource:
    service_name: "rr_test_grpc"
    service_version: "1.0.0"
    service_namespace: "RR-gRPC"
    service_instance_id: "UUID-super-long-unique-id"
  insecure: false
  exporter: stderr
```

{% endcode %}

Trace keys passed to the PHP workers are:

1. `Traceparent`
2. `Uber-Trace-Id`

Example:

{% code %}

```log
"Traceparent":["00-2678b910f57fe3320587f4126a390868-6b87f1600005b643-01"],"Uber-Trace-Id":["2678b910f57fe3320587f4126a390868:6b87f1600005b643:0:1"]
```

{% endcode %}

More about `OTLP` plugin you may read [here](../lab/otel.md).

## Common issues

1. Registering two services with the same name is not allowed. GRPC server will panic after that.
