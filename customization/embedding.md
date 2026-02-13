# Embedding a server

In some cases, it can be useful to embed a RoadRunner server inside another Go program. This is often the case in
microservice architectures where you may have a mandated Go framework for all the apps. In such cases, it might not be
possible to run a stock RoadRunner instance, and the only choice is to run RoadRunner inside the main app framework/
program.

Here's an example of how to embed RoadRunner into a Go program with an HTTP handler:

Import the RoadRunner library via the `go get` command into your Go project:

{% code title="main.go" %}

```bash
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

Here we use the default list of plugins. This is the same list of plugins you would get if you were to run `rr serve` with a
stock RoadRunner binary.

You can, however, choose only the plugins you want and add your own private plugins as well:

{% code title="main.go" %}

```go
import (
	"github.com/roadrunner-server/roadrunner/v2025/lib"
	httpPlugin "github.com/roadrunner-server/http/v5"
	"github.com/roadrunner-server/resetter/v5"
	"github.com/roadrunner-server/informer/v5"
	"github.com/yourCompay/yourCusomPlugin" // compilation error here, used only as an example.
)

func main() {
	overrides := []string{
    		"http.address=127.0.0.1:4444", // example override to set the HTTP address
    		"http.pool.num_workers=4", // example override of how to set the number of PHP workers
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


## Starting & stopping embedded RoadRunner

Once everything is ready, we can start the RoadRunner instance:

{% code title="main.go" %}

```go
errCh := make(chan error, 1)
go func() {
    errCh <- rr.Serve()
}()
```

{% endcode %}

`rr.Serve()` will block until it returns an error or `nil` if it was stopped gracefully.

To gracefully stop the server, simply call `rr.Stop()`.
