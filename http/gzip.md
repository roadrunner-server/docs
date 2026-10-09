# HTTP - Gzip middleware

The gzip middleware can compress HTTP responses for clients that send `Accept-Encoding: gzip`. It does not decompress incoming request bodies.

## Documentation

- MDN: [Accept-Encoding](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Encoding)

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

Put `gzip` before `static` or `sendfile` to apply compression to their responses. See [middleware order](./http.md#middleware-order).

This middleware provides gzip compression only. Use the separate [Zstd middleware](zstd.md) for clients that accept `zstd`.

The gzip middleware supports OpenTelemetry header propagation.
