# Lock

The RoadRunner lock plugin is a powerful tool that enables you to manage resource locks in their applications using the 
RPC protocol. By leveraging the benefits of using GO with PHP, it provides a lightweight, fast, and reliable way to
acquire, release, and manage locks. With this plugin, you can easily manage critical sections of your application and 
prevent race conditions, data corruption, and other synchronization issues that can occur in multiprocess environments.

{% hint style="warning" %}
RoadRunner lock plugin uses an in-memory storage to store information about locks at this moment. When multiple
instances of RoadRunner are used, each instance will have its own in-memory storage for locks. As a result, if a
process
acquires a lock on one instance of RoadRunner, it will not be aware of the lock state on the other instances.
{% endhint %}

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

Raw RPC requests use microseconds for `Request.ttl` and `Request.wait`. For example, `wait: 5000000` allows an acquisition wait of five seconds. If `wait` is omitted or zero, the server uses a one-millisecond acquisition window, not an unlimited wait. PHP SDK arguments use seconds and are converted before the RPC call.

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
