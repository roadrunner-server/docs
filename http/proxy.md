# Proxy IP parser

This middleware parses the following headers: `X-Forwarded-*`, `Forwarded`, `True-Client-IP`, and `X-Real-IP`.

## Description
When the proxy is trusted, the leftmost IP address is used as `RemoteAddr`. Otherwise, `RemoteAddr` is not changed.

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
