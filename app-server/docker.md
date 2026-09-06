# Docker Images

Build a local image with v6 plugins from the same RoadRunner revision as the [installation guide](../intro/install.md). The application, Nginx, and debugging examples use this image.

## Build the RoadRunner Image

Create `Dockerfile.rr` in the build directory:

{% code title="Dockerfile.rr" %}

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.27.1 AS build

ARG TARGETOS
ARG TARGETARCH

WORKDIR /src

ADD https://github.com/roadrunner-server/roadrunner/archive/b0cccd917f001b6584eafdc04ad6ba69a97cbb69.tar.gz /tmp/rr.tar.gz

RUN tar -xzf /tmp/rr.tar.gz --strip-components=1 -C /src \
    && CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -mod=readonly -trimpath \
       -ldflags "-s -X github.com/roadrunner-server/roadrunner/v2025/internal/meta.version=dev-b0cccd9" \
       -o /rr ./cmd/rr

FROM scratch
COPY --from=build /rr /usr/bin/rr
```

{% endcode %}

Build the image for the same target platform as your PHP application image:

```bash
docker build -f Dockerfile.rr -t roadrunner:v6-b0cccd9 .
```

`roadrunner:v6-b0cccd9` is a local image that supplies the compiled binary. It does not contain PHP. For a cross-build, pass the same `--platform` value to this command and the application image build.

## Build the Application Image

Here is an example of a `Dockerfile` that can be used to build a Docker image with RoadRunner for a PHP application:

{% hint style="warning" %} 
Note that this example utilizes a folder named `app` for your application. If your application is located in a different folder, feel free to customize this Dockerfile to suit your needs.
{% endhint %}

{% code title="Dockerfile" %}

```dockerfile
FROM roadrunner:v6-b0cccd9 AS roadrunner

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
