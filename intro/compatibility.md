# Upgrade and Compatibility

These reference pages describe RoadRunner v3 and its v6 plugins. Use the [2025 to v3 upgrade guide](v3-migration.md) when updating an existing application. See the [v3 release notes](../releases/v3-0-0.md) for the major changes.

## Configuration Version

| RoadRunner release | `.rr.yaml` version | Go plugin modules |
| --- | --- | --- |
| v2025 | `"3"` | `/v5` |
| v3 | `"3"` | `/v6` |

Keep `version: "3"` at the root of `.rr.yaml`. The configuration version is separate from the RoadRunner release and the Go plugin versions.

## PHP Applications

Use compatible RoadRunner PHP packages for each enabled plugin. The [RPC reference](../php/rpc.md) describes codec compatibility. The [plugin API reference](../customization/plugin.md#dto-compatibility) describes changes to generated DTOs.

## Custom Builds

Build custom Go plugins with the v6 interfaces and module paths. See [Writing a Plugin](../customization/plugin.md) and [Building RR with a Custom Plugin](../customization/build.md). Source builds require Go 1.27 or later.
