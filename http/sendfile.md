# HTTP — X-Sendfile middleware

The `Send` HTTP middleware and the `X-Sendfile` HTTP response header are used to stream large files using RoadRunner.
While the file is being streamed with the help of RoadRunner, the PHP worker may accept the next request.

The middleware reads the file with a buffer of up to 10 MiB. For smaller files, the buffer matches the file size. See the [X-Sendfile proposal](https://github.com/roadrunner-server/roadrunner-plugins/issues/9).

## Similar approaches

- [NGINX](https://www.nginx.com/resources/wiki/start/topics/examples/xsendfile/)
- [Apache2](https://tn123.org/mod_xsendfile/)

## Configuration

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  address: 127.0.0.1:55555
  max_request_size: 1024
  access_logs: false
  middleware: [ "sendfile" ]

  pool:
    num_workers: 2
    max_jobs: 0
    allocate_timeout: 60s
    destroy_timeout: 60s
```

{% endcode %}

## File responses

In v6 beta, a response with `X-Sendfile` uses `Content-Type: application/octet-stream`. This replaces any content type supplied by the PHP worker. Check clients that depend on a specific media type for inline display. Use `Content-Disposition: attachment` for downloads.

An empty file returns `200 OK` with no body. Paths are normalized before file access. Use paths controlled by the application: this middleware does not restrict access to a configured root directory.

To apply gzip to file responses, put `gzip` before `sendfile` in the v6 middleware list. See [middleware order](./http.md#middleware-order).
