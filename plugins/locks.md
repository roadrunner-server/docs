# Lock

The RoadRunner lock plugin is a powerful tool that enables you to manage resource locks in their applications using the 
RPC protocol. By leveraging the benefits of using GO with PHP, it provides a lightweight, fast, and reliable way to
acquire, release, and manage locks. With this plugin, you can easily manage critical sections of your application and 
prevent race conditions, data corruption, and other synchronization issues that can occur in multiprocess environments.

{% hint style="info" %}
The plugin has two lock backends. The in-memory backend is the default, and it keeps the lock state in one RoadRunner instance. The Redis backend shares locks between RoadRunner instances.
{% endhint %}

## Configuration

Omit the `lock` section or set `driver: memory` to use the in-memory backend. Each RoadRunner instance then keeps its own lock state.

Set `driver: redis` to share locks between RoadRunner instances:

{% code title=".rr.yaml" %}

```yaml
lock:
  driver: redis
  config:
    addrs: ["127.0.0.1:6379"]
    username: ""
    password: ""
    db: 0
    master_name: ""
    sentinel_password: ""
    pool_size: 0
    dial_timeout: 5s
    read_timeout: 5s
    write_timeout: 5s
    tls:
      root_ca: ""
      cert: ""
      key: ""
```

{% endcode %}

The Redis backend requires Redis 7 or later. It uses the `go-redis` v9 client.

A set `master_name` selects a failover client for any number of addresses. Without `master_name`, one address selects a standalone client, and two or more addresses select a cluster client. The `db` setting applies to a standalone client and to a failover client. A negative `db` is rejected at startup. A cluster client uses database 0 only. Without `master_name`, a non-zero `db` with more than one address is rejected at startup.

Set `master_name` to use Redis Sentinel. The `addrs` list then holds the Sentinel addresses. Set `sentinel_password` when the Sentinel nodes need their own password.

Set `pool_size` to size the connection pool for one Redis node. The client opens more connections when the pool is busy. The value `0` selects the `go-redis` default.

The `dial_timeout`, `read_timeout`, and `write_timeout` settings accept Go durations, such as `5s`. Negative values are rejected at startup. Omitted timeouts use the Redis client defaults.

Add the `tls` block to connect with TLS. Remove the block for a plaintext connection. An empty `root_ca` selects the system root certificates. Set `root_ca` to the PEM file of a private certificate authority. Set `cert` and `key` together to send a client certificate. The backend reads the pair for each handshake, so a renewed certificate needs no restart. The minimum protocol version is TLS 1.2.

Write at least one key in the `tls` block. The configuration reader drops a block that has no keys, and the connection then stays plaintext. Write `root_ca: ""` for a server with a public certificate authority.

The backend does not accept some keys of the [RoadRunner Redis plugin](../kv/redis.md). The `max_retries` key is absent, because a retry after a lost reply reports contention for a lock that the caller now holds. The `route_by_latency`, `route_randomly`, and `read_only` keys are absent, because the lock script writes and must run on the master. The `min_retry_backoff`, `max_retry_backoff`, `min_idle_conns`, `max_conn_age`, `pool_timeout`, `idle_timeout`, and `idle_check_freq` keys are absent as well.

The `lock` section requires `driver: memory` or `driver: redis`. Invalid configuration and connection failures stop plugin initialization.

The backend stores lock state under the fixed `rr:lock:` key prefix. The resource name is the namespace. Give resources unique names when different applications share one Redis server.

### Redis lock behavior

- `ForceRelease` removes all locks on the resource. It returns `Ok: true` only if it removed at least one lock.
- Both backends refuse a second read lock with the same ID on the same resource. Use `UpdateTTL` to extend a held lock.
- Contention, or an expired wait during an acquisition, returns `Ok: false`.
- A failed Redis command, or a deadline that occurs during a Redis command, returns an RPC error. The lock state is then unknown. Call `Exists` or `Release` to find the state of the lock.
- A `wait` of `0` on Redis makes one acquisition attempt, bounded by `read_timeout`, which is 5 seconds by default. The memory backend waits 1 millisecond instead.
- Stored locks stay in Redis after a RoadRunner restart until release or expiry.

## PHP client

The RoadRunner lock plugin comes with a convenient PHP package that simplifies the process of integrating the
plugin with your PHP application.

### Installation

To get started, you can install the package via Composer using the following command:

{% code %}

```bash
composer require roadrunner-php/lock
```

{% endcode %}

### Usage

After the installation, you can create an instance of the `RoadRunner\Lock\Lock` class, which will allow you to use the
available class methods.

**Here is an example:**

{% code title="app.php" %}

```php
use RoadRunner\Lock\Lock;
use Spiral\Goridge\RPC\RPC;

require __DIR__ . '/vendor/autoload.php';

$lock = new Lock(RPC::create('tcp://127.0.0.1:6001'));
```

{% endcode %}

{% hint style="warning" %}
To interact with the RoadRunner lock plugin, you will need to have the RPC defined in the rpc configuration
section. You can refer to the documentation page [here](../php/rpc.md) to learn more about the configuration and
installation.
{% endhint %}

The `RoadRunner\Lock\Lock` class provides four methods that allow you to manage locks:

#### Acquire lock

Attempts to acquire an exclusive lock on a resource. Set a positive `waitTTL` to wait for a conflicting lock to be released. The method returns a lock ID on success or `false` on failure. Check the result before accessing the protected resource.

The PHP SDK uses seconds for numeric `ttl` and `waitTTL` values. It converts these values to microseconds for RPC. In PHP SDK 1.0.x, `waitTTL` defaults to `0`, which sends a zero RPC wait. The server then uses a one-millisecond acquisition window. Set `waitTTL` explicitly when you need a longer wait.

{% code title="app.php" %}

```php
$id = $lock->lock('pdf:create', ttl: 10, waitTTL: 5);
if ($id === false) {
    throw new \RuntimeException('Could not acquire pdf:create.');
}

try {
    // Access the protected resource here.
} finally {
    $lock->release('pdf:create', $id);
}
```

{% endcode %}

These calls show alternative arguments. Use the same success check and release handling for each call:

{% code title="app.php" %}

```php
// Set a ten-second TTL.
$id = $lock->lock('pdf:create', ttl: 10);
// or
$id = $lock->lock('pdf:create', ttl: new \DateInterval('PT10S'));

// Wait for at most five seconds to acquire the lock.
$id = $lock->lock('pdf:create', waitTTL: 5);
// or
$id = $lock->lock('pdf:create', waitTTL: new \DateInterval('PT5S'));

// Acquire lock with id - 14e1b600-9e97-11d8-9f32-f2801f1b9fd1
$id = $lock->lock('pdf:create', id: '14e1b600-9e97-11d8-9f32-f2801f1b9fd1');
```

{% endcode %}

#### Acquire read lock

Attempts to acquire a shared lock on a resource. Multiple readers can hold shared locks. An exclusive acquisition must wait until all readers release their locks or its wait expires. Set a positive `waitTTL` to wait for an existing exclusive lock. `lockRead()` returns a lock ID on success or `false` on failure. The PHP time units and default wait are the same as for `lock()`.

{% code title="app.php" %}

```php
$id = $lock->lockRead('pdf:create', ttl: 10, waitTTL: 5);
if ($id === false) {
    throw new \RuntimeException('Could not acquire a read lock on pdf:create.');
}

try {
    // Read the protected resource here.
} finally {
    $lock->release('pdf:create', $id);
}
```

{% endcode %}

These calls show alternative arguments. Check each result before reading the resource:

{% code title="app.php" %}

```php
// Set a ten-second TTL.
$id = $lock->lockRead('pdf:create', ttl: 10);
// or
$id = $lock->lockRead('pdf:create', ttl: new \DateInterval('PT10S'));

// Wait for at most five seconds to acquire the read lock.
$id = $lock->lockRead('pdf:create', waitTTL: 5);
// or
$id = $lock->lockRead('pdf:create', waitTTL: new \DateInterval('PT5S'));

// Acquire lock with id - 14e1b600-9e97-11d8-9f32-f2801f1b9fd1
$id = $lock->lockRead('pdf:create', id: '14e1b600-9e97-11d8-9f32-f2801f1b9fd1');
```

{% endcode %}

#### Release lock

Releases an exclusive lock or read lock on a resource that was previously acquired by a call to `lock()`
or `lockRead()`.

{% code title="app.php" %}

```php
// Release lock after task is done.
$lock->release('pdf:create', $id);

// Force release lock
$lock->forceRelease('pdf:create');
```

{% endcode %}

#### Check lock

Checks if a resource is currently locked and returns information about the lock.

{% code title="app.php" %}

```php
$status = $lock->exists('pdf:create');
if($status) {
    // Lock exists
} else {
    // Lock not exists
}
```

{% endcode %}

#### Update TTL

Replaces the remaining TTL with a new duration from the time the server applies the update. It does not add time to the previous expiry. Numeric PHP values use seconds. A shorter value can release the lock before the protected operation finishes. Check the update result. Stop accessing the resource if the update fails. Finish the operation or renew the lock before its TTL expires.

{% code title="app.php" %}

```php
// Set the remaining TTL to ten seconds.
if (!$lock->updateTTL('pdf:create', $id, 10)) {
    throw new \RuntimeException('Could not renew pdf:create.');
}

// Alternative: use a DateInterval for the same duration.
if (!$lock->updateTTL('pdf:create', $id, new \DateInterval('PT10S'))) {
    throw new \RuntimeException('Could not renew pdf:create.');
}
```

{% endcode %}

## Symfony integration

### Installation

You can install the package via composer:

{% code %}

```bash
composer require roadrunner-php/symfony-lock-driver
```

{% endcode %}

### Usage

{% code title="app.php" %}

```php
use RoadRunner\Lock\Lock;
use Spiral\Goridge\RPC\RPC;
use Spiral\RoadRunner\Symfony\Lock\RoadRunnerStore;
use Symfony\Component\Lock\LockFactory;

require __DIR__ . '/vendor/autoload.php';

$lock = new Lock(RPC::create('tcp://127.0.0.1:6001'));
$factory = new LockFactory(
    new RoadRunnerStore($lock)
);
```

{% endcode %}

Read more about using a Symfony Lock component [here](https://symfony.com/doc/current/components/lock.html).

## API

### Protobuf API

To make it easy to use the Lock proto API in PHP, we provide
a [GitHub repository](https://github.com/roadrunner-php/roadrunner-api-dto), that contains all the generated
PHP DTO classes proto files, making it easy to work with these files in your PHP application.

- [Lock protobuf API](https://github.com/roadrunner-server/api/blob/25217e9/roadrunner/api/lock/v1/lock.proto)

### RPC API

RoadRunner provides an RPC API, which allows you to manage locks in your applications using remote procedure calls. The 
RPC API provides a set of methods that map to the available methods of the `RoadRunner\Lock\Lock` class in PHP.

Raw RPC requests use microseconds for `Request.ttl` and `Request.wait`. For example, `wait: 5000000` allows an acquisition wait of five seconds. An omitted or zero `wait` is not an unlimited wait. The memory backend then uses a one-millisecond acquisition window, and the Redis backend makes one acquisition attempt, bounded by `read_timeout`. PHP SDK arguments use seconds and are converted before the RPC call.

For `Lock` and `LockRead`, require `Response.Ok == true` before accessing the resource. A completed RPC call with no error can still return `Ok == false`, including when the acquisition wait expires.

### Lock

Attempts to acquire an exclusive lock. If the resource has a conflicting lock, the call waits until that lock is released or the acquisition deadline expires. Check `Response.Ok` before entering the protected section.

{% code %}

```go
func (r *rpc) Lock(req *lockApi.Request, resp *lockApi.Response) error {}
```

{% endcode %}

### LockRead

Attempts to acquire a shared lock. Readers can share the resource, but an existing exclusive lock prevents acquisition. The call waits until the conflict ends or the acquisition deadline expires. Check `Response.Ok` before reading the protected resource. An exclusive acquisition also has a finite wait when readers hold the resource.

{% code %}

```go
func (r *rpc) LockRead(req *lockApi.Request, resp *lockApi.Response) error {}
```

{% endcode %}

#### Release

Releases an exclusive lock or a read lock on a resource that was previously acquired by a call to `Lock` or `LockRead`.

{% code %}

```go
func (r *rpc) Release(req *lockApi.Request, resp *lockApi.Response) error {}
```

{% endcode %}

#### ForceRelease

Releases all locks on a resource, regardless of which process acquired them.

{% code %}

```go
func (r *rpc) ForceRelease(req *lockApi.Request, resp *lockApi.Response) error {}
```

{% endcode %}

#### Exists

Checks if a resource is currently locked and returns information about the lock.

{% code %}

```go
func (r *rpc) Exists(req *lockApi.Request, resp *lockApi.Response) error {}
```

{% endcode %}

#### UpdateTTL

Replaces the remaining TTL with `Request.ttl` microseconds from the time the server applies the update. It does not add to the previous expiry. Check `Response.Ok` to detect a failed update.

{% code %}

```go
func (r *rpc) UpdateTTL(req *lockApi.Request, resp *lockApi.Response) error {}
```

{% endcode %}
