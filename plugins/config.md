# Config

The config plugin reads configuration files and expands environment variables. Other plugins use it to read their settings.

## Configuration File Structure

Use YAML or JSON for the RoadRunner configuration. Set `version: "3"` in each file. This value identifies the configuration format.

{% hint style="warning" %}
The configuration format version is separate from the RoadRunner release number.
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

The file-based configuration loader rejects a missing or non-string version. The legacy value `"2.7"` still loads with an upgrade warning. Each included file must have the same version as the main file.

## Includes and overrides

Use [`include`](../intro/config.md#include-configuration-files) to load additional files. The loader applies CLI `-o` values before includes, so an included section can replace an override. List values are replaced. Includes have no delete operation and do not load nested include lists.

Each include is required. The config plugin does not support optional include objects. Use environment variables with defaults for values that differ between systems.

## Environment Files

The root `envfile` setting loads a file before the config plugin expands environment variables. With config plugin v6, it no longer requires experimental mode. A setting that was ignored without experimental mode in v5 is now active.

Remove an unused `envfile` setting. If you need it, supply the file it names. A missing or unreadable file stops initialization. See [Dotenv](../php/environment.md#dotenv) for configuration examples, relative paths, and environment precedence.

## Tips

1. RoadRunner looks for `.rr.yaml` in the current working directory. Use `-c` to select a file and `-w` to select a working directory.
