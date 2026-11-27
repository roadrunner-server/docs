# Experimental Features

## Introduction

RoadRunner supports experimental features starting from `v2023.3.4`. Experimental features are not ready for production use.

No bundled plugin uses the `-e` flag in RoadRunner v3.2. Custom plugins can still read the experimental flag from the configuration plugin.

These features were experimental in earlier releases and are stable now:

- HTTP/3 starts without `-e` in v3.2. See [HTTP/3](../http/http.md#http3).
- `envfile` and `include` configuration settings. See [Configuration](../plugins/config.md).

## How to enable experimental features

To enable experimental features, run RR with the `-e` (`--enable-experimental`) flag. For example:

{% code %}

```bash
./rr serve -e
```

{% endcode %}

Or:

{% code %}

```bash
./rr serve --enable-experimental
```

{% endcode %}
