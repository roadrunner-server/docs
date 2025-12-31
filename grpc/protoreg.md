# Protoreg Plugin

`protoreg` provides a shared protobuf registry, enabling other plugins to access proto definitions at runtime, making it ideal for building custom gRPC interceptors, middleware, and dynamic message handling.

## Features

- Parses and registers `.proto` files at startup
- Provides a shared `protoregistry` accessible by other plugins
- Supports multiple import paths for proto dependencies
- Automatically resolves and registers proto file dependencies
- Exposes service and method descriptors for reflection-based operations
- Thread-safe registry access

## Use Cases

- **Custom gRPC Interceptors**: Build interceptors that need to inspect or modify messages based on their proto definitions
- **Dynamic Message Handling**: Deserialize and serialize protobuf messages without compile-time generated code
- **Request/Response Validation**: Validate gRPC requests against proto schemas
- **Logging & Monitoring**: Create detailed logs with field-level message inspection

## Configuration

{% hint style="warning" %}
The `protoreg` plugin must be enabled alongside the `grpc` plugin in your configuration. Without the `grpc` plugin enabled, the `protoreg` plugin will not be started.
{% endhint %}

The plugin is configured under the `protoreg` section in your `.rr.yaml` configuration file:

```yaml
version: '3'

protoreg:
  proto_path:
    - proto/commonapis
    - proto/serviceapis
  files:
    - service/v1/service.proto
    - user/v1/user.proto
```

### Configuration Options

#### `proto_path` (required)

A list of directories where the plugin will search for proto files and their dependencies. These paths work [similarly to the `--proto-path`/`-I` flag in `protoc`](https://www.mankier.com/1/protoc#--proto_path).

**Why it's needed**: Proto files often import other proto files (e.g., `import "google/protobuf/timestamp.proto"`). The import paths tell the parser where to find these dependencies. You typically need:

- A path containing your service definitions
- A path containing shared/common message definitions
- Paths to any third-party proto files (like Google's well-known types)

```yaml
proto_path:
  - proto/commonapis     # Shared message definitions
  - proto/serviceapis    # Service-specific proto definitions
  - proto/third_party    # External dependencies (e.g., google/api)
```

#### `files` (required)

A list of proto files to parse and register. Paths are relative to the `proto_path` directories.
This works similarly to passing [variadic proto files argument in `protoc`](https://www.mankier.com/1/protoc#Options-@%3Cfilename%3E).

**Why it's needed**: This explicitly declares which proto files define the services and messages you want to access at runtime. The plugin will:

1. Parse each listed proto file
2. Automatically resolve and register all imported dependencies
3. Build a registry of all services and their methods

```yaml
files:
  - service/v1/service.proto    # Your gRPC service definitions
  - user/v1/user.proto          # Additional service definitions
```

## Example: Project Structure

A typical project structure when using this plugin:

```
my-project/
├── .rr.yaml
├── proto/
│   ├── commonapis/
│   │   └── common/
│   │       └── v1/
│   │           └── message.proto
│   └── serviceapis/
│       └── service/
│           └── v1/
│               └── service.proto
└── ...
```

**proto/commonapis/common/v1/message.proto**:
```protobuf
syntax = "proto3";

package common.v1;

message Request {
  string id = 1;
  string payload = 2;
}

message Response {
  string id = 1;
  string result = 2;
}
```

**proto/serviceapis/service/v1/service.proto**:
```protobuf
syntax = "proto3";

package service.v1;

import "common/v1/message.proto";

service MyService {
  rpc Process (common.v1.Request) returns (common.v1.Response) {}
  rpc Ping (common.v1.Request) returns (common.v1.Response) {}
}
```

**Configuration (.rr.yaml)**:
```yaml
version: '3'

protoreg:
  proto_path:
    - proto/commonapis
    - proto/serviceapis
  files:
    - service/v1/service.proto
```

## Using the Registry in Your Plugin

Other plugins can depend on `protoreg` to access the registry. Here's how to use it in a custom plugin:

```go
package myplugin

import (
	"github.com/jhump/protoreflect/desc"
	"github.com/jhump/protoreflect/v2/protoresolve"
)

type Registry interface {
	Registry() *protoresolve.Registry
	Services() map[string]*desc.ServiceDescriptor
	FindMethodByFullPath(method string) (*desc.MethodDescriptor, error)
}

type Plugin struct {
    registry Registry
}

func (p *Plugin) Init(registry Registry) error {

    p.registry = registry

    // Access the underlying registry
    reg := p.registry.Registry()

    // Get all registered services
    services := p.registry.Services()

    // Find a specific method by its full path
    method, err := p.registry.FindMethodByFullPath("/service.v1.MyService/Process")
    if err != nil {
        return err
    }

    // Use the method descriptor for reflection
    inputType := method.GetInputType()
    outputType := method.GetOutputType()

    return nil
}
```

### Registry Interface

The plugin exposes the following interface:

```go
type Registry interface {
    // Registry returns the underlying protoresolve.Registry that
	// contains all the parsed descriptors
    Registry() *protoresolve.Registry

    // Services returns a map of all registered service descriptors
    // keyed by their fully qualified name
    Services() map[string]*desc.ServiceDescriptor

    // FindMethodByFullPath finds a method descriptor by its full gRPC path
    // Format: "/package.Service/Method" or "package.Service/Method"
    FindMethodByFullPath(method string) (*desc.MethodDescriptor, error)
}
```

## Example: gRPC Interceptor

Here's an example of using the registry in a custom gRPC interceptor to log request details:

```go
package interceptor

import (
    "context"
    "log"

    "github.com/roadrunner-server/protoreg/v5"
    "google.golang.org/grpc"
    "google.golang.org/protobuf/proto"
    "google.golang.org/protobuf/types/dynamicpb"
)

type Plugin struct {
    registry protoreg.Registry
}

func (i *Plugin) UnaryInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    // Find the method descriptor
    method, err := i.registry.FindMethodByFullPath(info.FullMethod)
    if err != nil {
        log.Printf("Method not found in registry: %s", info.FullMethod)
        return handler(ctx, req)
    }

    // Log method information
    log.Printf("Method: %s", method.GetFullyQualifiedName())
    log.Printf("Input type: %s", method.GetInputType().GetFullyQualifiedName())
    log.Printf("Output type: %s", method.GetOutputType().GetFullyQualifiedName())

    // You can also dynamically inspect the message fields
    inputDesc := method.GetInputType()
    for _, field := range inputDesc.GetFields() {
        log.Printf("  Field: %s (type: %s)", field.GetName(), field.GetType().String())
    }

    return handler(ctx, req)
}
```

## Troubleshooting

### "proto path does not exist"

Ensure all paths in `proto_path` exist and are accessible. Paths can be absolute or relative to the RoadRunner working directory.

### "proto file does not exist"

The proto file must exist in one of the configured `proto_path`. Check that:
1. The file path in `files` is relative to an import path, not an absolute path
2. The directory structure matches the import path

### "file already registered"

This occurs when the same proto file is included multiple times (e.g., listed in `files` and also imported by another file). The plugin handles dependencies automatically, so you typically only need to list your top-level service definitions in the `files` section.

## License

MIT License - see LICENSE file for details.
