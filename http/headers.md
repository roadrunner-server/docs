# HTTP — Headers and CORS

The headers middleware sets request/response headers and controls CORS for your application.

## CORS

To enable CORS headers, add the following section to your configuration.

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  address: 127.0.0.1:44933
  middleware: ["headers"]
  # ...
  headers:
    cors:
      allowed_origin: "*"
      # If `allowed_origin_regex` option is set, the content of `allowed_origin` is ignored
      allowed_origin_regex: "^http://foo"
      allowed_headers: "*"
      allowed_methods: "GET,POST,PUT,DELETE"
      allow_credentials: true
      exposed_headers: "Cache-Control,Content-Language,Content-Type,Expires,Last-Modified,Pragma"
      max_age: 600
      # Status code to use for successful OPTIONS requests. Default value is 200.
      options_success_status: 200
      # Debugging flag adds additional output to debug server side CORS issues, consider disabling in production.
      debug: false
```

{% endcode %}

> Make sure to declare "headers" middleware.

{% hint style="info" %}
Since RoadRunner v2023.2.0, the following changes were made:

- Ability to define status code of successful OPTIONS request via `options_success_status` config;
- Debug flag (`debug: true/false`) was added to enable additional output to debug CORS issues;
- Allowed to define multiple `allowed_origin` values separated by comma;
- CORS requests are handled using [rs/cors](https://github.com/rs/cors) package.
{% endhint %}

## Custom headers for response or request

You can control additional headers for outgoing responses and headers to be added to requests sent to your application.

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  # ...
  headers:
      # Automatically add headers to every request passed to PHP.
      request:
        Example-Request-Header: "Value"
    
      # Automatically add headers to every response.
      response:
        X-Powered-By: "RoadRunner"
```

{% endcode %}
