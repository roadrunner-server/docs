# AWS Lambda

RoadRunner can run PHP as an AWS Lambda function.

## PHP Worker

The PHP worker does not require any specific configuration to run inside a Lambda function. We can use the default snippet with
an internal counter to demonstrate how workers are reused:

{% code title="handler.php" %}

```php
<?php
/**
 * @var Goridge\RelayInterface $relay
 */
use Spiral\Goridge;
use Spiral\RoadRunner;

ini_set('display_errors', 'stderr');
require __DIR__ . "/vendor/autoload.php";

$worker = RoadRunner\Worker::create();
$psr7 = new RoadRunner\Http\PSR7Worker(
    $worker,
    new \Nyholm\Psr7\Factory\Psr17Factory(),
    new \Nyholm\Psr7\Factory\Psr17Factory(),
    new \Nyholm\Psr7\Factory\Psr17Factory()
);

while ($req = $psr7->waitRequest()) {
    try {
        $resp = new \Nyholm\Psr7\Response();
        $resp->getBody()->write("hello world");

        $psr7->respond($resp);
    } catch (\Throwable $e) {
        $psr7->getWorker()->error((string)$e);
    }
}
```

{% endcode %}

Name this file `handler.php` and put it in the root of your project. Make sure to run:

{% code %}

```bash
composer require 'spiral/roadrunner-http:^4.1' nyholm/psr7
```

{% endcode %}

### Application

{% hint style="warning" %}
The Go example below uses v4 plugins and the legacy `sdk/v4` pool API. It is not a v6 build recipe. For a v6 build, apply the [plugin import and contract migration](../customization/plugin.md#v6-migration) and test the Lambda adapter with the selected RoadRunner version.
{% endhint %}

We can create a simple application to demonstrate how it works:

1. You need three files: `main.go` with the `Endure` container:

{% code title="main.go" %}

```go
package main

import (
  _ "embed"
  "log"
  "log/slog"
  "os"
  "os/signal"
  "sync"
  "syscall"
  "time"

  "github.com/roadrunner-server/config/v4"
  "github.com/roadrunner-server/endure/v2"
  "github.com/roadrunner-server/logger/v4"
  "github.com/roadrunner-server/server/v4"
)

//go:embed .rr.yaml
var rrYaml []byte

func main() {
  _ = os.Setenv("PATH", os.Getenv("PATH")+":"+os.Getenv("LAMBDA_TASK_ROOT"))
  _ = os.Setenv("LD_LIBRARY_PATH", "./lib:/lib64:/usr/lib64")

  cont := endure.New(slog.LevelError)

  cfg := &config.Plugin{
    Version:   "2024.1.0",
    Timeout:   time.Second * 30,
    Prefix:    "rr",
    Type:      "yaml",
    ReadInCfg: rrYaml,
  }

  err := cont.RegisterAll(
    cfg,
    &logger.Plugin{},
    &Plugin{},
    &server.Plugin{},
  )
  if err != nil {
    log.Fatal(err)
  }

  err = cont.Init()
  if err != nil {
    log.Fatal(err)
  }

  ch, err := cont.Serve()
  if err != nil {
    log.Fatal(err)
  }

  sig := make(chan os.Signal, 1)
  signal.Notify(sig, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)

  wg := &sync.WaitGroup{}
  wg.Add(1)

  go func() {
    defer wg.Done()
    for {
      select {
      case e := <-ch:
        err = cont.Stop()
        if err != nil {
          log.Println(e.Error.Error())
        }
      case <-sig:
        err = cont.Stop()
        if err != nil {
          log.Println(err.Error())
        }
        return
      }
    }
  }()

  wg.Wait()
}
```

{% endcode %}

2. `plugin.go` with the plugin implementation:

The adapter uses API Gateway HTTP API payload format `2.0` and the JSON protocol supported by [`spiral/roadrunner-http` 4.1](https://github.com/roadrunner-php/http/blob/v4.1.0/src/HttpWorker.php). HTTP metadata goes in `Payload.Context`. Raw body bytes go in `Payload.Body`. The adapter returns base64 response bodies so API Gateway can restore text and binary data. Form bodies remain raw; the PHP application must parse them if needed.

The execution context sets a 10-second budget for worker acquisition and the initial response. A nonzero `Supervisor.ExecTTL` gives each stream read a separate 10-second timeout. Cleanup cancels the execution context and drains the result channel. A stream read already in progress can continue until its timeout expires. The SDK then kills the worker, reaps the process, and starts a replacement. The handler reserves 10 seconds for cleanup plus a 1-second margin before the Lambda deadline. If too little time remains, the handler returns an error without executing PHP. To allow the full execution budget, set the Lambda timeout above 21 seconds, for example 30 seconds, with additional time for initialization and response delivery.

{% code title="plugin.go" %}

```go
package main

import (
  "context"
  "encoding/base64"
  "net/http"
  "net/url"
  "strings"
  "sync"
  "time"

  "github.com/goccy/go-json"
  "github.com/roadrunner-server/errors"
  "github.com/roadrunner-server/goridge/v3/pkg/frame"
  "github.com/roadrunner-server/sdk/v4/pool"
  "github.com/roadrunner-server/sdk/v4/worker"

  "github.com/aws/aws-lambda-go/events"
  "github.com/aws/aws-lambda-go/lambda"
  "github.com/roadrunner-server/sdk/v4/payload"
  poolImp "github.com/roadrunner-server/sdk/v4/pool/static_pool"
  "go.uber.org/zap"
)

const (
  pluginName string = "lambda"
  executionTimeout = 10 * time.Second
)

type Plugin struct {
  mu      sync.Mutex
  log     *zap.Logger
  srv     Server
  pldPool sync.Pool
  wrkPool Pool
}

// Logger plugin
type Logger interface {
  NamedLogger(name string) *zap.Logger
}

type Pool interface {
  // Workers returns workers list associated with the pool.
  Workers() (workers []*worker.Process)
  // Exec payload
  Exec(ctx context.Context, p *payload.Payload, stopCh chan struct{}) (chan *poolImp.PExec, error)
  // RemoveWorker removes worker from the pool.
  RemoveWorker(ctx context.Context) error
  // AddWorker adds worker to the pool.
  AddWorker() error
  // Reset kill all workers inside the watcher and replaces with new
  Reset(ctx context.Context) error
  // Destroy all underlying stacks (but let them complete the task).
  Destroy(ctx context.Context)
}

// Server creates workers for the application.
type Server interface {
  NewPool(ctx context.Context, cfg *pool.Config, env map[string]string, _ *zap.Logger) (*poolImp.Pool, error)
}

func (p *Plugin) Init(srv Server, log Logger) error {
  p.srv = srv
  p.log = log.NamedLogger(pluginName)
  p.pldPool = sync.Pool{
    New: func() any {
      return &payload.Payload{
        Codec:   frame.CodecJSON,
        Context: make([]byte, 0, 100),
        Body:    make([]byte, 0, 100),
      }
    },
  }

  return nil
}

func (p *Plugin) Serve() chan error {
  errCh := make(chan error, 1)
  const op = errors.Op("plugin_serve")

  p.mu.Lock()
  defer p.mu.Unlock()

  var err error
  p.wrkPool, err = p.srv.NewPool(context.Background(), &pool.Config{
    NumWorkers:      4,
    AllocateTimeout: time.Second * 20,
    DestroyTimeout:  time.Second * 20,
    StreamTimeout:   time.Second,
    Supervisor: &pool.SupervisorConfig{
      ExecTTL: executionTimeout,
    },
  }, nil, nil)
  if err != nil {
    errCh <- errors.E(op, err)
    return errCh
  }

  go func() {
    // register handler
    lambda.Start(p.handler())
  }()

  return errCh
}

func (p *Plugin) Stop(ctx context.Context) error {
  p.mu.Lock()
  defer p.mu.Unlock()

  if p.wrkPool != nil {
    p.wrkPool.Destroy(ctx)
  }

  return nil
}

func (p *Plugin) handler() func(ctx context.Context, request events.APIGatewayV2HTTPRequest) (events.APIGatewayV2HTTPResponse, error) {
  return func(ctx context.Context, request events.APIGatewayV2HTTPRequest) (events.APIGatewayV2HTTPResponse, error) {
    deadline := time.Now().Add(executionTimeout)
    if d, ok := ctx.Deadline(); ok {
      d = d.Add(-executionTimeout - time.Second)
      if d.Before(deadline) {
        deadline = d
      }
    }
    ctx, cancel := context.WithDeadline(ctx, deadline)
    defer cancel()
    if err := ctx.Err(); err != nil {
      return events.APIGatewayV2HTTPResponse{}, err
    }

    body := []byte(request.Body)
    if request.IsBase64Encoded {
      var err error
      body, err = base64.StdEncoding.DecodeString(request.Body)
      if err != nil {
        return events.APIGatewayV2HTTPResponse{StatusCode: 400}, nil
      }
    }

    headers := make(http.Header, len(request.Headers))
    for name, value := range request.Headers {
      headers.Set(name, value)
    }
    if len(request.Cookies) != 0 {
      headers.Set("Cookie", strings.Join(request.Cookies, "; "))
    }

    cookies := make(map[string]string)
    for _, cookie := range (&http.Request{Header: headers}).Cookies() {
      if value, err := url.QueryUnescape(cookie.Value); err == nil {
        cookies[cookie.Name] = value
      }
    }

    host := headers.Get("Host")
    if host == "" {
      host = request.RequestContext.DomainName
    }
    uri := "https://" + host + request.RawPath
    if request.RawQueryString != "" {
      uri += "?" + request.RawQueryString
    }

    metadata, err := json.Marshal(map[string]any{
      "remoteAddr": request.RequestContext.HTTP.SourceIP,
      "protocol":   request.RequestContext.HTTP.Protocol,
      "method":     request.RequestContext.HTTP.Method,
      "uri":        uri,
      "headers":    headers,
      "cookies":    cookies,
      "rawQuery":   request.RawQueryString,
      "parsed":     false,
    })
    if err != nil {
      return events.APIGatewayV2HTTPResponse{Body: "", StatusCode: 500}, nil
    }

    pld := p.getPld()
    defer p.putPld(pld)

    pld.Body = body
    pld.Context = metadata

    stopCh := make(chan struct{})
    re, err := p.wrkPool.Exec(ctx, pld, stopCh)
    if err != nil {
      return events.APIGatewayV2HTTPResponse{Body: "", StatusCode: 500}, nil
    }
    defer func() {
      // StreamCancel uses ctx. ExecTTL bounds a stream read already in progress.
      cancel()
      close(stopCh)
      for range re {
      }
    }()

    var r *payload.Payload

    select {
    case <-ctx.Done():
      return events.APIGatewayV2HTTPResponse{}, ctx.Err()
    case pl, ok := <-re:
      if !ok || pl == nil {
        return events.APIGatewayV2HTTPResponse{Body: "worker empty response", StatusCode: 500}, nil
      }
      if pl.Error() != nil {
        return events.APIGatewayV2HTTPResponse{Body: "", StatusCode: 500}, nil
      }
      r = pl.Payload()
      if r == nil {
        return events.APIGatewayV2HTTPResponse{Body: "worker empty response", StatusCode: 500}, nil
      }
      if r.Flags&frame.STREAM != 0 {
        return events.APIGatewayV2HTTPResponse{Body: "streaming is not supported", StatusCode: 500}, nil
      }
    }

    var responseMetadata struct {
      Status  int                 `json:"status"`
      Headers map[string][]string `json:"headers"`
    }
    err = json.Unmarshal(r.Context, &responseMetadata)
    if err != nil || responseMetadata.Status < 100 || responseMetadata.Status >= 600 {
      return events.APIGatewayV2HTTPResponse{Body: "", StatusCode: 500}, nil
    }

    response := events.APIGatewayV2HTTPResponse{
      StatusCode:      responseMetadata.Status,
      Headers:         make(map[string]string, len(responseMetadata.Headers)),
      Body:            base64.StdEncoding.EncodeToString(r.Body),
      IsBase64Encoded: true,
    }
    for name, values := range responseMetadata.Headers {
      if strings.EqualFold(name, "Set-Cookie") {
        response.Cookies = append(response.Cookies, values...)
      } else {
        response.Headers[name] = strings.Join(values, ", ")
      }
    }
    return response, nil
  }
}

func (p *Plugin) putPld(pld *payload.Payload) {
  pld.Body = nil
  pld.Context = nil
  p.pldPool.Put(pld)
}

func (p *Plugin) getPld() *payload.Payload {
  pld := p.pldPool.Get().(*payload.Payload)
  return pld
}
```

{% endcode %}

3. A config file, which can be embedded into the binary using the [`embed`](https://pkg.go.dev/embed) package:

{% code title=".rr.yaml" %}

```yaml
version: "3"

server:
  command: "php handler.php"
  relay: pipes

logs:
  mode: production
  level: error
  encoding: json
  output: [ stderr ]

endure:
  grace_period: 1s
```

{% endcode %}

Here you can take full advantage of RoadRunner: you can include any plugin here and configure it with the embedded config (within reasonable limits).

Build the binary with a supported Go release that includes current security fixes. If your project has no `go.mod`, run `go mod init example.com/lambda` from the project root.

The build uses `-mod=readonly` because Composer's `vendor` directory does not contain Go modules. AWS Lambda requires an executable named [`bootstrap`](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html#runtimes-custom-bootstrap) at the root of the deployment package. To build and package your Lambda function, run these commands from the project root:

{% code title="build.sh" %}

```bash
go mod tidy
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -mod=readonly -trimpath -ldflags "-s" -o bootstrap main.go plugin.go
zip main.zip * -r
```

{% endcode %}

You can now upload the function and connect it to an API Gateway HTTP API with payload format `2.0`. For a direct Lambda test, use an API Gateway v2 HTTP request event, not a string event. API Gateway v2 omits the custom-domain API mapping prefix from `rawPath`; this adapter uses the path supplied in the event.

## Repository with the full example

- [link](https://github.com/roadrunner-server/aws-lambda)

## Notes

There are multiple notes to acknowledge:

- Start with one worker per Lambda function to control your memory usage.
- Make sure to include the environment variables listed in the code to properly resolve the location of the PHP binary and its dependencies.
