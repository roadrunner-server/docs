# Quick Start Guide

This guide will walk you through the steps to get started with RoadRunner. You'll learn how to install RoadRunner and configure it for your project.

## Step 1: Download RR for your platform

To begin, you need to download RR for your platform. Visit the [RR installation guide](install.md) and download the appropriate version for your operating system.

## Step 2: Install PHP

RR requires PHP to run. If you don't have PHP installed, you can download it from the [official PHP website](https://www.php.net/downloads.php) and follow the installation instructions for your operating system.

## Step 3: Create a simple configuration

Next, you need to create a simple configuration file for RR. Open a text editor and create a new file called `.rr.yaml`. Add the following content to the file:

{% code title=".rr.yaml" overflow="wrap" lineNumbers="true" %}

```yaml
server:
  command: "php psr-worker.php"

http:
  address: 0.0.0.0:8080

# do not use development mode in production
logs:
  level: debug
  mode: development
```

{% endcode %}

## Step 4: Create a simple worker

Now you need to create a simple worker. Create a new file called `psr-worker.php` and add the following content to the file:

{% code title="psr-worker.php" overflow="wrap" lineNumbers="true" %}

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use Nyholm\Psr7\Response;
use Nyholm\Psr7\Factory\Psr17Factory;

use Spiral\RoadRunner\Worker;
use Spiral\RoadRunner\Http\PSR7Worker;


// Create new RoadRunner worker from global environment
$worker = Worker::create();

// Create common PSR-17 HTTP factory
$factory = new Psr17Factory();

$psr7 = new PSR7Worker($worker, $factory, $factory, $factory);

while (true) {
    try {
        $request = $psr7->waitRequest();
        if ($request === null) {
            break;
        }
    } catch (\Throwable $e) {
        // Although the PSR-17 specification clearly states that there can be
        // no exceptions when creating a request, however, some implementations
        // may violate this rule. Therefore, it is recommended to process the 
        // incoming request for errors.
        //
        // Send "Bad Request" response.
        $psr7->respond(new Response(400));
        continue;
    }

    try {
        // Here is where the call to your application code will be located. 
        // For example:
        //  $response = $app->send($request);
        //
        // Reply by the 200 OK response
        $psr7->respond(new Response(200, [], 'Hello RoadRunner!'));
    } catch (\Throwable $e) {
        // In case of any exceptions in the application code, you should handle
        // them and inform the client about the presence of a server error.
        //
        // Reply by the 500 Internal Server Error response
        $psr7->respond(new Response(500, [], 'Something Went Wrong!'));
        
        // Additionally, we can inform the RoadRunner that the processing 
        // of the request failed.
        $psr7->getWorker()->error((string)$e);
    }
}

{% endcode %}


## Step 5: Start the server

Now you can start the server. You should have the following files in the current folder:
- `.rr.yaml`
- `psr-worker.php`
- `rr` binary

Then, open a terminal window in the current folder and run the following command:

```bash
./rr serve
```

Since you have the logs in the development mode, you should see the following output:
![RR logs](image.png)
