# X-Sendremotefile middleware

The `Sendremotefile` HTTP middleware and the `X-Sendremotefile` HTTP response headers are used to stream large files using the RoadRunner.
Unlike [X-Sendfile middleware](./sendfile.md) `Sendremotefile` allows passing a URL as header value.
While the file is being streamed with the help of the RoadRunner, the PHP worker may be accepting the next request.

The middleware reads the file in 10MB chunks. For example, for the 5Gb file, only 10MB of RSS is used. If the file
is smaller than 10MB, the middleware adjusts the buffer to fit the file size.

## Configuration

{% code title=".rr.yaml" %}

```yaml
version: "3"

http:
  address: 127.0.0.1:55555
  max_request_size: 1024
  access_logs: false
  middleware: [ "sendremotefile" ]

  pool:
    num_workers: 2
    max_jobs: 0
    allocate_timeout: 60s
    destroy_timeout: 60s
```

{% endcode %}

## Samples

{% code title="worker.php" %}

```php
<?php

use Nyholm\Psr7\Response;
use Spiral\Goridge;
use Spiral\RoadRunner;

ini_set('display_errors', 'stderr');
require __DIR__ . "/vendor/autoload.php";

$worker = new RoadRunner\Worker(new Goridge\StreamRelay(STDIN, STDOUT));
$psr17Factory = new \Nyholm\Psr7\Factory\Psr17Factory();
$psr7 = new RoadRunner\Http\PSR7Worker($worker, $psr17Factory, $psr17Factory, $psr17Factory);

while ($req = $psr7->waitRequest()) {
    try {
        $resp = new Response(
            200,
            ["X-Sendremotefile" => "https://example.com/file.mp4", "Content-Disposition" => "attachment; filename=file.mp4"]
        );

        $psr7->respond($resp);
        unset($resp);
    } catch (\Throwable $e) {
        $psr7->getWorker()->error((string)$e);
    }
}
```

{% endcode %}
