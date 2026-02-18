# gRPC Interceptors

RoadRunner gRPC plugin supports custom unary interceptors.

Use interceptors when you want to add cross-cutting behavior around RPC calls, such as:

- request/response logging,
- auth checks,
- rate limiting,
- custom metrics.

{% hint style="info" %}
At the moment, RoadRunner supports unary gRPC interceptors (`grpc.UnaryServerInterceptor`).
{% endhint %}

## Interceptor contract

Your plugin must implement this interface:

{% code title="grpc/api/interfaces.go" %}

```go
type Interceptor interface {
	UnaryServerInterceptor() grpc.UnaryServerInterceptor
	Name() string
}
```

{% endcode %}

The `Name()` return value is used in the `grpc.interceptors` configuration list.

## Configuration

Add interceptor names under the `grpc.interceptors` section:

{% code title=".rr.yaml" %}

```yaml
version: "3"

server:
  command: "php grpc-worker.php"

grpc:
  listen: "tcp://127.0.0.1:9001"

  proto:
    - "proto/helloworld.proto"

  interceptors:
    - "sample-grpc-interceptor"
```

{% endcode %}

{% hint style="warning" %}
Make sure every name in `grpc.interceptors` matches a registered interceptor plugin name.
{% endhint %}

## Execution order

RoadRunner applies configured interceptors in reverse order from the config list.

Example:

{% code title=".rr.yaml" %}

```yaml
grpc:
  interceptors: ["first", "second", "third"]
```

{% endcode %}

Execution order will be:

`third -> second -> first -> handler`

## Example plugin

Ready-to-use example code is available in the samples repository:

- [gRPC interceptor sample](https://github.com/roadrunner-server/samples/tree/master/grpc_interceptor)

## Build a custom RR binary with interceptor (step by step)

1. Clone the RoadRunner repository.
2. Add your interceptor plugin package (you can start from the sample above).
3. Register interceptor plugin in `container/plugins.go`.
4. Build RoadRunner.
5. Configure `grpc.interceptors` in `.rr.yaml`.
6. Start RR and verify requests pass through the interceptor.

### 1) Clone RoadRunner

{% code %}

```bash
git clone https://github.com/roadrunner-server/roadrunner.git
cd roadrunner
```

{% endcode %}

### 2) Add interceptor package

You can either:

- copy sample code into your project, or
- import the sample package directly.

To import the sample package directly:

{% code %}

```bash
go get github.com/roadrunner-server/samples/grpc_interceptor@latest
```

{% endcode %}

### 3) Register plugin in container

Edit `container/plugins.go` and add your interceptor plugin to the plugin list:

{% code title="container/plugins.go" %}

```go
import (
	grpcPlugin "github.com/roadrunner-server/grpc/v5"
	grpcInterceptor "github.com/roadrunner-server/samples/grpc_interceptor"
)

func Plugins() []any {
	return []any{
		// ...
		&grpcInterceptor.Plugin{},
		&grpcPlugin.Plugin{},
		// ...
	}
}
```

{% endcode %}

### 4) Build RR

{% code %}

```bash
make build
```

{% endcode %}

The binary is created as `./rr`.

### 4a) Build RR with Velox (alternative)

You can also use [Velox](../customization/build.md) to build a custom RoadRunner binary and include the interceptor plugin without editing `container/plugins.go` for every build.

{% code title="velox.toml" %}

```toml
[roadrunner]
ref = "v2025.1.5"

[github]
[github.token]
token = "${RT_TOKEN}"

[github.plugins]
appLogger = { ref = "v5.0.2", owner = "roadrunner-server", repository = "app-logger" }
logger = { ref = "v5.0.2", owner = "roadrunner-server", repository = "logger" }
lock = { ref = "v5.0.2", owner = "roadrunner-server", repository = "lock" }
rpc = { ref = "v5.0.2", owner = "roadrunner-server", repository = "rpc" }
server = { ref = "v5.0.2", owner = "roadrunner-server", repository = "server" }
grpc = { ref = "v5.0.2", owner = "roadrunner-server", repository = "grpc" }
grpcInterceptor = { ref = "master", owner = "roadrunner-server", repository = "samples", folder = "grpc_interceptor" }

[log]
level = "info"
mode = "production"
```

{% endcode %}

{% code %}

```bash
go install github.com/roadrunner-server/velox/v2025/cmd/vx@latest
vx build -c velox.toml -o .
```

{% endcode %}

The binary is created as `./rr`.

### 5) Configure interceptor in `.rr.yaml`

Use the plugin name returned by `Name()`:

{% code title=".rr.yaml" %}

```yaml
grpc:
  interceptors:
    - "sample-grpc-interceptor"
```

{% endcode %}

### 6) Run and verify

Start RoadRunner:

{% code %}

```bash
./rr serve -c .rr.yaml
```

{% endcode %}

Send a request using your preferred gRPC client (for example, `grpc-client-cli`) and verify interceptor logs in RR output.

## What's next?

1. [Intro into gRPC](./grpc.md)
2. [Writing a Middleware](../customization/middleware.md)
3. [Building RR with a custom plugin](../customization/build.md)
