# Access logs (HTTP)

RoadRunner writes HTTP access logs through the `http` logger channel at `info` level.

{% hint style="info" %}
This feature is disabled by default, but it can be enabled by configuring the HTTP server.
{% endhint %}

## Enabling HTTP Access Logs

To enable HTTP access logs in RoadRunner, you need to modify the configuration file of the HTTP server.

**Here is an example configuration file:**

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  address: 127.0.0.1:8000
  access_logs: true
  # ...

logs:
  channels:
    http:
      mode: production
      level: info
```

{% endcode %}

The message is `http access log`. Use production mode or a [custom format](logger.md#custom-format) with `%attrs%` to retain its fields:

| Field | Value |
| --- | --- |
| `method` | HTTP request method. |
| `remote_address` | Client address. |
| `read_bytes` | Request body bytes read. |
| `write_bytes` | Response body bytes written. |
| `content_len` | Request content length. |
| `host` | Request host. |
| `URI` | Request URI. |
| `URL` | Request URL. |
| `query` | Query string. |
| `status` | Final HTTP response status. |
| `user_agent` | User-Agent header. |
| `referer` | Referer header. |
| `time_local` | Request start time in the `02/Jan/06:15:04:05 -0700` layout. |
| `request_time`, `start` | Request start timestamp. |
| `elapsed` | Request duration in whole milliseconds. |

Use `elapsed` to measure request duration. The `request_time` field is a timestamp. See [Logger](logger.md) for JSON record fields and asynchronous output.

## Client Addresses Behind a Proxy

Enable `proxy_ip_parser` to use forwarded client addresses in `remote_address`. Set `trusted_subnets` to the networks that contain your proxies. Set `trusted_headers` to select the headers and their priority:

{% code title=".rr.yaml fragment" %}

```yaml
http:
  access_logs: true
  middleware: [proxy_ip_parser]
  trusted_subnets: ["10.0.0.0/8"]
  trusted_headers: ["X-Forwarded-For", "X-Real-IP"]
```

{% endcode %}

See [Trusted Subnets](../http/proxy.md) for address selection and configuration.
