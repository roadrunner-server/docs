# Embedding a Server

In some cases, it can be useful to embed a RoadRunner server inside another GO program. This is often the case in
microservice architectures where you may have a mandated GO framework for all the apps. In such cases it might not be
possible to run a stock roadrunner instance, and the only choice is to run roadrunner inside the main app framework /
program.

Here's an example of how to embed RoadRunner into a Go program with an HTTP handler:

Import RoadRunner library via `go get` command into your Go project:

{% code title="main.go" %}

```Bash
go get -u github.com/roadrunner-server/roadrunner/v2025/lib
```

{% endcode %}


## Create an RR instance

{% code title="main.go" %}

```go

import (
	"github.com/roadrunner-server/roadrunner/v2025/lib"
)

func main() {
	overrides := []string{} // List of configuration overrides
	plugins := lib.DefaultPluginsList() // List of RR plugins to enable
	rr, err := lib.NewRR(".rr.yaml", overrides, plugins)
}

```

{% endcode %}

Here we use the default list of plugins. The same list of plugin you would get if you were to run `rr serve` with a
stock roadrunner binary.

You can, however, choose only the plugins you want and add your own private plugins as well:

{% code title="main.go" %}

```go
import (
	"github.com/roadrunner-server/roadrunner/v2025/lib"
	httpPlugin "github.com/roadrunner-server/http/v5"
	"github.com/roadrunner-server/resetter/v5"
	"github.com/roadrunner-server/informer/v5"
	"github.com/yourCompay/yourCusomPlugin" // compillation error here, used only as an example.
)

func main() {
	overrides := []string{
    		"http.address=127.0.0.1:4444", // example override to set the http address
    		"http.pool.num_workers=4", // example override of how to set the number of php workers
	} // List of configuration overrides

	plugins := []interface{}{
		&informer.Plugin{},
    		&resetter.Plugin{},
    		// ...
    		&httpPlugin.Plugin{},
    		// ...
    		&yourCustomPlugin.Plugin{},
	}
	plugins := lib.DefaultPluginsList() // List of RR plugins to enable
	rr, err := lib.NewRR(".rr.yaml", overrides, plugins)
}
```

{% endcode %}


## Starting & Stopping Embedded Roadrunner

Once everything is ready, we can start the roadrunner instance:

{% code title="main.go" %}

```go
errCh := make(chan error, 1)
go func() {
    errCh <- rr.Serve()
}()
```

{% endcode %}

`rr.Serve()` will block until it returns an error or `nil` if it was stopped gracefully.

To gracefully stop the server, simply call `rr.Stop()`
