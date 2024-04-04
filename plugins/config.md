# Config

The config plugin is an essential component in RoadRunner, responsible for parsing configuration files and environment
variables. It serves as a central hub for managing configurations for the plugins and RoadRunner server itself.

## Configuration File Structure

The RoadRunner configuration file should be formatted using **YAML** or **JSON**. Each configuration file must include
a `version` at the top, indicating the format's version. The currently supported configuration version is **version 3**.

{% hint style="warning" %}
The configuration version is not synonymous with the RoadRunner (RR) version. The configuration version and RR version
have separate versioning systems.
{% endhint %}

**Example of a YAML configuration file:**

{% code title=".rr.yaml" %}

```yaml
version: '3'

# ... other config values
```

{% endcode %}

{% hint style="warning" %}
Version numbers are strings, not numbers. For example, `version: "3"` is correct, but `version: 3` is not.
{% endhint %}

## Tips

1. By default, `.rr.yaml` used as the configuration, located in the same directory with RR binary.
