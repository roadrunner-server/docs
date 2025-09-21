# Jobs — AMQP 1.0 Driver

AMQP 1.0 driver for RoadRunner provides unified support for both **Azure Service Bus** and **RabbitMQ** using the pure `github.com/Azure/go-amqp` library. It implements the standardized AMQP 1.0 protocol for better interoperability and broker-agnostic messaging.

{% hint style="warning" %}
This is a third-party plugin and isn't included by default. See the "Building RoadRunner with AMQP1" section for more information.
{% endhint %}

## Features

* **Pure AMQP 1.0 Implementation**: Uses `github.com/Azure/go-amqp` v1.4.0 for standardized protocol support
* **Dual Broker Support**: Works with both Azure Service Bus and RabbitMQ (with AMQP 1.0 plugin)
* **Automatic Broker Detection**: Identifies Azure Service Bus vs RabbitMQ automatically
* **Unified Configuration**: Same configuration format works with both brokers
* **TLS/SSL Support**: Automatic encryption for Azure Service Bus, configurable for RabbitMQ
* **Connection Resilience**: Built-in retry mechanisms and connection management
* **Distributed Tracing**: OpenTelemetry integration for observability
* **Container-based Architecture**: AMQP 1.0 sessions and links for efficient messaging
* **Event-driven Design**: Asynchronous message processing capabilities

## Building RoadRunner with AMQP1

To include the AMQP1 driver in your RoadRunner build, you need to configure it in your velox.toml build configuration.

{% code title="velox.toml" %}

```toml
[roadrunner]
ref = "master"

[debug]
enabled = false

[log]
level = "debug"
mode = "production"

# optional, needed only to download RR once (per version)
[github.token]
token = "${GITHUB_TOKEN}"

# Include the AMQP1 plugin
[plugins.amqp1]
tag = "master"
module_name = "github.com/ammadfa/amqp1"

# Include jobs plugin (required dependency)
[plugins.jobs]
tag = "latest"
module_name = "github.com/roadrunner-server/jobs/v5"

# Other common plugins
[plugins.server]
tag = "latest"
module_name = "github.com/roadrunner-server/server/v5"

[plugins.http]
tag = "latest"
module_name = "github.com/roadrunner-server/http/v5"
```

{% endcode %}

More info about customizing RR with your own plugins: [link](../customization/plugin.md)

## Configuration

### Azure Service Bus Configuration

For Azure Service Bus, use the `amqps://` protocol with your connection string:

{% code title=".rr.yaml" %}

```yaml
amqp1:
  addr: "amqps://RootManageSharedAccessKey:YOUR_ACCESS_KEY@YOUR_NAMESPACE.servicebus.windows.net:5671/"
  container_id: "roadrunner-jobs-azure"

jobs:
  consume: ["azure-queue"]

  pipelines:
    azure-queue:
      driver: amqp1
      config:
        queue: "test-queue"                   # Must exist in Azure Service Bus
        prefetch: 10
        priority: 1
        durable: false
        exclusive: false
```

{% endcode %}

**Azure Service Bus Requirements:**
- Queue must be pre-created in Azure portal or via Azure CLI
- Uses Shared Access Key authentication
- TLS is automatically enabled with `amqps://` protocol
- Routing occurs directly to queue (no exchanges)

### RabbitMQ Configuration

For RabbitMQ, ensure the AMQP 1.0 plugin is enabled:

{% code title=".rr.yaml" %}

```yaml
amqp1:
  addr: "amqp://username:password@rabbitmq:5672/"
  container_id: "roadrunner-jobs-rabbitmq"

jobs:
  consume: ["rabbit-queue"]

  pipelines:
    rabbit-queue:
      driver: amqp1
      config:
        queue: "test-queue"
        routing_key: "test-queue"
        exchange: "test-queue"                # Use default exchange
        exchange_type: "direct"               # informational; configure server-side
        prefetch: 10
        priority: 10
        durable: true
        exclusive: false
```

{% endcode %}

**RabbitMQ Requirements:**
- Enable AMQP 1.0 plugin: `rabbitmq-plugins enable rabbitmq_amqp1_0`
- Queues and exchanges must be created ahead of time
- Supports exchange-based routing; ensure bindings are configured server-side
- For delayed messages, enable: `rabbitmq-plugins enable rabbitmq_delayed_message_exchange`

### TLS Configuration

For secure connections, configure TLS settings:

{% code title=".rr.yaml" %}

```yaml
amqp1:
  addr: "amqps://guest:guest@127.0.0.1:5671/"
  container_id: "roadrunner-secure"
  tls:
    cert: "/path/to/cert.pem"
    key: "/path/to/key.pem"
    root_ca: "/path/to/ca.pem"
    insecure_skip_verify: false
```

{% endcode %}

### Advanced Pipeline Configuration

{% code title=".rr.yaml" %}

```yaml
jobs:
  pipelines:
    # Azure Service Bus optimized pipeline
    priority-orders:
      driver: amqp1
      config:
        queue: "priority-orders"
        prefetch: 50
        priority: 5

    # RabbitMQ with topic exchange
    events-pipeline:
      driver: amqp1
      config:
        queue: "events-queue"
        exchange: "events-exchange"
        exchange_type: "topic"
        routing_key: "events.#"
        prefetch: 25
        priority: 3
        durable: true
        exclusive: false
```

{% endcode %}

## Migration Benefits

The AMQP1 driver provides significant advantages over RabbitMQ-specific implementations:

### From RabbitMQ-specific to Universal
- **Previous**: `github.com/rabbitmq/rabbitmq-amqp-go-client` (RabbitMQ-only)
- **Current**: `github.com/Azure/go-amqp` (Works with any AMQP 1.0 broker)
- **Benefit**: Single codebase supports multiple message brokers

### Protocol Standardization
- **AMQP 1.0 Compliance**: Standardized protocol ensures better interoperability
- **Container Model**: Modern connection architecture with sessions and links
- **Message Format**: Structured AMQP 1.0 messages with application properties

### Cloud-Native Ready
- **Azure Service Bus**: Native support for Microsoft's cloud messaging service
- **Hybrid Deployments**: Use RabbitMQ on-premises and Azure Service Bus in cloud
- **Consistent API**: Same job queue interface regardless of broker

## Implementation Details

### Driver Components

1. **Plugin** (`plugin.go`): Main plugin interface and registration
2. **Driver** (`amqp1jobs/driver.go`): Core driver with pure AMQP 1.0 support
3. **Config** (`amqp1jobs/config.go`): Configuration structure and validation
4. **Item** (`amqp1jobs/item.go`): Message/job item handling and serialization

### Connection Management

The driver uses the AMQP 1.0 container model:

```go
// Create AMQP 1.0 connection
conn, err := amqp.Dial(ctx, addr, &amqp.ConnOptions{
    ContainerID: conf.ContainerID,
    TLSConfig:   tlsConfig,
})

// Session for logical grouping
session, err := conn.NewSession(ctx, nil)

// Receivers and senders for message flow
receiver, err := session.NewReceiver(ctx, queueName, options)
sender, err := session.NewSender(ctx, target, options)
```

### Broker Detection

The driver automatically detects the broker type and adapts message routing:

- **Azure Service Bus**: Direct queue addressing
- **RabbitMQ**: Exchange-based routing with AMQP v2 addressing

### Message Processing

Unified message processing pattern for both brokers:

```go
// Receive messages
msg, err := receiver.Receive(ctx, nil)

// Process job
jobItem := convertFromAMQP1Message(msg)

// Acknowledge based on result
if success {
    receiver.AcceptMessage(ctx, msg)
} else {
    receiver.RejectMessage(ctx, msg, nil)
}
```

## Troubleshooting

### Common Issues

1. **RabbitMQ AMQP 1.0 Plugin**: Ensure `rabbitmq_amqp1_0` plugin is enabled
2. **Azure Service Bus**: Verify access keys and queue existence
3. **TLS Configuration**: Check certificate paths and validity
4. **Queue Declaration**: Queues must be pre-created for AMQP 1.0

### Debugging

Enable debug logging to troubleshoot connection and message issues:

```yaml
log:
  level: "debug"
  mode: "development"
```

### Performance Tuning

- **Prefetch**: Adjust based on message processing speed
- **Priority**: Set appropriate job priority levels
- **Connection Pooling**: Use appropriate container IDs for load distribution