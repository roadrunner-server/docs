# Writing Plugins

RoadRunner provides the ability to create custom plugins, event listeners, middlewares, etc., that extend its
functionality. It uses the Endure container to manage dependencies. This approach is similar to the PHP container
implementation with automatic method injection.

**To create a custom plugin, you can follow these steps:**

- Define a struct with a public `Init` method that returns an error value.
- Implement `Serve` and `Stop` only if the plugin starts a service.
- Request dependencies using their respective interfaces and inject them using the Endure container.
- Register the plugin in the RoadRunner container and [build the binary](build.md).

Below you can find more information about the plugin interface, how to define a plugin, and how to access other plugins.

## v6 migration

Use the module versions selected by the RoadRunner build. The v6 plugin beta still uses `endure/v2 v2.6.2`. Its lifecycle and dependency injection interfaces do not require a migration. The RoadRunner library module remains `roadrunner/v2025`.

The API repositories now have separate roles. [api](https://github.com/roadrunner-server/api) contains protobuf source, not a Go module. [api-go](https://github.com/roadrunner-server/api-go) contains generated Go bindings. [api-plugins](https://github.com/roadrunner-server/api-plugins) contains Go plugin contracts, not RPC messages.

All import paths in this table start with `github.com/roadrunner-server/`:

| Previous import | v6 plugin beta import |
| --- | --- |
| `<plugin>/v5` | `<plugin>/v6` |
| `pool/<package>` | `pool/v2/<package>` |
| `goridge/v3/pkg/<package>` | `goridge/v4/pkg/<package>` |
| `api/v4/build/<component>/v1` | `api-go/v6/<component>/v1` |
| `api/v4/build/lock/v1beta1` | `api-go/v6/lock/v1` |
| `api/v4/build/status/v1beta1` | `api-go/v6/status/v1` |
| `api/v4/plugins/v4/jobs` | `api-plugins/v6/jobs` |
| `api/v4/plugins/v1/{kv,lock,logger,status}` | `api-plugins/v6/{kv,lock,logger,status}` |
| `api/v4/plugins/v4/priority_queue` | `api-plugins/v6/priority_queue` |

Generated imports have no `build/` segment. For example:

```go
import jobsv1 "github.com/roadrunner-server/api-go/v6/jobs/v1"
```

Update implementations, local interfaces, and call sites together:

- **Logging:** use `logger.Named` from `api-plugins/v6/logger` or a local interface with `NamedLogger(string) *slog.Logger`. The old `logger.Log` interface is removed. Pool constructors, worker factories, and logger options also take `*slog.Logger`. Replace `log.Info("started", zap.String("plugin", name))` with `log.Info("started", "plugin", name)`. Slog has no `Fatal`, `Panic`, or `DPanic` methods.
- **Jobs:** add a leading `context.Context` to `DriverFromConfig` and `DriverFromPipeline`. Calls become `constructor.DriverFromConfig(ctx, key, queue, pipeline)` and `constructor.DriverFromPipeline(ctx, pipeline, queue)`. Existing `Driver` methods already take contexts. See the [Jobs driver tutorial](jobs-driver.md).
- **KV:** every `Storage` method now takes a leading context, including `Stop`. Update calls such as `storage.Get(ctx, key)`, `storage.Set(ctx, items...)`, and `storage.Stop(ctx)`. Construction becomes `constructor.KvFromConfig(ctx, key)`. Pass the context to backend operations. See the [KV contracts](https://github.com/roadrunner-server/api-plugins/blob/v6.0.0-beta.2/kv/interface.go).
- **Queues:** lock queue signatures use `lock.Item` and `[]lock.Item`, not the old priority-queue package's named interface. Jobs defines its own `jobs.Item`. Both retain `ID`, `GroupID`, and `Priority`. Update queue type arguments and method signatures. The `priority_queue` package now declares the Go package name `priorityqueue`.
- **Pool defaults:** direct calls to `DynamicAllocationOpts.InitDefaults()` must pass the base worker count: `InitDefaults(cfg.NumWorkers)`. Pool execution and shutdown guidance must match the [pinned pool version](../php/pool.md).
- **Removed helpers:** replace `proxy.Cidrs` from `proxy_ip_parser` with `[]*net.IPNet`. Resetter no longer exposes `Plugin.Reset(string)`; its `resetter.Reset` RPC remains available. OTEL removes `HTTPHandler` and `TemporalHandler`; use `Plugin.Middleware` and `Plugin.WorkerInterceptor`. Temporal no longer exposes `ResetAP`; normal activity-worker replacement is handled by the pool.

### DTO compatibility

`api-go/v6 v6.0.0-beta.14` retains the v1 message set. Do not use the v2 DTO packages from earlier betas. The `lock/v1` package defines `Request` and `Response`, not `LockRequest` and `LockResponse`. Regenerated PHP lock DTOs use `RoadRunner\Lock\DTO\V1`. The lock field numbers and types are unchanged; custom descriptor or protobuf `Any` users must account for the package-name change.

Relocation alone does not change the retained HTTP or Jobs wire fields and does not require a PHP worker-loop rewrite. RPC still uses Goridge and Go `net/rpc`, not Connect. See [RPC compatibility](../php/rpc.md#v6-compatibility) for the MessagePack change.

Direct Centrifugo DTO users have separate changes. Use typed fields instead of `Command.id/method/params` and `Reply.id/result`, which are removed. The `RateLimit` RPC and its types are removed. `UpdatePushStatusRequest.uid` becomes `analytics_uid` at the same string field number 1; update generated accessors and JSON names. The proxy bindings add experimental `NotifyCacheEmpty`; a custom server must implement it or embed the generated unimplemented server. The RoadRunner plugin forwards this event to PHP; update handlers and DTOs as described in [Centrifuge](../plugins/centrifuge.md).

## Interface

A plugin that starts a service implements `Service`, which provides `Serve` and `Stop`. Middleware and other plugins that do not start a service do not need those methods. Optional interfaces such as `Named`, `Provider`, `Weighted`, and `Collector` provide names, dependencies, initialization weights, and dependency collection.

**Here is an example:**

{% code title="plugin.go" %}

```go
package sample

import (
    "context"

    "github.com/roadrunner-server/endure/v2/dep"
)

type (
    // Service interface can be implemented by the plugin to use start/stop functionality
    Service interface {
        // Serve starts the plugin
        Serve() chan error
        // Stop stops the plugin
        Stop(context.Context) error
    }

    // Named -> name of the service
    Named interface {
        // Name returns a user-friendly name of the plugin
        Name() string
    }

    // Provider declares the ability to provide service edges of declared types.
    Provider interface {
        // Provides returns a set of functions that provide dependencies to other plugins
        Provides() []*dep.Out
    }

    // Weighted is optional to implement, but when implemented, the return value is used during topological sort
    Weighted interface {
        Weight() uint
    }

    // Collector declares the ability to accept plugins that match the provided method signature.
    Collector interface {
        // Collects searches for plugins that implement the given interfaces in the args
        Collects() []*dep.In
    }
)
```

{% endcode %}

## Plugin definition

To define a custom plugin, create a struct with a public `Init` method that returns an error value (you can
use `roadrunner-server/errors` as the `error` package). In this method, you can access other plugins by requesting
dependencies.

{% code title="plugin.go" %}

```go
package custom

const PluginName = "custom"

type Plugin struct{}

func (s *Plugin) Init() error {
    return nil
}
```

{% endcode %}

## Disabling a plugin

Sometimes, you may want to disable a plugin at runtime based on certain conditions. For example, if there are no
configurations for the plugin, or if there is an initialization error but you still do not want to stop server execution.
In such cases, you can return the special type of error called `Disabled`, which can be found in
the `github.com/roadrunner-server/errors` package. This type of error can only be used in the `Init` function of the
plugin.

{% code title="plugin.go" %}

```go
package custom

import (
    "github.com/roadrunner-server/errors"
)

const PluginName = "custom"

type Configurer interface {
    // UnmarshalKey takes a single key and unmarshal it into a Struct.
    UnmarshalKey(name string, out any) error
    // Has checks if config section exists.
    Has(name string) bool
}

type Plugin struct{}

func (s *Plugin) Init(cfg Configurer) error {
    const op = errors.Op("custom_plugin_init")
    // In this sample code, we're checking with the help of the configurer plugin if the `custom` configuration section exists
    // and if not, disabling the plugin.
    if !cfg.Has(PluginName) {
        return errors.E(op, errors.Disabled)
    }

    return nil
}
```

{% endcode %}

## Dependencies

You can access other plugins by requesting dependencies in your `Init` method. All dependencies should be represented as
interfaces, and a plugin implementing this interface should be registered in RR's container, Endure.

{% code title="plugin.go" %}

```go
package custom

import (
    "log/slog"
)

type Configurer interface { // <-- config plugin implements
    // UnmarshalKey takes a single key and unmarshal it into a Struct.
    UnmarshalKey(name string, out any) error
    // Has checks if config section exists.
    Has(name string) bool
}

type Logger interface { // <-- logger plugin implements
    NamedLogger(name string) *slog.Logger
}

type Service struct{}

func (s *Service) Init(r Configurer, log Logger) error {
    return nil
}
```

{% endcode %}

## Configuration

In most cases, your services would require a set of configuration values. RoadRunner can automatically populate and
validate your configuration structure using the `config` plugin via an interface.

### YAML configuration sample

{% code title=".rr.yaml" %}

```yaml
custom:
  address: tcp://127.0.0.1:8888
```

{% endcode %}

### Plugin

{% code title="plugin.go" %}

```go
package custom

import (
    "log/slog"

    "github.com/roadrunner-server/errors"
)

const PluginName = "custom"

type Configurer interface { // <-- config plugin implements
    // UnmarshalKey takes a single key and unmarshal it into a Struct.
    UnmarshalKey(name string, out any) error
    // Has checks if config section exists.
    Has(name string) bool
}

type Logger interface { // <-- logger plugin implements
    NamedLogger(name string) *slog.Logger
}

type Plugin struct {
    cfg *Config
    log *slog.Logger
}

// Init plugin
// file: plugin.go
func (s *Plugin) Init(cfg Configurer, log Logger) error {
    const op = errors.Op("custom_plugin_init") // error operation name
    if !cfg.Has(PluginName) {
        return errors.E(op, errors.Disabled)
    }

    s.log = log.NamedLogger(PluginName)

    // unmarshal initial configuration
    err := cfg.UnmarshalKey(PluginName, &s.cfg)
    if err != nil {
        // Error will stop execution
        return errors.E(op, err)
    }

    // Check the unmarshaled configuration and fill in defaults if not provided by the configuration
    s.cfg.InitDefaults()

    return nil
}
```

{% endcode %}

### Configuration type

{% code title="config.go" %}

```go
package custom

type Config struct {
    Address string `mapstructure:"address"`
}

// InitDefaults .. You can also initialize some default values for config keys
func (cfg *Config) InitDefaults() {
    if cfg.Address == "" {
        cfg.Address = "tcp://127.0.0.1:8088"
    }
}
```

{% endcode %}

## Serving

Endure calls `Serve()` synchronously. Start blocking work in a goroutine owned by the plugin, then return an error channel promptly. A blocking `Serve()` prevents the remaining plugins from starting and prevents container shutdown.

`Stop(ctx)` must stop the service cooperatively and respect the context deadline. The `endure.grace_period` setting determines this deadline. Endure does not terminate plugin goroutines when the deadline expires.

{% code title=".rr.yaml" %}

```yaml
## RoadRunner internal container configuration (docs: https://github.com/roadrunner-server/endure).
endure:
  # How long to wait for stopping.
  #
  # Default: 30s
  grace_period: 30s
```

{% endcode %}

### Plugin

This example starts a local HTTP server. Its `Stop` method uses `http.Server.Shutdown(ctx)` to wait for active requests until the context expires.

{% code title="plugin.go" %}

```go
package custom

import (
    "context"
    "net/http"
)

type Plugin struct {
    server *http.Server
}

func (s *Plugin) Init() error {
    s.server = &http.Server{
        Addr:    "127.0.0.1:8088",
        Handler: http.NotFoundHandler(),
    }
    return nil
}

func (s *Plugin) Serve() chan error {
    errCh := make(chan error, 1)

    go func() {
        if err := s.server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            errCh <- err
        }
    }()

    return errCh
}

func (s *Plugin) Stop(ctx context.Context) error {
    return s.server.Shutdown(ctx)
}
```

{% endcode %}

`http.ErrServerClosed` is the normal result of HTTP shutdown, so the example does not send it to Endure. Other errors are sent through the buffered channel. Endure does not make plugin code thread-safe; the plugin must synchronize access to shared state.

## Collecting dependencies at runtime

RoadRunner provides a way to collect dependencies at runtime via the `Collects` interface.
This is very useful for middlewares or extending plugins with additional functionality without changing them.

Let's create an HTTP middleware:

Declare the required interface:

{% code title="middleware.go" %}

```go
package custom

import (
    "net/http"
)

// Middleware interface
type Middleware interface {
    Middleware(f http.Handler) http.Handler
    Name() string
}
```

{% endcode %}

Implement the `Collects` interface in the plugin that accepts the middleware:

{% code title="middleware.go" %}

```go
package custom

// Collects HTTP middleware
func (p *Plugin) Collects() []*dep.In {
    return []*dep.In{
        dep.Fits(func(pp any) {
            mdw := pp.(Middleware)
            // add the middleware to the list
        }, (*Middleware)(nil)),
    }
}
```

{% endcode %}

Important notes:

1. `dep.Fits`: method used to check all registered plugins that fit the specified interface.
2. `func(pp any){}`: is a callback. You can pass an existing method with a `func (_ any)` signature or anonymous as in
   the example.
3. `(*Middleware)(nil)`: is the second argument of the `dep.Fits` method which should be an interface you want to find
   in the registered plugins.

## RPC Methods

Expose an RPC receiver through `RPC() any`. Its exported methods use the Go `net/rpc` signature: an input argument, a reply pointer, and an `error` result. Do not add a context argument to these RPC methods when updating the Jobs or KV contracts.

**Example based on the `informer` plugin:**

Suppose we have created a file `rpc.go`. The next step is to create a structure:

Create the receiver type:

{% code title="rpc.go" %}

```go
package custom

import (
    "log/slog"
)

type rpc struct {
    plugin *Plugin
    log    *slog.Logger
}
```

{% endcode %}

Add an exported method:

{% code title="rpc.go" %}

```go
package custom

func (s *rpc) Hello(input string, output *string) error {
    *output = input
    // s.plugin.Foo() <-- you may also use methods from the Plugin itself
    s.log.Debug("foo")
    return nil
}
```

{% endcode %}

Add `RPC()` to the plugin:

{% code title="rpc.go" %}

```go
package custom

func (p *Plugin) RPC() any {
    return &rpc{plugin: p, log: p.log}
}
```

{% endcode %}

RPC plugin will automatically find and register
your [RPC](https://github.com/roadrunner-server/rpc/blob/master/plugin.go#L184) methods under your plugin name. So, for
example, to call the `Hello` method you might use the following sample:

{% code title="file.php" %}

```php
var_dump($rpc->call('custom.Hello', 'world'));
```

{% endcode %}

## Tips

1. More about plugins can be found here: [link](https://github.com/roadrunner-server/endure/tree/master/examples)
