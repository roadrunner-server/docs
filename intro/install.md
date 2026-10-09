# RoadRunner Installation

These guides describe RoadRunner v3 with v6 plugins.

## Requirements

- Go `1.27.1` for the source build.
- PHP `8.5` with the `sockets` extension for the examples.
- Composer 2 for PHP dependencies.
- `curl` and `tar` to download and extract archives.

Check the installed versions and PHP extensions:

```bash
go version
php --version
php --modules
composer --version
```

## Pre-built Binaries

Download RoadRunner `v3.0.0` from the [GitHub release page](https://github.com/roadrunner-server/roadrunner/releases/tag/v3.0.0). Select the archive for your operating system and architecture. Extract `rr` into your application directory.

For Linux AMD64, run:

```bash
curl --fail --location --output roadrunner-3.0.0-linux-amd64.tar.gz \
  https://github.com/roadrunner-server/roadrunner/releases/download/v3.0.0/roadrunner-3.0.0-linux-amd64.tar.gz
tar -xzf roadrunner-3.0.0-linux-amd64.tar.gz --strip-components=1 \
  roadrunner-3.0.0-linux-amd64/rr
./rr --version
```

## Build From Source

Run these commands from your application directory. The build uses the plugin versions in the downloaded `go.mod` and produces `./rr` for the host operating system and architecture. Set `RR_REF` to a full commit SHA when you need a repeatable build.

```bash
RR_REF=v3.0.0
curl --fail --location --output rr-source.tar.gz \
  "https://github.com/roadrunner-server/roadrunner/archive/${RR_REF}.tar.gz"
mkdir rr-source
tar -xzf rr-source.tar.gz --strip-components=1 -C rr-source
CGO_ENABLED=0 go -C rr-source build -mod=readonly -trimpath \
  -ldflags "-s -X github.com/roadrunner-server/roadrunner/v3/internal/meta.version=${RR_REF}" \
  -o ../rr ./cmd/rr
./rr --version
```

Inspect the compiled module versions with `go version -m ./rr`. Keep the source revision and `go.sum` with your build inputs. To select a different set of plugins, use the [Velox build guide](../customization/build.md).

## Docker

Use the official image `ghcr.io/roadrunner-server/roadrunner:3.0.0`. The [Docker guide](../app-server/docker.md#build-the-application-image) shows how to copy its binary into a PHP application image. The v3 image tags are `3`, `3.0`, and `3.0.0`.

For a custom source build, set `CGO_ENABLED=0` and build for Linux and the container architecture. For example, add `GOOS=linux GOARCH=amd64` before `go` for a Linux AMD64 image. Run the version check on the target platform when you cross-compile. See [Build the RoadRunner Image](../app-server/docker.md#build-the-roadrunner-image) to package a custom binary.

## Composer

Install the PHP packages needed by your worker. For an HTTP worker, run:

```bash
composer require spiral/roadrunner-http nyholm/psr7
```

Composer installs the PHP libraries. Use a pre-built binary or a source build for the RoadRunner server.

## What's Next?

1. [Quick Start Guide](quick-start.md).
2. [RoadRunner Configuration](config.md).
3. [Developer Mode](../php/developer.md).
