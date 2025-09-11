# Community plugins

## Introduction

Community plugins are plugins that are not officially supported by the RoadRunner team. They are maintained by the community and are not part of the RoadRunner distribution. They can be built using the tool called [Velox](https://github.com/roadrunner-server/velox).

Not being official does not mean they are not good. In fact, some community plugins are very useful and can be used in production.
A community plugin does not necessarily need to be in the `roadrunner-server` organization. It can be in any other organization or even a personal repository. The only requirement is that the plugin has a `velox.toml` configuration file and can be built with the latest version of RoadRunner.

Community plugins have their own issues and PRs. If you have any questions or issues with a community plugin, you should create an issue or discussion in the plugin's repository.

Community plugins can have their own licenses and versioning.

## How to install a community plugin

To install a community plugin, you need to build it using the Velox tool. The Velox tool is a CLI tool that can be used to build RoadRunner with plugins. You can find more information about the Velox tool in the [Customization](../customization/build.md) section.

## List of community plugins

Here is a list of community plugins that are available:

| Plugin name              | Repository      | Description    | Docs |
|--------------------------|-----------------|----------------|------|
| SendRemoteFile | [link](https://github.com/roadrunner-server/sendremotefile)| Used to send a file from a remote endpoint into the response | [docs](./sendremotefile.md) |
| CircuitBreaker | [link](https://github.com/roadrunner-server/circuit-breaker)| Circuit breaker pattern implementation for RoadRunner | [docs](./circuit-breaker.md) |
| RFC 7234 Cache | [link](https://github.com/darkweak/souin/tree/master/plugins/roadrunner)| RFC 7234 Cache implementation for RoadRunner | [docs](https://github.com/darkweak/souin?tab=readme-ov-file#roadrunner-middleware) |

Feel free to make a PR to add your plugin to the list.
