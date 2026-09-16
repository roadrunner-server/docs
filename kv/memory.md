# Memory Driver

This type of driver is already supported by the RoadRunner and does not require any additional installations.

{% hint style="warning" %}
Restarting RoadRunner removes all data from this storage. For persistent values, see [BoltDB](./boltdb.md#persistence), which does not persist expiration metadata.
{% endhint %}

## Configuration

The complete memory driver configuration:

{% code title=".rr.yaml" %}

```yaml
version: "3"

kv:
  # User defined name of the storage.
  memory:
    # Required section.
    # Should be "memory" for the memory driver.
    driver: memory
    config: {}
```

{% endcode %}

There are no additional configuration options for this driver. The `in-memory` driver will automatically create callbacks for items with TTL.

## Expiration

The `kv.MExpire` RPC method takes an absolute RFC 3339 `timeout` timestamp.

{% hint style="warning" %}
In the memory driver at `v6.0.0-beta.5`, `kv.MExpire` truncates the remaining time to whole seconds. A result of zero or less includes past deadlines and deadlines less than one second ahead when RoadRunner handles the request. In this case, the driver replaces the entry with the request's value and removes its TTL. A request with only a key and timeout can replace an existing value with empty bytes. This is not immediate expiration.
{% endhint %}

Use `kv.Delete` for immediate removal. Do not use past or subsecond deadlines with `kv.MExpire`.
