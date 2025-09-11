# Access logs (HTTP)

RoadRunner supports HTTP access logs, which provide detailed information about incoming HTTP requests and responses.

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
```

{% endcode %}

Once enabled, RoadRunner will log the following information for each incoming HTTP request:

- `method` - HTTP method of the request
- `remote_addr` - Remote address of the client
- `bytes_sent` - Content length of the response
- `http_host` - Host name of the server
- `request` - Query string of the request
- `time_local` - Local time of the server in Common Log Format
- `request_length` - Request body size including headers in bytes (content length + size of all headers). The maximum allowed header
  size for RR is 1 MB.
- `request_time` - Request processing time in seconds with millisecond resolution
- `status` - HTTP response status
- `http_user_agent` - HTTP user [agent](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/User-Agent) string of the client
- `http_referer` - HTTP [referer](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referer) string of the client
