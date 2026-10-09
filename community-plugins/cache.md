# HTTP — Cache (RFC7234) middleware

Cache middleware implements HTTP caching (RFC 7234). It's based on the [Souin](https://github.com/darkweak/souin) HTTP
cache library.

Have a look at the [Souin documentation](https://github.com/darkweak/souin) if you need more information.

{% hint style="warning" %}
This is a third-party plugin and isn't included by default. See the "Building RoadRunner with Cache" section for more
information.
{% endhint %}

## Features

* Sets [the `Cache-Status` HTTP Response Header](https://httpwg.org/http-extensions/draft-ietf-httpbis-cache-header.html)
* [RFC 7234](https://httpwg.org/specs/rfc7234.html) compliant HTTP Cache.
* REST API to purge the cache and list stored resources.
* Built-in support for distributed cache.
* Tag-based invalidation.
* Partial GraphQL caching.
* Configure multiple HTTP verbs to cache (especially for GraphQL).
* Built-in timeout.

## Building RoadRunner with Cache

RoadRunner v3 requires a Souin revision that implements the [v6 plugin interfaces](../customization/plugin.md#v6-migration), including `NamedLogger(string) *slog.Logger` for logging.

Follow the [Velox build guide](../customization/build.md). Add `[plugins.cache]` with `module_name = "github.com/darkweak/souin/plugins/roadrunner"` and set `tag` to a compatible revision. The module path includes the plugin subdirectory.

**Available storages**:
In-memory/Filesystem

* `nutsdb`
* `badger` (default one)

Distributed

* `etcd`
* `olric`

More info about customizing RR with your own plugins: [link](../customization/plugin.md)

## Configuration

You can set each Souin configuration key under the `http.cache` key. There is a configuration example below.

{% code title=".rr.yaml" %}

```yaml
http:
  # Other HTTP subkeys
  cache:
    api:
      basepath: /httpcache_api
      prometheus:
        basepath: /anything-for-prometheus-metrics
      souin: { }
    default_cache:
      allowed_http_verbs:
        - GET
        - POST
        - HEAD
      cdn:
        api_key: XXXX
        dynamic: true
        hostname: XXXX
        network: XXXX
        provider: fastly
        strategy: soft
      headers:
        - Authorization
      regex:
        exclude: '/excluded'
      timeout:
        backend: 5s
        cache: 1ms
      ttl: 5s
      stale: 10s
    log_level: debug
    ykeys:
      The_First_Test:
        headers:
          Content-Type: '.+'
      The_Second_Test:
        url: 'the/second/.+'
    surrogate_keys:
      The_First_Test:
        headers:
          Content-Type: '.+'
      The_Second_Test:
        url: 'the/second/.+'
  middleware:
    - cache
    # Other middlewares
```

{% endcode %}
