# Proxy IP parser

This middleware gets the client address from HTTP forwarding headers when the request comes from a trusted proxy. In v6, `trusted_headers` selects the headers and their order.

## Description

When the immediate peer is within `trusted_subnets`, the middleware uses the first nonempty parsed header value as `RemoteAddr`. Otherwise, it leaves `RemoteAddr` unchanged. This setting controls trust in forwarding headers; it does not block incoming connections.

Add `proxy_ip_parser` to `http.middleware` and configure a nonempty `http.trusted_subnets` list to enable it. An omitted or empty subnet list disables it. Each subnet must use CIDR notation, such as `127.0.0.1/32` or `::1/128`.

## Usage

{% code title=".rr.yaml" %}

```yaml
http:
  address: 127.0.0.1:12811
  max_request_size: 1024
  middleware: [ "proxy_ip_parser" ] # Middleware
  uploads:
    forbid: [ ".php", ".exe", ".bat" ]
  # Replace this with the immediate proxy's actual CIDR.
  trusted_subnets: [ "127.0.0.1/32" ]
  pool:
    num_workers: 2
    allocate_timeout: 60s
    destroy_timeout: 60s
```

{% endcode %}

## Trusted headers

`http.trusted_headers` is an ordered allowlist. The middleware ignores headers that are not listed. It removes whitespace around configured header names, compares names without case sensitivity, and removes duplicate names.

An omitted, empty, or all-blank list uses the default order: `Forwarded`, `X-Forwarded-For`, `X-Real-IP`, `True-Client-IP`, `CF-Connecting-IP`. An empty header list does not disable trust. If a header produces no parsed value, the middleware tries the next header. For example, a `Forwarded` value without `for=` does not prevent use of `X-Forwarded-For`.

The following example trusts only `X-Real-IP` and `CF-Connecting-IP`:

{% code title=".rr.yaml" %}

```yaml
http:
  middleware: [ "proxy_ip_parser" ]
  trusted_subnets: [ "10.20.0.10/32" ]
  trusted_headers: [ "X-Real-IP", "CF-Connecting-IP" ]
```

{% endcode %}

{% hint style="info" %}
`X-Forwarded-For` uses the first value before a comma. `Forwarded` uses the first `for=` value and removes its surrounding quotes. All other headers, including custom headers, are used without changing their values.
{% endhint %}

{% hint style="warning" %}
The parser does not validate that a selected header value is an IP address. Trust only proxy addresses you control. Configure each trusted proxy to overwrite the selected headers so a client cannot supply the address used by the application.
{% endhint %}

## PROXY protocol

HTTP forwarding headers are separate from the TCP PROXY protocol. The pinned HTTP beta does not support PROXY protocol. For custom development builds, see [Development: PROXY protocol](./http.md#development-proxy-protocol).
