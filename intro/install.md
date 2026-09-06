# RoadRunner Installation

Build RoadRunner with v6 plugins from revision [`b0cccd917f001b6584eafdc04ad6ba69a97cbb69`](https://github.com/roadrunner-server/roadrunner/tree/b0cccd917f001b6584eafdc04ad6ba69a97cbb69). This revision and its plugin dependencies are the target for these guides.

## Requirements

- Go `1.27.1` for the source build.
- PHP `8.5` with the `sockets` extension for the examples.
- Composer 2 for PHP dependencies.
- `curl` and `tar` to download and extract the source archive.

Check the installed versions and PHP extensions:

```bash
go version
php --version
php --modules
composer --version
```

## Build From Source

Run these commands from your application directory. The build uses the plugin versions in the downloaded `go.mod` and produces `./rr` for the host operating system and architecture.

```bash
curl --fail --location --output rr-source.tar.gz \
  https://github.com/roadrunner-server/roadrunner/archive/b0cccd917f001b6584eafdc04ad6ba69a97cbb69.tar.gz
mkdir rr-source
tar -xzf rr-source.tar.gz --strip-components=1 -C rr-source
CGO_ENABLED=0 go -C rr-source build -mod=readonly -trimpath \
  -ldflags "-s -X github.com/roadrunner-server/roadrunner/v2025/internal/meta.version=dev-b0cccd9" \
  -o ../rr ./cmd/rr
./rr --version
```

Inspect the compiled module versions with `go version -m ./rr`. Keep the source revision and `go.sum` with your build inputs. To select a different set of plugins, use the [Velox build guide](../customization/build.md).

## Docker

Use the [Docker source build](../app-server/docker.md#build-the-roadrunner-image) to create a local RoadRunner image from the same revision. That guide also shows how to copy the binary into a PHP application image.

## Composer

Install the PHP packages needed by your worker. For an HTTP worker, run:

```bash
composer require spiral/roadrunner-http nyholm/psr7
```

Composer installs the PHP libraries. The source build above supplies the RoadRunner binary.

## What's Next?

1. [Quick Start Guide](quick-start.md).
2. [RoadRunner Configuration](config.md).
3. [Developer Mode](../php/developer.md).
