# Docker Images

Build a local image from the RoadRunner binary selected in the [installation guide](../intro/install.md). The application, Nginx, and debugging examples use this image.

## Build the RoadRunner Image

Build a static Linux `rr` binary for the target architecture. Use `CGO_ENABLED=0` for a source build. Put the binary and `Dockerfile.rr` in the same build directory:

{% code title="Dockerfile.rr" %}

```dockerfile
FROM scratch AS roadrunner
COPY rr /usr/bin/rr
```

{% endcode %}

Build the image for the same target platform as your PHP application image:

```bash
docker build -f Dockerfile.rr -t roadrunner:v3-local .
```

`roadrunner:v3-local` contains the compiled binary. Copy this binary into an application image with PHP. The binary, this image, and the application image must use the same target architecture. Setting Docker `--platform` does not recompile a binary copied from the host.

## Build the Application Image

Here is an example of a `Dockerfile` that can be used to build a Docker image with RoadRunner for a PHP application:

{% hint style="warning" %} 
Note that this example utilizes a folder named `app` for your application. If your application is located in a different folder, feel free to customize this Dockerfile to suit your needs.
{% endhint %}

{% code title="Dockerfile" %}

```dockerfile
FROM roadrunner:v3-local AS roadrunner

FROM php:8.5-cli-alpine

# https://github.com/mlocati/docker-php-extension-installer
# https://github.com/docker-library/docs/tree/0fbef0e8b8c403f581b794030f9180a68935af9d/php#how-to-install-more-php-extensions
RUN --mount=type=bind,from=mlocati/php-extension-installer:2,source=/usr/bin/install-php-extensions,target=/usr/local/bin/install-php-extensions \
     install-php-extensions @composer-2 opcache zip intl sockets protobuf

COPY --from=roadrunner /usr/bin/rr /usr/local/bin/rr

EXPOSE 8080/tcp

WORKDIR /app

ENV COMPOSER_ALLOW_SUPERUSER=1

# Copy composer files from the app directory to install dependencies
COPY ./app/composer.* .

RUN composer install --optimize-autoloader --no-dev

# Copy application files
COPY ./app .

# Run the RoadRunner server
CMD ["/usr/local/bin/rr", "serve", "-c", ".rr.yaml"]
```

{% endcode %}
