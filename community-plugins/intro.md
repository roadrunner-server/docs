# Community plugins

## Introduction

Community plugins are plugins that are not officially supported by the RoadRunner team. They are maintained by the community and are not part of the RoadRunner distribution. They can be build using the tools called [Velox](https://github.com/roadrunner-server/velox).

Not official does not mean that they are not good. In fact, some of the community plugins are very useful and can be used in production.
Community plugin doesn't neesesary should be in the `roadrunner-server` organization. It can be in any other organization or even in the personal repository. The only requirement is that the plugin should have an `velox.toml` configuration file and can be build with the latest version of RoadRunner.

Community plugins has their own issues and PRs. If you have any questions or issues with the community plugin, you should create an issue/discussion in the repository of the plugin.

Community plugins can have their own licenses and versioning.

## How to install a community plugin

To install a community plugin, you need to build it using the Velox tool. The Velox tool is a CLI tool that can be used to build RoadRunner with plugins. You can find more information about the Velox tool in the [Customization](../customization/build.md) section.

## List of the community plugins

Here is the list of the community plugins that are available:

| Plugin name              | Repository      | Description    | Docs |
|--------------------------|-----------------|----------------|------|
| SendRemoteFile | [link](https://github.com/roadrunner-server/sendremotefile)| Used to send a file from a remote endpoint into the response | [docs](./sendremotefile.md) |
| CircuitBreaker | [link](https://github.com/roadrunner-server/circuit-breaker)| Circuit breaker pattern implementation for RoadRunner | [docs](./circuit-breaker.md) |
| RFC 7234 Cache | [link](https://github.com/darkweak/souin/tree/master/plugins/roadrunner)| RFC 7234 Cache implementation for RoadRunner | [docs](https://github.com/darkweak/souin?tab=readme-ov-file#roadrunner-middleware) |

Feel free to make a PR to add your plugin to the list.
