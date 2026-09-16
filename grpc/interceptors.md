# gRPC Interceptors

The RoadRunner gRPC plugin supports custom unary interceptors in both v5.3.0 and v6.

Use an interceptor to add processing before or after an RPC call, such as:

- request/response logging,
- authentication checks,
- rate limiting,
- custom metrics.

{% hint style="info" %}
The plugin supports only unary interceptors (`grpc.UnaryServerInterceptor`). Streaming RPCs, including [server reflection](./grpc.md#server-reflection), do not call these interceptors.
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
    - "custom-grpc-interceptor"
```

{% endcode %}

{% hint style="warning" %}
Each name in `grpc.interceptors` must match the `Name()` of a registered interceptor plugin. RR fails to start if a configured interceptor is missing.
{% endhint %}

## Execution order

RoadRunner applies configured interceptors in the same order as the config list.

Example:

{% code title=".rr.yaml" %}

```yaml
grpc:
  interceptors: ["first", "second", "third"]
```

{% endcode %}

Execution order will be:

`first -> second -> third -> handler`

## Build a custom binary

Include your interceptor plugin in a [custom RR build](../customization/build.md). Use plugin versions compatible with the v6 beta. Listing an interceptor in `.rr.yaml` does not add its code to the binary.

For an interceptor that reads protobuf descriptors, see the [registry-based example](./protoreg.md#example-grpc-interceptor). Use the plugin's `Name()` in `grpc.interceptors`.

### Run and verify

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
