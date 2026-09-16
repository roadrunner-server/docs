# HTTP Middleware

RoadRunner provides a flexible and extensible architecture that allows developers to build custom middleware for
`http` and custom interceptors for the `grpc` and `temporal` plugins. Moving highly loaded parts of an application, such as
authentication, to middleware written in Go can provide a significant performance boost. By leveraging the speed and
efficiency of Go, developers can improve the overall performance of their application and handle spikes in traffic more
effectively.

The middleware architecture allows developers to create custom middleware for their specific needs. The HTTP
middleware can be used to intercept and modify HTTP requests and responses, while a gRPC interceptor can be used to
intercept and modify gRPC requests and responses. This allows developers to add functionality to their
applications without having to modify the core application logic.

## HTTP

The HTTP middleware intercepts incoming HTTP requests and can be used to perform additional processing, such as
authentication, rate limiting, and logging.

To create custom middleware for HTTP requests in RoadRunner, follow these steps:

1. Define a struct that implements the `Init()`, `Middleware()`, and `Name()` methods. The `Init()` method is called
   when the plugin is initialized, the `Middleware()` method is called for each incoming HTTP request, and the `Name()`
   method returns the name of the middleware/plugin.

2. In the `Middleware()` method, perform any necessary processing on the incoming HTTP request, and then call the next
   middleware in the pipeline using the `next.ServeHTTP()` method.

Here is an example:

{% code title="middleware.go" %}

```go
package middleware

import (
    "net/http"
)

const PluginName = "middleware"

type Plugin struct{}

// to declare plugin
func (p *Plugin) Init() error {
    return nil
}

func (p *Plugin) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // do something
        // ...
        // continue request through the middleware pipeline
        next.ServeHTTP(w, r)
    })
}

// Middleware/plugin name.
func (p *Plugin) Name() string {
    return PluginName
}
```

{% endcode %}

{% hint style="info" %}
The plugin must implement the [HTTP middleware interface](https://github.com/roadrunner-server/http/blob/v6.0.0-beta.10/api/interfaces.go#L36-L40), including `Name() string`. See [plugin migration](plugin.md#v6-migration) for the v6 imports and shared contracts.
{% endhint %}

## gRPC

The interceptor intercepts incoming gRPC requests and can be used to perform additional processing, such as
authentication, rate limiting, and logging.

To create a custom interceptor for gRPC requests in RoadRunner, follow these steps:

1. Define a struct with `Init()`, `UnaryServerInterceptor()`, and `Name()` methods. `UnaryServerInterceptor()` returns the interceptor used for incoming requests.

2. Process the request in the returned function, then call `handler(ctx, req)` to continue execution.

{% hint style="warning" %}
RoadRunner supports `gRPC` interceptors since version `v2023.2.0`.
{% endhint %}

Here is an example:

{% code title="middleware.go" %}

```go
package middleware

import (
    "context"

    "google.golang.org/grpc"
)

const PluginName = "interceptor"

type Plugin struct{}

// to declare plugin
func (p *Plugin) Init() error {
    return nil
}

func (p *Plugin) UnaryServerInterceptor() grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
        return handler(ctx, req)
    }
}

// Middleware/plugin name.
func (p *Plugin) Name() string {
    return PluginName
}
```

{% endcode %}

{% hint style="info" %}
The plugin must implement the [gRPC interceptor interface](https://github.com/roadrunner-server/grpc/blob/v6.0.0-beta.6/api/interfaces.go#L16-L19), including `Name() string`.
{% endhint %}

See [unary gRPC interceptors](../grpc/interceptors.md) for configuration and [go-grpc-middleware](https://github.com/grpc-ecosystem/go-grpc-middleware) for interceptor examples.

## PSR7 Attributes

PSR-7 server request attributes hold metadata for request processing. They are not response attributes.

Attributes can be used to store any kind of metadata that might be useful for processing the request or response. For
example, you might use attributes to store information about the authenticated user, the user's IP address, or any other
custom data that you want to attach to the request.

Use `Psr\Http\Message\ServerRequestInterface::getAttributes()` to read the attributes in PHP.

You can safely pass values to a PHP application and retrieve attributes on the PHP side using the `Psr\Http\Message\ServerRequestInterface->getAttributes()` method through the [attributes](https://github.com/roadrunner-server/http/blob/master/attributes/attributes.go) package:

{% code title="middleware.go" %}

```go
import (
    "net/http"

    "github.com/roadrunner-server/http/v6/attributes"
)

func (p *Plugin) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        r = attributes.Init(r)
        if err := attributes.Set(r, "key", "value"); err != nil {
            http.Error(w, "cannot set request attribute", http.StatusInternalServerError)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

{% endcode %}

{% hint style="info" %}
To retrieve the attributes in a PHP application, you would need to use a PSR-7 implementation that supports
the `getAttributes()` method. For example, the `nyholm/psr7` package provides a PSR-7 implementation that supports it.
{% endhint %}

## Registering middleware

Include the middleware plugin in your RoadRunner binary. Follow [Building RoadRunner](build.md) for the Velox configuration and build steps.

If you maintain the Go entry point yourself, import the middleware module and add `&middleware.Plugin{}` to the existing plugin list in [container/plugins.go](https://github.com/roadrunner-server/roadrunner/blob/master/container/plugins.go). Keep the other required plugins in that list.

Then add the value returned by `Name()` to `http.middleware`. A plugin included in the binary does not handle HTTP requests until it is selected in this list.

{% code title=".rr.yaml" %}

```yaml
http:
  # Provide the plugin name (in this example, "middleware")
  middleware: [ "middleware" ]
```

{% endcode %}

## Video tutorial

### Writing a middleware for HTTP

{% embed url="https://www.youtube.com/watch?v=f5fUSYaDKxo" %}
