# Memory Driver

This type of driver is already supported by the RoadRunner and does not require any additional installations.

{% hint style="warning" %}
This type of storage, all data is contained in memory and will be destroyed when the RoadRunner Server is restarted.
If you need persistent storage without additional dependencies, then it is recommended to use the boltdb driver.
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