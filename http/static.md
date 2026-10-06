# HTTP - Serving static content

The `static` HTTP middleware serves static content using RoadRunner on the main HTTP plugin endpoint.

{% hint style="info" %}
The middleware serves files for `GET` and `HEAD` requests. It forwards other requests to PHP. See [cache and prefixes](#cache-and-prefixes) to limit filesystem checks and cache file metadata.
{% endhint %}

## Enable HTTP middleware

To enable static content serving, use the configuration inside the HTTP section:

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  # Host and port separated by a colon.
  address: 127.0.0.1:44933
  middleware: [ "static" ] # Add static to the list of middleware
  static:
    dir: "."
    forbid: [ ".php", ".htaccess" ]
    calculate_etag: false
    weak: false
    allow: [ ".txt", ".css", ".js" ]
    request:
      input: "custom-header"
    response:
      output: "output-header"
```

{% endcode %}

Where:

1. `dir`: required path to an existing directory.
2. `forbid`: file extensions that should not be served.
3. `allow`: extensions that should be served (empty = serve all except forbidden). If an extension is present in both lists (allow and forbid), it is treated as forbidden.
4. `calculate_etag`: enable [ETag calculation](#etags) for the static file.
5. `weak`: use file size and modification time for a weak ETag (`W/`). With `weak: false`, RR calculates a strong CRC32C ETag from the file contents. This setting requires `calculate_etag: true`.
6. `request/response`: custom headers for the static files.

Put `static` after `gzip` and `headers` so they also apply to static responses. See [middleware order](./http.md#middleware-order) when migrating a v5 configuration.

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  # Host and port separated by a colon.
  address: 127.0.0.1:44933
  middleware: [ "gzip", "headers", "static" ]
  # Settings for "headers" middleware.
  headers:
    cors:
      allowed_origin: "https://app.example.com"
      allowed_headers: "*"
      allowed_methods: "GET,POST,PUT,DELETE"
      allow_credentials: true
      exposed_headers: "Cache-Control,Content-Language,Content-Type,Expires,Last-Modified,Pragma"
      max_age: 600
  # Settings for "static" middleware.
  static:
    dir: "."
    forbid: [ ".php", ".htaccess" ]
    calculate_etag: false
    weak: false
    allow: [ ".txt", ".css", ".js" ]
    request:
      input: "custom-header"
    response:
      output: "output-header"
```

{% endcode %}

## Cache and prefixes

The middleware normalizes the URL path before checking prefixes and file extensions. Requests for missing files, directories, or paths without a file extension go to PHP.

{% code title=".rr.yaml" %}

```yaml
http:
  address: 127.0.0.1:44933
  middleware: [ "static" ]
  static:
    dir: "."
    forbid: [ ".php", ".htaccess" ]
    allow: [ ".txt", ".css", ".js" ]
    calculate_etag: true
    weak: false
    prefixes: [ "/assets/", "/build/" ]
    cache_ttl: 10s
    cache_miss_ttl: 10s
    cache_max_entries: 16384
```

{% endcode %}

- `prefixes`: serve only normalized paths that start with a listed prefix. An empty list considers every path. Each prefix must start with `/`. Use a trailing slash, such as `/assets/`, to match a directory. Prefixes are not removed from the file path.
- `cache_ttl`: cache file metadata, including the ETag and content type. This cache is active only when `calculate_etag` is enabled. The default is `10s`. An explicit `0s` disables it.
- `cache_miss_ttl`: cache missing-file and directory results. The default is `10s`. An explicit `0s` disables it.
- `cache_max_entries`: entry limit for each cache. The default is `16384`; zero selects the default. A full cache attempts to remove expired entries. If it cannot free space, RR serves the request without adding a new entry.

Negative TTLs and negative entry limits are invalid.

A positive cache hit still opens the file and reads its metadata. RR reuses cached metadata only when the file size and modification time match. It detects deleted files and changed metadata on the next request. A cached miss avoids the filesystem lookup. RR can continue to send requests for a newly created file to PHP until `cache_miss_ttl` expires.

If a deployment preserves both file size and modification time, RR can reuse an old ETag until `cache_ttl` expires. Run `rr reset static` after such a deployment to clear both caches. This command requires [RPC](../php/rpc.md). Set both TTLs to `0s` to disable caching.

These options were added in [static PR #147](https://github.com/roadrunner-server/static/pull/147).

## ETags

Set `calculate_etag: true` to add ETags. Strong ETags use CRC32C of the file contents. RR does not generate a strong ETag for an empty file or a file larger than 32 MiB. Set `weak: true` to use file size and modification time for a weak ETag.

Clients can send `If-None-Match` with a stored ETag. An unchanged file can return `304 Not Modified` without its body. Store and send ETags as opaque values. Do not calculate them in the client.

## Fileserver plugin

The Fileserver plugin serves static files. It works similarly to the `static` HTTP middleware and has extended functionality.
The `static` middleware runs on the main HTTP endpoint. The Fileserver plugin uses a separate listener and serves only static files.

## File server configuration

Startup fails if `address` is empty or `serve` has no entries. Each `prefix` must be nonempty and start with `/`.

{% code title=".rr.yaml" %}

```yaml
fileserver:
  # File server address
  #
  # Error on empty
  address: 127.0.0.1:10101
  # ETag calculation from the response body.
  #
  # Default: false
  calculate_etag: true

  # Weak etag calculation
  #
  # Default: false
  weak: false

  # Stream incoming request bodies.
  #
  # Default: false
  stream_request_body: true

  serve:
    # HTTP prefix
    #
    # Required. Must start with a forward slash.
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
      # Default: 0 (no Cache-Control header)
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

### Unix Socket

The Fileserver plugin supports [Unix socket attributes](../intro/config.md#unix-socket-attributes). Set optional `mode`, `uid`, and `gid` fields under `fileserver.unix_socket`:

{% code title=".rr.yaml fragment" %}

```yaml
fileserver:
  address: "unix:///run/roadrunner/files.sock"
  unix_socket:
    mode: "0660"
  serve:
    - prefix: "/"
      root: "public"
```

{% endcode %}

These settings apply to the Fileserver listener. See [Fileserver PR #95](https://github.com/roadrunner-server/fileserver/pull/95).
