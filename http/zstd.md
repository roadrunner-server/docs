# HTTP - Zstd middleware

The `zstd` middleware compresses HTTP response bodies when the client sends `Accept-Encoding: zstd`. It uses a pure Go implementation and does not require CGO. It does not decompress request bodies.

## Availability

{% hint style="info" %}
Your RoadRunner build must include the `github.com/roadrunner-server/zstd/v6` plugin. Adding `zstd` to the configuration does not install the plugin. See [Building a Server](../customization/build.md) and [Registering middleware](../customization/middleware.md#registering-middleware) for custom builds.
{% endhint %}

## Configuration

Add `zstd` to the middleware list in your existing HTTP configuration:

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  address: 127.0.0.1:8080
  middleware: [ "zstd" ]
```

{% endcode %}

The middleware does not require a separate configuration section. Eligible responses use `Content-Encoding: zstd`.

## Compression behavior

- Clients must request `zstd` explicitly with a nonzero quality value. Missing or unsupported encodings and `zstd;q=0` leave the response uncompressed.
- The default compression level is `zstd.SpeedFastest`. The implementation reuses encoders through a pool.
- The normal minimum response size is 1,024 bytes. A flush can start compression below this limit.
- The middleware skips HEAD requests, empty bodies, responses with an existing `Content-Encoding` or `Content-Range`, and content types excluded by the compression library.
- The middleware adds `Vary: Accept-Encoding`. It removes the original `Content-Length` when it compresses a response.
- The compression library does not change ETags by default.

The middleware supports OpenTelemetry header propagation when RoadRunner tracing is active.

## Using gzip and zstd

The zstd middleware does not provide gzip fallback. The [gzip middleware](gzip.md) is a separate plugin.

{% hint style="warning" %}
If both gzip and zstd middleware are enabled, their order can determine the selected encoding. The separate plugins do not compare quality values with each other. Do not expect them to select the encoding with the highest quality value across both plugins.
{% endhint %}

## Documentation

- [Zstd middleware plugin](https://github.com/roadrunner-server/zstd)
- [MDN Accept-Encoding header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Accept-Encoding)
- [HTTP response streaming](resp-streaming.md)
