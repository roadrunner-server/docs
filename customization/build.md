# Building a Server

Velox builds a RoadRunner binary from the plugins listed in `velox.toml`. Use it to select plugins or build with a custom plugin or fork.

{% hint style="warning" %}
**Velox v3 is untagged.** This guide requires development revision `6b71101ce0080143b4927cf2d84ab0ba02189b67`, not a released Velox binary. The example uses RR development source with v6 beta plugins. Use the pinned installation command below.
{% endhint %}

For an existing v5 build, see the [Velox v2025.1.7 configuration](https://github.com/roadrunner-server/velox/blob/v2025.1.7/velox.toml). That release uses the older format and does not support the configuration below.

## Configuration

This TOML configuration pins the RR commit and plugin versions. The plugin tags match the requirements in the [pinned RR source](https://github.com/roadrunner-server/roadrunner/blob/b0cccd917f001b6584eafdc04ad6ba69a97cbb69/go.mod).

{% code title="velox.toml" %}

```toml
[roadrunner]
ref = "b0cccd917f001b6584eafdc04ad6ba69a97cbb69"

[github]
base_url = "https://github.com"

[github.token]
token = "${GITHUB_TOKEN}"

[plugins.logger]
module_name = "github.com/roadrunner-server/logger/v6"
tag = "v6.0.0-beta.4"

[plugins.server]
module_name = "github.com/roadrunner-server/server/v6"
tag = "v6.0.0-beta.7"

[plugins.rpc]
module_name = "github.com/roadrunner-server/rpc/v6"
tag = "v6.0.0-beta.6"

[plugins.http]
module_name = "github.com/roadrunner-server/http/v6"
tag = "v6.0.0-beta.10"

[log]
level = "info"
mode = "production"
```

{% endcode %}

Velox includes `informer` and `resetter` automatically from the downloaded RR `go.mod`. Do not add them to `[plugins]`. Velox warns and ignores those entries.

List each plugin module once. Custom plugins must export a `Plugin` type from the module root.

### Options

| Key | Meaning |
| --- | --- |
| `roadrunner.ref` | RR semver tag, branch, or 40-character hexadecimal commit SHA. Defaults to `master`. |
| `plugins.<name>.module_name` | Full Go module path, including its major-version suffix. |
| `plugins.<name>.tag` | Plugin version or ref. Use an exact semver tag to enable version checks. |
| `target_platform.os` | Target `GOOS`. Defaults to the host OS. |
| `target_platform.arch` | Target `GOARCH`. Defaults to the host architecture. |
| `debug.enabled` | Set to `true` to disable optimization and inlining, retain debug symbols, and enable the `debug` build tag. Defaults to `false`. |
| `debug.race` | Set to `true` to build with `-race` and `CGO_ENABLED=1`. Defaults to `false`. |

Each missing target platform value defaults independently. Race builds need a C compiler and C libraries for the target platform. Without `debug.race`, Velox sets `CGO_ENABLED=0`.

### Module replacements

Append `[[replaces]]` and `[[excludes]]` sections when you need Go module overrides. This example uses a compatible local HTTP fork at `../http` and excludes one dependency version:

{% code title="velox.toml" %}

```toml
[[replaces]]
new = "../http"
old = "github.com/roadrunner-server/http/v6"

[[excludes]]
module = "github.com/redis/go-redis/v9"
version = "v9.15.0"
```

{% endcode %}

Relative replacement paths resolve against the working directory of `vx`, not the configuration file or downloaded RR directory. A container build must mount the local module at a path available inside the container.

For a remote replacement, use `module@version` in `new`. The `old` value can include `@version` to restrict the replacement to that version. A local path must not have a version suffix. Each `old` value must be unique.

An excluded version must be canonical semver and match the module path major. Velox applies these directives before Go resolves dependencies. It retains the downloaded RR `go.mod` as the starting point.

### Older configuration

Velox v3 ignores the old `[github.plugins.*]` and `[gitlab.*]` tables. It does not convert them. Move plugin entries to `[plugins.<name>]` with `module_name` and `tag`.

The old per-plugin `ref`, `owner`, `repository`, `folder`, and inline `replace` fields are also ignored. Use the module path declared in the plugin's `go.mod`, including for plugins in repository subdirectories. Move inline replacements to `[[replaces]]`.

### Private repositories

Go downloads plugin modules, including modules hosted on GitLab. Configure SSH or HTTPS credentials for Go module downloads. Set `GOPRIVATE` for your private module prefixes. Replace the organization names in this example:

```bash
export GOPRIVATE="github.com/your-org/*,gitlab.com/your-org/*"
```

Velox inherits the Go environment, including `GOPRIVATE`, `GOPROXY`, and `GOFLAGS`. The GitHub archive token does not authenticate plugin module downloads.

### RR archive access

The `[github.token]` section is optional for public RR source. Its `token` value expands environment variables and is sent as a bearer token on the archive request. With the example above, export `GITHUB_TOKEN` when authentication is needed. Exporting the variable without the token configuration does not enable authentication.

To download from GitHub Enterprise, set `[github] base_url` to your host, such as `https://ghe.example.com`. The RR mirror must be at `<base_url>/roadrunner-server/roadrunner`. This setting does not change plugin module hosts. Archive downloads accept redirects and direct HTTP 200 responses.

## Building

Use Go `1.27.1` for this example. Install the pinned Velox development revision:

{% code title="go install" %}

```bash
go install github.com/roadrunner-server/velox/v3/cmd/vx@6b71101ce0080143b4927cf2d84ab0ba02189b67
```

{% endcode %}

Add the Go binary installation directory to `PATH`. Build from the directory that contains `velox.toml`:

{% code title="vx build" %}

```bash
SOURCE_DATE_EPOCH=1788438954 vx build -c velox.toml -o .
```

{% endcode %}

| Option | Meaning |
| --- | --- |
| `-c`, `--config` | Configuration file. Defaults to `velox.toml`. |
| `-o`, `--out` | Output directory. Defaults to the current directory. |

The command produces `./rr`. When the target matches the host OS and architecture, Velox checks `rr --version` before it replaces the output binary. For a cross-build, run that check on the target system. The output directory can be on another filesystem, including a container volume mount.

### Reproducible builds

`SOURCE_DATE_EPOCH` sets the binary build timestamp in Unix seconds. The value above is the pinned RR commit's timestamp. Without a valid value, Velox uses the current time. The reported binary version comes from `roadrunner.ref`; the old `VERSION` and `TIME` environment variables are not used.

Keep the Velox revision, RR commit, plugin tags, Go toolchain, target platform, build flags, and environment fixed for repeated builds. Keep local replacement contents fixed.

After Go resolves dependencies, Velox checks semver plugin pins. A different resolved version causes the build to fail. Use `[[replaces]]` to force a version only after you check its compatibility.

Non-semver plugin refs, including `latest`, branch names, and commit SHAs, skip this version comparison. A same-module replacement compares its replacement version with the requested plugin tag. Local and different-module replacements skip the comparison.

## Known limitations

- Windows targets are rejected, including `target_platform.os = "windows"`.
- The Connect/gRPC build server, `vx server`, and `--address`/`-a` are removed. Use `vx build`.
