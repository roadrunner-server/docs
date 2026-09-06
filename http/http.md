# HTTP Plugin

HTTP plugin is used to pass `HTTP`/`HTTPS`/`fCGI`/`HTTP2(h2c)`/`HTTP3` requests to the PHP worker.

## Configuration reference

{% code title=".rr.yaml" %}

```yaml .rr.yaml
version: "3"

# HTTP plugin settings.
http:
  # Host and port to listen on (e.g.: `127.0.0.1:8080`).
  #
  # Required for plain HTTP. Omit to use only HTTPS or FastCGI.
  address: 127.0.0.1:8080

  # Override HTTP error code for internal RR errors
  #
  # Default: 500
  internal_error_code: 505

  # HTTP access logs
  #
  # Default: false
  access_logs: false

  # Maximum incoming request size in MiB. Zero selects the default limit.
  #
  # Default: 1000
  max_request_size: 256

  # Send raw body (unescaped) to the PHP worker for the application/x-www-form-urlencoded content type
  #
  # Optional, default: false
  raw_body: false

  # Middleware names depend on the plugins in the build. Requests run left to right in v6.
  # The "zstd" middleware requires a build that includes the zstd plugin.
  #
  # Default value: []
  middleware: [ "headers", "gzip" ]

  # Trust HTTP forwarding headers from these proxy addresses.
  # Requires "proxy_ip_parser" in middleware. This is not a network access filter.
  #
  # Default: [] (forwarding headers are not trusted)
  trusted_subnets: [ "127.0.0.1/32" ]

  # File uploading settings.
  uploads:
    # Directory for file uploads. Empty value means to use $TEMP based on your OS.
    #
    # Default: ""
    dir: "/tmp"

    # Deny files with the following extensions to upload.
    #
    # Default: [".php", ".exe", ".bat"]
    forbid: [ ".php", ".exe", ".bat", ".sh" ]

    # [SINCE 2.6] Allow files with the following extensions to upload
    #
    # Default: empty
    allow: [ ".html", ".aaa" ]

  # Settings for "headers" middleware.
  headers:
    # Allows to control CORS headers. Additional headers "Vary: Origin", "Vary: Access-Control-Request-Method",
    # "Vary: Access-Control-Request-Headers" will be added to the server responses. Drop this section for this
    # feature disabling.
    cors:
      # Controls "Access-Control-Allow-Origin" header value (docs: https://mzl.la/2OgD4Qf).
      #
      # Default: ""
      allowed_origin: "*"

      # Controls "Access-Control-Allow-Headers" header value (docs: https://mzl.la/2OzDVvk).
      #
      # Default: ""
      allowed_headers: "*"

      # Controls "Access-Control-Allow-Methods" header value (docs: https://mzl.la/3lbwyXf).
      #
      # Default: ""
      allowed_methods: "GET,POST,PUT,DELETE"

      # Controls "Access-Control-Allow-Credentials" header value (docs: https://mzl.la/3ekJGaY).
      #
      # Default: false
      allow_credentials: true

      # Controls "Access-Control-Expose-Headers" header value (docs: https://mzl.la/3qAqgkF).
      #
      # Default: ""
      exposed_headers: "Cache-Control,Content-Language,Content-Type,Expires,Last-Modified,Pragma"

      # Controls "Access-Control-Max-Age" header value in seconds (docs: https://mzl.la/2PCSdvt).
      #
      # Default: 0
      max_age: 600

    # Automatically add headers to every request passed to PHP.
    #
    # Default: <empty map>
    request:
      input: "custom-header"

    # Automatically add headers to every response.
    #
    # Default: <empty map>
    response:
      X-Powered-By: "RoadRunner"

  # Settings for "static" middleware.
  static:
    # Existing directory to serve.
    #
    # Required when static middleware is enabled.
    dir: "."

    # File extensions to forbid
    #
    # Default: empty
    forbid: [ ".php", ".htaccess" ]

    # ETag calculation (based on the body CRC32)
    #
    # Default: false
    calculate_etag: false

    # Weak ETags use the file name in the pinned static beta.
    #
    # Default: false
    weak: false

    # File extensions to allow
    #
    # Default: empty
    allow: [ ".txt", ".css", ".js" ]

    # Request headers
    #
    # Default: empty
    request:
      input: "custom-header"

    # Response headers
    #
    # Default: empty
    response:
      output: "output-header"

  # Workers pool settings.
  pool:
    # Debug mode for the pool. In this mode, the pool will not pre-allocate the worker. A worker (only 1; num_workers ignored) will be allocated right after a request arrives.
    #
    # Default: false
    debug: false

    # Override server's command
    #
    # Default: empty
    command: "php my-super-app.php"

    # How many worker processes will be started. Zero (or nothing) means the number of logical CPUs.
    #
    # Default: 0
    num_workers: 0

    # Maximal count of worker executions. Zero (or nothing) means no limit.
    #
    # Default: 0
    max_jobs: 0

    # [2023.3.10]
    # Maximum size of the internal requests queue. After reaching the limit, all additional requests would be rejected with error.
    #
    # Default: 0 (no limit)
    max_queue_size: 0

    # Timeout for worker allocation. Zero means 60s.
    #
    # Default: 60s
    allocate_timeout: 60s

    # Timeout for the reset operation. Zero means 60s.
    #
    # Default: 60s
    reset_timeout: 60s

    # Timeout for worker destroying before process killing. Zero means 60s.
    #
    # Default: 60s
    destroy_timeout: 60s

    # Timeout for streaming responses. Zero means 60s.
    #
    # Default: 60s
    stream_timeout: 60s

    # Supervisor is used to control HTTP workers (previous name was "limit", video: https://www.youtube.com/watch?v=NdrlZhyFqyQ).
    # "Soft" limits will not interrupt current request processing. "Hard"
    # limit on the contrary - interrupts the execution of the request.
    supervisor:
      # How often to check the state of the workers.
      #
      # Default: 5s
      watch_tick: 5s

      # Maximum time worker is allowed to live (soft limit). Zero means no limit.
      #
      # Default: 0s
      ttl: 0s

      # How long worker can spend in IDLE mode after first using (soft limit). Zero means no limit.
      #
      # Default: 0s
      idle_ttl: 10s

      # Maximal worker memory usage in megabytes (soft limit). Zero means no limit.
      #
      # Default: 0
      max_worker_memory: 128

      # Maximal job lifetime (hard limit). Zero means no limit.
      #
      # Default: 0s
      exec_ttl: 60s

  # SSL/TLS settings.
  ssl:
    # Host and port to listen on (e.g.: `127.0.0.1:443`).
    #
    # Default: "127.0.0.1:443"
    address: "127.0.0.1:443"

    # Use ACME certificates provider (Let's encrypt)
    acme:
      # Directory to use as a certificate/pk, account info storage
      #
      # Optional. Default: rr_cache
      cache_dir: rr_le_certs

      # User email
      #
      # Used to create LE account. Mandatory. Error on empty.
      email: you-email-here@email

      # Alternate port for the HTTP challenge. Challenge traffic should be redirected to this port if overridden.
      #
      # Optional. Default: 80
      alt_http_port: 80


      # Alternate port for the tls-alpn-01 challenge. Challenge traffic should be redirected to this port if overridden.
      #
      # Optional. Default: 443.
      alt_tlsalpn_port: 443

      # Challenge types
      #
      # Optional. Default: http-01. Possible values: http-01, tlsalpn-01
      challenge_type: http-01

      # Use production or staging endpoint. NOTE: try to use the staging endpoint to make sure that everything works correctly.
      #
      # Optional, but for production should be set to true. Default: false
      use_production_endpoint: true

      # List of your domains to obtain certificates
      #
      # Mandatory. Error on empty.
      domains: [
        "your-cool-domain.here",
        "your-second-domain.here"
      ]

    # Automatic redirect from http:// to https:// schema.
    #
    # Default: false
    redirect: true

    # Path to the cert file. This option is required for SSL working.
    #
    # This option is required.
    cert: /ssl/server.crt

    # Path to the cert key file.
    #
    # This option is required.
    key: /ssl/server.key

    # Path to the root certificate authority file.
    #
    # This option is optional (required for the mTLS).
    root_ca: /ssl/root.crt

    # Client auth type (mTLS)
    #
    # This option is optional. Default value: no_client_certs. Possible values: request_client_cert, require_any_client_cert, verify_client_cert_if_given, require_and_verify_client_cert, no_client_certs
    client_auth_type: no_client_certs

  # FastCGI frontend support.
  fcgi:
    # FastCGI connection DSN. Supported TCP and Unix sockets. An empty value disables this.
    #
    # Default: ""
    address: tcp://0.0.0.0:7921

  # HTTP/2 settings.
  http2:
    # HTTP/2 over non-encrypted TCP connection using H2C.
    #
    # Default: false
    h2c: false

    # Maximal concurrent streams count.
    #
    # Default: 128
    max_concurrent_streams: 128
```

{% endcode %}

## HTTPS

You can enable HTTPS support by adding the `ssl` section to the `http` config.

Use brackets around IPv6 addresses, such as `"[::1]:8443"`. The port must be an unsigned integer from 0 through 65535. HTTP-to-HTTPS redirects preserve IPv6 brackets.

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  address: 127.0.0.1:8080

  ssl:
    # Host and port separated by a colon.
    address: :8892
    redirect: false
    cert: fixtures/server.crt
    key: fixtures/server.key
    root_ca: root.crt
```

{% endcode %}

### Let's Encrypt

RR can automatically obtain TLS certificates for your domain. The folder with your certs might be moved between servers,
RR will check the `cache_dir` and obtain a new certificate if the old one is about to expire.

RR will track your certificate's expiration date and refresh it automatically.

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  # other HTTP sections are omitted 
  # .......

  ssl:
    address: '0.0.0.0:443'
    # ACME section
    #
    # TLS provider
    acme:
      # directory to store your certificate and key from the LE
      #
      # Default: rr_cache_dir
      cache_dir: rr_le_certs
      # Your email
      #
      # Mandatory. Error on empty.
      email: you-email-here@email
      # Alternate port for the HTTP challenge (make sure, that you redirected traffic to the specified port from 80)
      #
      # Default: 80
      alt_http_port: 80
      # Alternate port for the TLS-ALPN challenge (make sure, that you redirected traffic to the specified port from 443)
      #
      # Default: 443
      alt_tlsalpn_port: 443
      # Challenge type to use
      #
      # Default: http-01
      challenge_type: http-01
      # Use staging or production endpoint
      #
      # Would be a good practice to test your setup, before obtaining a real certificate
      use_production_endpoint: false
      # List of your domains
      #
      # Mandatory. Error on empty
      domains:
        - your-cool-domains.here

  # other HTTP sections are omitted
  # ........
```

{% endcode %}

### mTLS

To enable [mTLS](https://www.cloudflare.com/en-gb/learning/access-management/what-is-mutual-tls/) use the following
configuration:

{% code title=".rr.yaml" %}

```yaml
http:
  pool:
    num_workers: 1
    max_jobs: 0
    allocate_timeout: 60s
    destroy_timeout: 60s
  ssl:
    address: :8895
    key: "server-key.pem"
    cert: "server-cert.pem"
    root_ca: "rootCA.pem"
    client_auth_type: require_and_verify_client_cert 
```

{% endcode %}

**Options for the `client_auth_type` are:**

- `request_client_cert`
- `require_any_client_cert`
- `verify_client_cert_if_given`
- `require_and_verify_client_cert`
- `no_client_certs`

### Redirecting HTTP to HTTPS

To enable an automatic redirect from `http://` to `https://` set `redirect` option to `true` (disabled by default).

### Root certificate authority support

Root CA supported by the option in `.rr.yaml`

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  ssl:
    root_ca: root.crt
```

{% endcode %}

## HTTP/2

You can enable HTTP/2 support by adding the `http2` section to the `http` config.

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  address: 127.0.0.1:8080

  http2:
    h2c: false
    max_concurrent_streams: 128
```

{% endcode %}

### HTTP/2 Push Resources

RoadRunner supports [HTTP/2 push](https://en.wikipedia.org/wiki/HTTP/2_Server_Push) via virtual headers provided by the PHP
response.

{% code title="script.php" %}

```php
return $response->withAddedHeader('http2-push', '/test.js');
```

{% endcode %}

Note that the path of the resource must be related to the public application directory and must include `/` at the
beginning.

{% hint style="info" %}
HTTP/2 push only works under HTTPS with the `static` service enabled.
{% endhint %}

### H2C

H2C provides HTTP/2 over an unencrypted TCP connection. In v6 beta, the client must start with HTTP/2 prior knowledge. HTTP/1.1 requests with `Upgrade: h2c` are handled as HTTP/1.1; RR does not upgrade them.

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  address: 127.0.0.1:8080
  http2:
    h2c: true
```

{% endcode %}

For example, use a curl build with HTTP/2 support:

{% code %}

```bash
curl --http2-prior-knowledge http://127.0.0.1:8080/
```

{% endcode %}

## FastCGI

FastCGI frontend support is available inside the HTTP module; you can enable it (disabled by default):

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  fcgi:
    # FastCGI connection DSN. Supported TCP and Unix sockets.
    address: tcp://0.0.0.0:6920
```

{% endcode %}

## Development: PROXY protocol

{% hint style="warning" %}
This section requires a custom build that includes the untagged HTTP change [8bebd3b](https://github.com/roadrunner-server/http/commit/8bebd3b). The currently pinned HTTP plugin, `v6.0.0-beta.10`, does not include PROXY protocol support.
{% endhint %}

PROXY protocol v1 and v2 let a TCP proxy supply client addresses. Configure each application listener separately. `http.proxy_protocol` controls plain HTTP, including H2C. `http.ssl.proxy_protocol` controls HTTPS. Omit the corresponding section to disable it.

The following example enables PROXY protocol on both listeners:

{% code title=".rr.yaml" %}

```yaml
http:
  address: "0.0.0.0:8080"
  proxy_protocol:
    trusted_proxies: [ "10.20.0.10" ]
    read_header_timeout: 5s
  ssl:
    address: "0.0.0.0:8443"
    cert: "server.crt"
    key: "server.key"
    proxy_protocol:
      trusted_proxies: [ "10.20.0.10" ]
      read_header_timeout: 5s
```

{% endcode %}

- `trusted_proxies` must contain at least one IP address or CIDR for an immediate TCP peer. Hostnames are not accepted. Replace the example address with the address of your proxy.
- RR rejects peers outside this list. Trusted peers must send a valid PROXY header. Direct HTTP requests without that header are rejected.
- `read_header_timeout` limits the time to read the PROXY header. The default is `5s`. Zero selects the default; negative values are invalid. This setting does not control HTTP or TLS timeouts.
- For HTTPS, the proxy must send the PROXY header before the TLS handshake. The HTTPS listener still requires certificates or ACME configuration. Temporary ACME challenge listeners are not wrapped.
- Only TCP application listeners support this setting. It does not apply to HTTP/3 or FastCGI. Headers with TCP4 or TCP6 addresses replace the connection addresses; v1 `UNKNOWN` and v2 `LOCAL` retain the socket addresses.

This trust list is separate from [HTTP forwarding-header trust](./proxy.md). If `proxy_ip_parser` also runs, it checks `trusted_subnets` against the client address supplied by PROXY protocol, not the original TCP peer.

Route HTTP readiness checks through a trusted proxy that sends a PROXY header. For direct checks, use the status plugin's separate [health and readiness endpoints](../lab/health.md). A successful TCP connection alone does not prove that RR accepted the PROXY header or that a worker is ready.

## HTTP/3

HTTP/3 support is experimental and might change in the future. Docs are available in the [experimental](../experimental/experimental.md) section.

## Overriding HTTP default error code

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  # Override HTTP error code for internal RR errors (default: 500)
  internal_error_code: 505
```

{% endcode %}

The `http.internal_error_code` is used for `SoftJob`, allocation, TTL, network, and similar errors. For example, a load balancer might require a different code, so you may override the default.

In v6 beta, malformed request bodies that RR cannot parse return `400 Bad Request`. Requests that exceed `max_request_size` return `413 Request Entity Too Large`. `internal_error_code` does not override these responses.

{% hint style="warning" %}
With `http.pool.debug: true`, internal error responses can contain HTML-escaped error text. Keep debug mode disabled on public production servers.
{% endhint %}

## Middleware order

In v6 beta, requests enter middleware from left to right in the configuration list. Responses return through the same middleware in reverse order. A middleware can return a response without calling the remaining handlers.

The order was reversed in v5. Reverse an existing list to preserve its v5 behavior.

{% code title="v5 configuration" %}

```yaml
http:
  middleware: [ "static", "headers", "gzip" ]
```

{% endcode %}

{% code title="Equivalent v6 configuration" %}

```yaml
http:
  middleware: [ "gzip", "headers", "static" ]
```

{% endcode %}

In the v6 example, requests enter `gzip`, then `headers`, then `static`. Put `headers` and `gzip` before `static` to apply them to static responses. Middleware can replace headers set by an earlier handler.

## Request queues

RR has an internal queue for requests. The `allocate_timeout` is used to assign a worker to a request.
If your worker runs for 1 minute but `allocate_timeout` is 30 seconds, after this timeout RR will start rejecting the first request in the queue,
then another 30 seconds for the second, and so on.
