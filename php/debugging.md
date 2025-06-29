# Debugging

You can use RoadRunner scripts with xDebug extension. In order to enable configure your IDE to accept remote connections.

{% hint style="info" %}
If you run multiple PHP processes you have to extend the maximum number of allowed connections to the number of
active workers, otherwise some calls would not be caught on your breakpoints.
{% endhint %}

### XDebug for RoadRunner installed on your system

![xdebug](https://user-images.githubusercontent.com/796136/46493729-c767b400-c819-11e8-9110-505a256994b0.png)

To activate xDebug make sure to set the `xdebug.mode=debug` in your `php.ini`.

To enable xDebug in your application make sure to set ENV variable `XDEBUG_SESSION`:

{% code title=".rr.yaml" %}

```yaml
rpc:
   listen: tcp://127.0.0.1:6001

server:
   command: "php worker.php"
   env:
     - XDEBUG_SESSION: 1

http:
   address: "0.0.0.0:8080"
   pool:
      num_workers: 1
      debug: true
```

{% endcode %}

Please, keep in mind this guide: [xdebug3](https://xdebug.org/docs/upgrade_guide).  

You should be able to use breakpoints and view state at this point.

## PhpStorm notes

{% code %}

```bash
export PHP_IDE_CONFIG="serverName=octane-app.test"
export XDEBUG_SESSION="mode=debug start_with_request=yes client_host=127.0.0.1 client_port=9003 idekey=PHPSTORM"

php -dvariables_order=EGPCS artisan octane:start --max-requests=250 --server=roadrunner --port=8000 --rpc-port=6001 --watch --workers=1
```

{% endcode %}

### XDebug for RoadRunner installed on Docker

First create a `docker-compose.yml` file in your project root or copy environment and extra_hosts sections in your existing `docker-compose.yml` file:

{% code title="docker-compose.yml" %}

```yml
    services:
    api:
      build:
        context: ./
        dockerfile: Dockerfile
      container_name: projectName-api
      command: rr serve -c .rr.yaml
      tty: true
      ports:
        - "8080:8080"
      volumes:
        - ./api:/app
      networks:
        - default
      environment:
        - PHP_IDE_CONFIG=serverName=docker
      extra_hosts:
        - "host.docker.internal:host-gateway"
```

{% endcode %}

Next, create a `.rr.yaml` file in your project root or copy the following content:

{% code title=".rr.yaml" %}

```yaml
version: '3'
rpc:
    listen: 'tcp://127.0.0.1:6001'
http:
    address: '0.0.0.0:8080'
    middleware:
        - gzip
        - static
    static:
        dir: public
        forbid:
            - .php
            - .htaccess
    pool:
        debug: true
        num_workers: 1
        supervisor:
            max_worker_memory: 100
server:
    command: 'php app.php'
    relay: pipes
    env:
      - XDEBUG_SESSION: 1
metrics:
    address: '127.0.0.1:2112'
```

{% endcode %}

Next create a `Dockerfile` file in your project root or copy the following content:

{% code title="Dockerfile" %}

```dockerfile
FROM ghcr.io/roadrunner-server/roadrunner:2025.1.1 AS roadrunner

FROM php:8.4-cli-alpine3.21

# change version to latest if you want to use the latest version of xDebug, see https://xdebug.org
ENV XDEBUG_VERSION=3.4.4

RUN apk add --no-cache autoconf g++ make postgresql-dev coreutils --update linux-headers \
    && pecl install xdebug-$XDEBUG_VERSION \
    && rm -rf /tmp/pear \
    && docker-php-ext-configure pgsql -with-pgsql=/usr/local/pgsql \
    && docker-php-ext-install pdo_pgsql sockets \
    && docker-php-ext-enable xdebug \
    && apk del linux-headers make g++ autoconf

RUN mv $PHP_INI_DIR/php.ini-development $PHP_INI_DIR/php.ini
COPY xdebug.ini /usr/local/etc/php/conf.d/xdebug.ini

RUN curl -sS https://getcomposer.org/installer | php \
    && mv composer.phar /usr/local/bin/composer \
    && chmod +x /usr/local/bin/composer

RUN addgroup -g 1000 app && adduser -u 1000 -G app -s /bin/sh -D app

WORKDIR /app

COPY --from=roadrunner /usr/bin/rr /usr/local/bin/rr

RUN chown app:app /usr/local/bin/rr && chmod +x /usr/local/bin/rr

USER app
```

{% endcode %}

Important: Make sure to change the `XDEBUG_VERSION` to the version you want to use, see [xdebug.org](https://xdebug.org/docs/install). Also note that the example container is run on behalf of the `app` user.

Next create a `xdebug.ini` file in your project root or copy the following content:

{% code title="xdebug.ini" %}

```ini
xdebug.mode=develop,debug
xdebug.start_with_request=yes
xdebug.discover_client_host=0
xdebug.client_host=host.docker.internal
xdebug.client_port=9003
xdebug.log=/dev/stdout
xdebug.log_level=0
```

{% endcode %}

Next you're needed change settings in your IDE to accept remote connections. For example, in PhpStorm you can do it by going to `PHP > Debug` and set the following options:
![PhpStorm xDebug settings](https://raw.githubusercontent.com/lobanovkirill/roadrunner-docs/442d06de35c41f30a5249bb087204cf60beec971/Screenshot%20from%202025-06-29%2021-42-36.png)

Next you're needed create `Server` in your IDE. For example, in PhpStorm you can do it by going to `PHP > Servers` and set the following options:
![PhpStorm xDebug server settings](https://raw.githubusercontent.com/lobanovkirill/roadrunner-docs/442d06de35c41f30a5249bb087204cf60beec971/Screenshot%20from%202025-06-29%2021-42-26.png)

Also, please check `Docker` setting in your IDE. For example, in PhpStorm you can do it by going to `Build, Execution, Deployment > Docker` and set the following options:
![PhpStorm Docker settings](https://raw.githubusercontent.com/lobanovkirill/roadrunner-docs/442d06de35c41f30a5249bb087204cf60beec971/Screenshot%20from%202025-06-29%2021-43-03.png)

{% code %}

```bash

## Jobs debugging

### Prerequisites

1. XDebug 3 installed and properly configured.
2. Number of jobs workers set to `1` with `jobs.pool.num_workers` configuration option in `.rr.yaml`.

### Debug process

If you have any active XDebug listener while starting RoadRunner with XDebug enabled — disable it. This will prevent
false-positive debug session.

Once RoadRunner starts all workers, enable XDebug listener and reset jobs workers with:

{% code %}

```bash
./rr reset jobs
```

{% endcode %}

Now you should see debug session started:

1. Step over to some place where job task is being resolved with `$consumer->waitTask()`.
2. As soon as you reach it, debugger stops but session will still be active.
3. Trigger task consumption with either an HTTP request or a console command (depending on how your application works).
and continue to debug job worker as usual within active session started before.
4. You can continue to debug jobs as long as debug session active.

If connections session broken or timed out, you can repeat instruction above to reestablish connection by resetting jobs workers.
