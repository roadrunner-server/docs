# HTTP — Cache (RFC7234) middleware

Cache middleware implements http-caching RFC 7234. It's based on the [Souin](https://github.com/darkweak/souin) HTTP
cache library.

Have a look at the [Souin documentation](https://github.com/darkweak/souin) if you need more information.

{% hint style="warning" %}
This is a third party plugin and isn't included by default. See the `Building RoadRunner with Cache` section for more
information.
{% endhint %}

## Features

* [RFC 7234](https://httpwg.org/specs/rfc7234.html) compliant HTTP Cache.
*

Sets [the `Cache-Status` HTTP Response Header](https://httpwg.org/http-extensions/draft-ietf-httpbis-cache-header.html)

* REST API to purge the cache and list stored resources.
* Builtin support for distributed cache.
* Tag-based invalidation.
* Partial GraphQL caching.
* Configure multiple HTTP verbs to cache (especially for GraphQL).
* Builtin timeout.

## Building RoadRunner with Cache

As it's based on the [Souin](https://github.com/darkweak/souin) HTTP cache library we can use directly the roadrunner
middleware implementation. We have to set the `folder` property because this middleware is located in a subdirectory.

{% code title="configuration.toml" %}

```ini
[github]
[github.token]
token = "YOUR_GH_TOKEN"

[github.plugins]
# Use the Souin third-party middleware.
cache = { ref = "master", owner = "darkweak", repository = "souin", folder = "/plugins/roadrunner" }
# others ...

[log]
level = "debug"
mode = "development"
```

{% endcode %}

**Available storages**:  
In-memory/Filesystem

- `nutsdb`
- `badger` (default one)

Distributed

- `etcd`
- `olric`

More info about customizing RR with your own plugins: [link](../customization/plugin.md)

## Configuration

You can set each Souin configuration key under the `http.cache` key. There is a configuration example below.

{% code title=".rr.yaml" %}

```yaml
http:
  # Other http sub keys
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

**Detailled configuration: [link](https://github.com/darkweak/souin)**
