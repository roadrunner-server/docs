# RoadRunner Installation

These guides describe the upcoming RoadRunner v3 release with v6 plugins. Use the `master` branch to build the development version.

{% hint style="info" %}
Development builds use the module versions selected in RoadRunner's `go.mod`. Those beta versions can lag behind the merged plugin changes described here. Check the selected versions before using a new option. Use a compatible set of plugin revisions through [Velox](../customization/build.md) to test untagged changes. Default Zstd and Protoreg registration is tracked in [RoadRunner PR #2410](https://github.com/roadrunner-server/roadrunner/pull/2410).
{% endhint %}

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

Run these commands from your application directory. The build uses the plugin versions in the downloaded `go.mod` and produces `./rr` for the host operating system and architecture. Set `RR_REF` to a full commit SHA when you need a repeatable build.

```bash
RR_REF=master
curl --fail --location --output rr-source.tar.gz \
  "https://github.com/roadrunner-server/roadrunner/archive/${RR_REF}.tar.gz"
mkdir rr-source
tar -xzf rr-source.tar.gz --strip-components=1 -C rr-source
CGO_ENABLED=0 go -C rr-source build -mod=readonly -trimpath \
  -ldflags "-s -X github.com/roadrunner-server/roadrunner/v2025/internal/meta.version=${RR_REF}" \
  -o ../rr ./cmd/rr
./rr --version
```

Inspect the compiled module versions with `go version -m ./rr`. Keep the source revision and `go.sum` with your build inputs. To select a different set of plugins, use the [Velox build guide](../customization/build.md).

## Docker

The build above sets `CGO_ENABLED=0` to produce a static binary. For Docker, build for Linux and the container architecture. For example, add `GOOS=linux GOARCH=amd64` before `go` for a Linux AMD64 image. Run the version check on the target platform when you cross-compile.

Use the [Docker guide](../app-server/docker.md#build-the-roadrunner-image) to package the binary as `roadrunner:v3-local` and copy it into a PHP application image.

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
