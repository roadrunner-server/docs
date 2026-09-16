# AWS Lambda

RoadRunner can run PHP as an AWS Lambda function.

## PHP Worker

Use PHP `8.5` with the `sockets` extension and Composer 2. This worker returns an HTTP response for each invocation:

{% code title="handler.php" %}

```php
<?php
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

The application uses v6 plugins, `pool/v2`, `goridge/v4`, and the HTTP protobuf messages from `api-go/v6`. Its RoadRunner module versions match the [RoadRunner source build](../intro/install.md#build-from-source). The adapter follows the protobuf request and response structure in the [AWS Lambda example](https://github.com/roadrunner-server/aws-lambda).

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
  "syscall"
  "time"

  "github.com/roadrunner-server/config/v6"
  "github.com/roadrunner-server/endure/v2"
  "github.com/roadrunner-server/logger/v6"
  "github.com/roadrunner-server/server/v6"
)

//go:embed .rr.yaml
var rrYaml []byte

func main() {
  _ = os.Setenv("PATH", os.Getenv("PATH")+":"+os.Getenv("LAMBDA_TASK_ROOT"))
  _ = os.Setenv("LD_LIBRARY_PATH", "./lib:/lib64:/usr/lib64")

  cont := endure.New(slog.LevelError)

  cfg := &config.Plugin{
    Version:   "lambda-v6",
    Timeout:   time.Second * 30,
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
  signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
  defer signal.Stop(sig)

  select {
  case e := <-ch:
    if e != nil {
      log.Println(e.Error)
    }
  case <-sig:
  }

  if err := cont.Stop(); err != nil {
    log.Println(err)
  }
}
```

{% endcode %}

2. `plugin.go` with the plugin implementation:

The adapter uses API Gateway HTTP API payload format `2.0` and the HTTP protobuf protocol. It encodes `http/v1.Request` in `Payload.Context` and sends raw body bytes in `Payload.Body`. It decodes response metadata as `http/v1.Response`. The adapter returns base64 response bodies so API Gateway can restore text and binary data. Form bodies remain raw; the PHP application must parse them if needed.

The execution context sets a 10-second budget for worker acquisition and the initial response. A nonzero `Supervisor.ExecTTL` gives each stream read a separate 10-second timeout. Cleanup cancels the execution context and drains the result channel. A stream read already in progress can continue until its timeout expires. The pool then kills the worker, reaps the process, and starts a replacement. The handler reserves 10 seconds for cleanup plus a 1-second margin before the Lambda deadline. If too little time remains, the handler returns an error without executing PHP. To allow the full execution budget, set the Lambda timeout above 21 seconds, for example 30 seconds, with additional time for initialization and response delivery.

{% code title="plugin.go" %}

```go
package main

import (
  "context"
  "encoding/base64"
  "log/slog"
  "net/http"
  "net/url"
  "strings"
  "sync"
  "time"

  httpV1 "github.com/roadrunner-server/api-go/v6/http/v1"
  "github.com/roadrunner-server/errors"
  "github.com/roadrunner-server/goridge/v4/pkg/frame"
  "github.com/roadrunner-server/pool/v2/pool"
  "github.com/roadrunner-server/pool/v2/worker"

  "github.com/aws/aws-lambda-go/events"
  "github.com/aws/aws-lambda-go/lambda"
  "github.com/roadrunner-server/pool/v2/payload"
  poolImp "github.com/roadrunner-server/pool/v2/pool/static_pool"
  "google.golang.org/protobuf/proto"
)

const (
  pluginName string = "lambda"
  executionTimeout = 10 * time.Second
)

type Plugin struct {
  mu      sync.Mutex
  log     *slog.Logger
  srv     Server
  pldPool sync.Pool
  wrkPool Pool
}

// Logger plugin
type Logger interface {
  NamedLogger(name string) *slog.Logger
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
  NewPool(ctx context.Context, cfg *pool.Config, env map[string]string, _ *slog.Logger) (*poolImp.Pool, error)
}

func (p *Plugin) Init(srv Server, log Logger) error {
  p.srv = srv
  p.log = log.NamedLogger(pluginName)
  p.pldPool = sync.Pool{
    New: func() any {
      return &payload.Payload{
        Codec:   frame.CodecProto,
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

  workers, err := p.srv.NewPool(context.Background(), &pool.Config{
    NumWorkers:      1,
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
  p.wrkPool = workers

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

    cookies := make(map[string]*httpV1.HeaderValue)
    for _, cookie := range (&http.Request{Header: headers}).Cookies() {
      if value, err := url.QueryUnescape(cookie.Value); err == nil {
        cookies[cookie.Name] = &httpV1.HeaderValue{Value: [][]byte{[]byte(value)}}
      }
    }

    protoHeaders := make(map[string]*httpV1.HeaderValue, len(headers))
    for name, values := range headers {
      header := &httpV1.HeaderValue{}
      for _, value := range values {
        header.Value = append(header.Value, []byte(value))
      }
      protoHeaders[name] = header
    }

    host := headers.Get("Host")
    if host == "" {
      host = request.RequestContext.DomainName
    }
    uri := "https://" + host + request.RawPath
    if request.RawQueryString != "" {
      uri += "?" + request.RawQueryString
    }

    metadata, err := proto.Marshal(&httpV1.Request{
      RemoteAddr: request.RequestContext.HTTP.SourceIP,
      Protocol:   request.RequestContext.HTTP.Protocol,
      Method:     request.RequestContext.HTTP.Method,
      Uri:        uri,
      Header:     protoHeaders,
      Cookies:    cookies,
      RawQuery:   request.RawQueryString,
      Parsed:     false,
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

    var responseMetadata httpV1.Response
    err = proto.Unmarshal(r.Context, &responseMetadata)
    if err != nil || responseMetadata.Status < 100 || responseMetadata.Status >= 600 {
      return events.APIGatewayV2HTTPResponse{Body: "", StatusCode: 500}, nil
    }

    response := events.APIGatewayV2HTTPResponse{
      StatusCode:      int(responseMetadata.Status),
      Headers:         make(map[string]string, len(responseMetadata.Headers)),
      Body:            base64.StdEncoding.EncodeToString(r.Body),
      IsBase64Encoded: true,
    }
    for name, header := range responseMetadata.Headers {
      values := make([]string, 0, len(header.GetValue()))
      for _, value := range header.GetValue() {
        values = append(values, string(value))
      }
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

Use Go `1.27.1`. If your project has no `go.mod`, run `go mod init example.com/lambda` from the project root. Select the dependencies before building:

```bash
go get github.com/aws/aws-lambda-go@v1.55.0 \
  github.com/roadrunner-server/api-go/v6@v6.0.0-beta.14 \
  github.com/roadrunner-server/config/v6@v6.0.0-beta.4 \
  github.com/roadrunner-server/endure/v2@v2.6.2 \
  github.com/roadrunner-server/errors@v1.5.0 \
  github.com/roadrunner-server/goridge/v4@v4.0.0-beta.3 \
  github.com/roadrunner-server/logger/v6@v6.0.0-beta.4 \
  github.com/roadrunner-server/pool/v2@v2.0.0-beta.1 \
  github.com/roadrunner-server/server/v6@v6.0.0-beta.7 \
  google.golang.org/protobuf@v1.36.12
```

The build uses `-mod=readonly` because Composer's `vendor` directory does not contain Go modules. AWS Lambda requires an executable named [`bootstrap`](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html#runtimes-custom-bootstrap) at the root of the deployment package. Include a Linux PHP `8.5` executable named `php` and its shared libraries in `lib/`, built for Amazon Linux 2023 and `x86_64`. Run these commands from the project root:

{% code title="build.sh" %}

```bash
go mod tidy
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -mod=readonly -trimpath -ldflags "-s" -o bootstrap main.go plugin.go
zip -r main.zip bootstrap php lib handler.php vendor
```

{% endcode %}

Use the Lambda `provided.al2023` runtime and `x86_64` architecture. Upload the package and connect it to an API Gateway HTTP API with payload format `2.0`. For a direct Lambda test, use an API Gateway v2 HTTP request event, not a string event. API Gateway v2 omits the custom-domain API mapping prefix from `rawPath`; this adapter uses the path supplied in the event.

## Repository with the full example

- [link](https://github.com/roadrunner-server/aws-lambda)

## Notes

There are multiple notes to acknowledge:

- Start with one worker per Lambda function to control your memory usage.
- Make sure to include the environment variables listed in the code to properly resolve the location of the PHP binary and its dependencies.
