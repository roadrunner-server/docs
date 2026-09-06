# HTTP — Serving static content

The `static` HTTP middleware serves static content using RoadRunner on the main HTTP plugin endpoint.

To avoid a filesystem lookup on every request, the middleware caches file metadata and misses in memory with a short TTL (`10s` by default). A repeated request for the same path is then answered without touching the filesystem, so the overhead of enabling the middleware stays small. The cache trades a few seconds of staleness for that speed: a file added, changed, or removed on disk is picked up within one TTL. Set the TTLs to `0s` to check the filesystem on every request, or run `rr reset static` to flush the cache at once (for example, after a deploy).

{% hint style="info" %}
If there is no file to serve, RR forwards the request back to the PHP worker. Only `GET` and `HEAD` requests are served; every other method goes to the worker.
{% endhint %}

## Enable HTTP middleware

To enable static content serving, use the configuration inside the HTTP section:

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  # host and port separated by semicolon
  address: 127.0.0.1:44933
  middleware: [ "static" ] # Add static to the list of middleware
  static:
    dir: "."
    forbid: [ "" ]
    calculate_etag: false
    weak: false
    allow: [ ".txt", ".php" ]
    # Serve only request paths that start with one of these prefixes.
    # Empty (the default) considers every request.
    prefixes: [ "/assets/", "/build/" ]
    # File metadata cache TTL. "0s" disables it. Default: 10s.
    cache_ttl: 10s
    # Miss cache TTL. "0s" disables it. Default: 10s.
    cache_miss_ttl: 10s
    # Maximum entries kept in each cache map. Default: 16384.
    cache_max_entries: 16384
    request:
      input: "custom-header"
    response:
      output: "output-header"
```

{% endcode %}

Where:

1. `dir`: path to the directory.
2. `forbid`: file extensions that should not be served.
3. `allow`: extensions that should be served (empty = serve all except forbidden). If an extension is present in both lists (allow and forbid), it is treated as forbidden.
4. `calculate_etag`: enable etag calculation for the static file.
5. `weak`: use a weak generator (`W/`). The weak etag is derived from the file size and modification time. If false, the whole file content is used to produce a strong CRC32 etag.
6. `prefixes`: restrict serving to request paths that start with one of these prefixes. When empty (the default), every path is considered. A path that matches no prefix goes to the worker without a filesystem lookup. The match is a plain prefix, so include the trailing slash (`/assets/`) to scope it to a directory.
7. `cache_ttl`: TTL for the file metadata cache (etag, content type, size, modification time). Active only when `calculate_etag` is enabled. `0s` disables it. Default: `10s`.
8. `cache_miss_ttl`: TTL for the miss cache. A request whose path does not resolve to a file records a miss and skips the filesystem check for this duration, which removes the per-request lookup on dynamic routes that share the URL space with static files. `0s` disables it. Default: `10s`.
9. `cache_max_entries`: upper bound on the number of entries in each cache map (metadata and miss). When a map is full, expired entries are evicted; if none can be freed, the middleware checks the filesystem instead. Default: `16384`.
10. `request/response`: custom headers for the static files.

To combine static content with other middleware, use the following sequence (static last, then headers and gzip):

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  # host and port separated by semicolon
  address: 127.0.0.1:44933
  middleware: [ "static", "headers", "gzip" ]
  # Settings for "headers" middleware.
  headers:
    cors:
      allowed_origin: "*"
      allowed_headers: "*"
      allowed_methods: "GET,POST,PUT,DELETE"
      allow_credentials: true
      exposed_headers: "Cache-Control,Content-Language,Content-Type,Expires,Last-Modified,Pragma"
      max_age: 600
  # Settings for "static" middleware.
  static:
    dir: "."
    forbid: [ "" ]
    calculate_etag: false
    weak: false
    allow: [ ".txt", ".php" ]
    request:
      input: "custom-header"
    response:
      output: "output-header"
```

{% endcode %}

## Fileserver plugin

The Fileserver plugin serves static files. It works similarly to the `static` HTTP middleware and has extended functionality.
The `static` middleware runs on the main HTTP endpoint, while the file server plugin uses a different port and serves only static files.

## File server configuration

{% code title=".rr.yaml" %}

```yaml
fileserver:
  # File server address
  #
  # Error on empty
  address: 127.0.0.1:10101
  # Etag calculation. Request body CRC32.
  #
  # Default: false
  calculate_etag: true

  # Weak etag calculation
  #
  # Default: false
  weak: false

  # Enable body streaming for files more than 4KB
  #
  # Default: false
  stream_request_body: true

  serve:
    # HTTP prefix
    #
    # Error on empty
    - prefix: "/foo"

      # Directory to serve
      #
      # Default: "."
      root: "../../../tests"

      # When set to true, the server tries minimizing CPU usage by caching compressed files
      #
      # Default: false
      compress: false

      # Expiration duration for inactive file handlers. Units: seconds.
      #
      # Default: 10, use a negative value to disable it.
      cache_duration: 10

      # The value for the Cache-Control HTTP-header. Units: seconds
      #
      # Default: 10 seconds
      max_age: 10

      # Enable range requests
      # https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests
      #
      # Default: false
      bytes_range: true

    - prefix: "/foo/bar"
      root: "../../../tests"
      compress: false
      cache_duration: 10
      max_age: 10
      bytes_range: true
```

{% endcode %}
