# Experimental Features

## Introduction

Starting with RR `v2023.3.4`, RoadRunner can run with experimental features enabled. These are features that are not yet ready for production use.

In RoadRunner v3.2, the `-e` flag affects the Jobs plugin. If Jobs cannot process a worker response, it normally tries to acknowledge the job. With `-e`, Jobs skips this fallback acknowledgment.

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
