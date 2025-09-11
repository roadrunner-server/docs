# HTTP — Gzip middleware

The gzip middleware supports the `Accept-Encoding: gzip` header and compresses or decompresses the contents of
outgoing and incoming requests.

## Documentation

- MDN [link](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Encoding)

## Configuration

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  address: 127.0.0.1:15389
  middleware: [ gzip ]
  pool:
    num_workers: 10
    allocate_timeout: 60s
    destroy_timeout: 60s
```

{% endcode %}

The gzip middleware supports OpenTelemetry header propagation.
