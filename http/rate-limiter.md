# HTTP - Rate Limiter

The `rate_limiter` middleware limits requests before they reach PHP. It uses one process-local token bucket policy with a global, IP, or header key.

## Availability

{% hint style="info" %}
This middleware is planned for the RoadRunner bundle in the upcoming v3 release with v6 plugins. The pinned [RR source build](../intro/install.md) at `b0cccd917f001b6584eafdc04ad6ba69a97cbb69` does not include it. The implementation is in [rate-limiter PR #1](https://github.com/roadrunner-server/rate-limiter/pull/1).
{% endhint %}

## Configuration

Add `rate_limiter` to `http.middleware`. Define its policy under `http.rate_limiter`:

{% code title=".rr.yaml" %}

```yaml
version: "3"

server:
  command: "php worker.php"
  relay: pipes

http:
  address: "127.0.0.1:8080"
  middleware: ["rate_limiter"]
  rate_limiter:
    key: ip
    rate: 10
    interval: 1s
    burst: 20
    max_entries: 10000
```

{% endcode %}

| Field | Default | Meaning |
| --- | --- | --- |
| `key` | `ip` | One of `global`, `ip`, or `header`. |
| `rate` | Required | Positive integer. Tokens added per `interval`. |
| `interval` | `1s` | Positive Go duration, such as `250ms`, `1s`, or `1m`. |
| `burst` | `1` | Positive integer. Bucket capacity and initial token count. |
| `max_entries` | `10000` | Positive integer. Maximum number of stored buckets. |
| `header` | Empty | A valid HTTP header name is required for `key: header`. It must be empty in other modes. |

Omitted fields use their defaults. An explicit zero or negative value for `rate`, `interval`, `burst`, or `max_entries` is invalid. Invalid configuration fails initialization. An absent `http.rate_limiter` section disables the plugin. A configured policy has no effect unless the middleware list selects it.

## Token Accounting

The middleware uses the [`golang.org/x/time/rate` token bucket](https://pkg.go.dev/golang.org/x/time/rate#Limiter). Each new bucket starts with `burst` tokens. Each allowed request uses one token. Tokens refill continuously at `rate / interval`, up to `burst`. Rejected requests do not reserve future tokens.

For example, `rate: 60`, `interval: 1m`, and `burst: 10` allow 10 requests immediately and add one token per second. This is not a fixed-window limit of 60 requests per minute. The bucket does not reset at minute boundaries, and `burst` can exceed `rate`.

## Client Identity

| Key | Bucket identity |
| --- | --- |
| `global` | All requests share one bucket. |
| `ip` | The normalized IP from `RemoteAddr`, without its port. IPv4-mapped IPv6 addresses share the IPv4 bucket. |
| `header` | The trimmed value of the configured header. Values are case-sensitive. The plugin stores a SHA-256 hash, not the raw value. For `Host`, it reads Go's `Request.Host` field. |

For header mode, set `key: header` and `header: X-Client-ID`. A header value is not proof of identity. For per-user limits, a trusted upstream must supply a stable, verified identity and overwrite client-supplied values. A client that controls the selected value can obtain new buckets and fill the map.

IP mode reads `RemoteAddr`. It does not read `Forwarded` or `X-Forwarded-For` directly. Behind a proxy, configure the [Proxy IP parser](proxy.md) and place it before the limiter. The proxy must overwrite the selected forwarding headers. Without this setup, IP mode can limit the proxy address or use a forged client identity. Use `global` or `header` mode when the transport provides no client IP.

## Responses

| Condition | Response |
| --- | --- |
| A token is available | Call the next handler without changing its response. |
| The selected header is missing or empty after trimming | `400 Bad Request`. |
| `RemoteAddr` has no valid IP in IP mode | `400 Bad Request`. |
| The bucket has no token | `429 Too Many Requests` with `Retry-After`. |
| A new identity arrives when the map is full | `503 Service Unavailable`. Existing identities still use their buckets. |

The middleware sets `Cache-Control: no-store` on its error responses. `Retry-After` gives the delay until the next token, rounded up to whole seconds, with a minimum of `1`. It does not reserve capacity; other requests can consume the token first. Rejected requests do not reach PHP. Rejected HEAD requests have no response body.

See [RFC 6585, section 4](https://www.rfc-editor.org/rfc/rfc6585.html#section-4) for `429` and [RFC 9110, section 10.2.3](https://www.rfc-editor.org/rfc/rfc9110.html#section-10.2.3) for `Retry-After`.

## State And Cleanup

All HTTP listeners and PHP workers in one RoadRunner process share the policy state. Separate RoadRunner processes have independent quotas. A process restart clears the state. An HTTP worker reset does not clear it.

`max_entries` bounds the number of buckets. Cleanup runs during requests, at most once per minute. It removes only fully refilled buckets. The plugin does not evict depleted buckets to admit new identities. Idle entries can remain until a later request starts cleanup. There is no background cleanup worker.

The middleware provides HTTP admission control only. It has no RPC API, PHP-supplied policy updates, database lookups, path rules, or distributed storage.

## Middleware Order

In v6, requests enter middleware from left to right. Place `proxy_ip_parser` before the limiter to resolve client IPs from trusted proxy headers. Place `headers` before it when error responses need CORS headers. Place `http_metrics` and `otel` before it to include rejected requests in metrics and tracing.

```yaml
http:
  middleware: ["otel", "proxy_ip_parser", "headers", "http_metrics", "rate_limiter"]
```

A handler that answers before the limiter consumes no token. This includes CORS preflight responses when `headers` comes first. Place the limiter before `static` to count static-file requests. Place `static` first to let its responses bypass the limit.

The built-in HTTP access logger runs inside the configured middleware chain. It does not record limiter rejections. The limiter's internal OpenTelemetry span ends before the next handler starts. The HTTP server span includes downstream request time.
