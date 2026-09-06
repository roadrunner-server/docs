# Embedding a server

In some cases, it can be useful to embed a RoadRunner server inside another Go program. This is often the case in
microservice architectures where you may have a mandated Go framework for all the apps. In such cases, it might not be
possible to run a stock RoadRunner instance, and the only choice is to run RoadRunner inside the main app framework/
program.

Here's an example of how to embed RoadRunner into a Go program with an HTTP handler:

Import the RoadRunner library into your Go project. Replace `REVISION` with the tag or commit you want to embed. For v6 plugins, select a RoadRunner revision that uses v6 plugin modules; do not assume that `@latest` selects that revision.

{% code title="main.go" %}

```bash
go get github.com/roadrunner-server/roadrunner/v2025/lib@REVISION
```

{% endcode %}

The library import remains `roadrunner/v2025/lib`. `NewRR` returns `(*RR, error)`. Use matching plugin versions and the [v6 plugin contracts](plugin.md#v6-migration); the embedding lifecycle below is unchanged.

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

You can select plugins and add your own plugin. Replace `example.com/my-plugin` with your module path. The omitted entries must include the dependencies required by the selected plugins:

{% code title="main.go" %}

```go
import (
    custom "example.com/my-plugin"
    httpPlugin "github.com/roadrunner-server/http/v6"
    "github.com/roadrunner-server/informer/v6"
    "github.com/roadrunner-server/resetter/v6"
    "github.com/roadrunner-server/roadrunner/v2025/lib"
)

func main() {
    overrides := []string{
        "http.address=127.0.0.1:4444",
        "http.pool.num_workers=4",
    }

    plugins := []any{
        &informer.Plugin{},
        &resetter.Plugin{},
        // ...
        &httpPlugin.Plugin{},
        // ...
        &custom.Plugin{},
    }
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
