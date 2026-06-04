# Proxy IP parser

This middleware resolves the real client IP from proxy headers when a request arrives
through a trusted subnet. By default it consults, in order: `Forwarded`, `X-Forwarded-For`,
`X-Real-IP`, `True-Client-IP`, and `CF-Connecting-IP`. The set and order of headers can be
customized with `trusted_headers`.

## Description

When the immediate peer is within one of the `trusted_subnets`, the middleware resolves the
client IP from the configured headers — the first non-empty match wins — and sets it as
`RemoteAddr`. Otherwise `RemoteAddr` is left unchanged.

The middleware is active only when `trusted_subnets` is configured; without it, forwarding
headers are never trusted.

## Usage

{% code title=".rr.yaml" %}

```yaml
http:
  address: 127.0.0.1:12811
  max_request_size: 1024
  middleware: [ "proxy_ip_parser" ] # Middleware
  uploads:
    forbid: [ ".php", ".exe", ".bat" ]
  trusted_subnets: # Trusted addresses in CIDR format
    [
      "10.0.0.0/8",
      "127.0.0.0/8",
      "172.16.0.0/12",
      "192.168.0.0/16",
      "::1/128",
      "fc00::/7",
      "fe80::/10"
    ]
  pool:
    num_workers: 2
    allocate_timeout: 60s
    destroy_timeout: 60s
```

{% endcode %}

## Trusted headers

`trusted_headers` is an ordered allowlist of the headers used to resolve the client IP. The
middleware checks them in order and uses the first non-empty value; headers that are not
listed are ignored, and custom headers are supported. When `trusted_headers` is omitted, the
default order is used: `Forwarded`, `X-Forwarded-For`, `X-Real-IP`, `True-Client-IP`,
`CF-Connecting-IP`.

For example, to trust only `X-Real-IP` and Cloudflare's `CF-Connecting-IP` while ignoring
`X-Forwarded-*`:

{% code title=".rr.yaml" %}

```yaml
http:
  trusted_subnets: [ "10.0.0.0/8", "127.0.0.0/8" ]
  trusted_headers: [ "X-Real-IP", "CF-Connecting-IP" ]
```

{% endcode %}

{% hint style="info" %}
`X-Forwarded-For` uses the left-most address from its comma-separated list, and `Forwarded`
is parsed per [RFC 7239](https://datatracker.ietf.org/doc/html/rfc7239) (`for=`). All other
headers, including custom ones, are taken verbatim.
{% endhint %}
