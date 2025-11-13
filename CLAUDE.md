# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is the **RoadRunner documentation repository**. RoadRunner is a high-performance PHP application server and process manager written in Go. This repository contains **only documentation** - no application code.

Documentation is organized in GitBook format using Markdown files, structured hierarchically with `SUMMARY.md` as the table of contents and `.gitbook.yaml` as the configuration file.

## Documentation Structure

### File Organization

- **SUMMARY.md** - Main table of contents defining the entire documentation hierarchy
- **README.md** - Landing page with feature overview and badges
- **.gitbook.yaml** - GitBook configuration (root, summary, readme paths)
- **releases/** - Version-specific changelogs (v2024.x.x, v2025.x.x format)
- **intro/** - Getting started guides (about, install, quick-start, config, contributing)
- **php/** - PHP worker configuration and management
- **plugins/** - Core plugins documentation (grpc, centrifuge, server, config, locks, tcp, service)
- **community-plugins/** - Third-party plugins
- **app-server/** - Production deployment guides (nginx, docker, systemd, AWS Lambda)
- **http/** - HTTP server features (headers, static files, gzip, streaming, sendfile)
- **queues/** - Job queue drivers (AMQP, Kafka, SQS, NATS, Beanstalk, Google Pub/Sub, BoltDB, in-memory)
- **kv/** - Key-value storage drivers (Redis, Memcached, BoltDB, in-memory)
- **workflow/** - Temporal.io workflow engine integration
- **lab/** - Observability features (OpenTelemetry, metrics, logging, health checks, Grafana dashboards)
- **customization/** - Development guides (building custom plugins, middleware, embedding, events)
- **integration/** - Framework integrations (Spiral, Laravel, Symfony, Yii, ChubbyPHP)
- **experimental/** - Experimental features list
- **known-issues/** - Common error codes and troubleshooting

### Documentation Conventions

- **File naming**: Kebab-case (e.g., `quick-start.md`, `nginx-with-rr.md`)
- **Internal links**: Relative paths from document location (e.g., `../http/http.md`, `../plugins/grpc.md`)
- **GitBook hints**: Use `{% hint style="info" %}`, `{% hint style="warning" %}` for callouts
- **Code blocks**: Use `{% code %}` / `{% endcode %}` wrappers with optional title attribute
- **Configuration examples**: YAML format (`.rr.yaml`) is preferred, JSON is also supported
- **Version tags**: Features introduced in specific versions marked as `[>=2024.2.0]`
- **Cross-references**: "What's Next?" sections at document end with numbered lists

## RoadRunner Core Concepts

### Architecture Components

1. **PHP Workers** - Long-running PHP processes managed by RoadRunner, persist between requests to eliminate framework bootstrap overhead
2. **Plugins** - Modular components that handle different protocols/features (HTTP, gRPC, Jobs, TCP, Centrifuge, Temporal)
3. **Goridge Protocol** - Communication layer between Go server and PHP workers
4. **RPC Interface** - gRPC-based communication for runtime control (KV access, locks, job management, metrics)
5. **Configuration** - YAML/JSON based (`.rr.yaml`), supports includes, env variables, and CLI overrides

### Request Flow Plugins

These plugins run PHP workers and route specific request types:
- **HTTP** - PSR-7/PSR-17 compatible HTTP/1.1/2/3/fCGI server
- **Jobs** - Queue consumer for various message brokers
- **Centrifuge** - WebSocket event handling via Centrifugo integration
- **gRPC** - gRPC server with protobuf support
- **TCP** - Raw TCP connection handling
- **Temporal** - Workflow/activity execution

### Configuration System

- Default file: `.rr.yaml` in working directory
- CLI options: `-c` (config path), `-w` (working directory), `-o` (override options)
- Supports nested includes: `include: [.rr.sub1.yaml, ../configs/http.yaml]`
- Environment variable substitution: `${VAR_NAME:-default_value}`
- Version header required: `version: "3"`

## Working with Documentation

### Adding New Documentation

1. Create markdown file in appropriate category directory
2. Add entry to `SUMMARY.md` in correct hierarchical position
3. Follow existing file naming conventions (kebab-case)
4. Include "What's Next?" section with related links
5. Use GitBook hint styles consistently with existing docs

### Release Documentation

- Release notes follow format: `releases/vYYYY-M-N.md` (e.g., `v2025-1-4.md`)
- Structure: Title with emoji, Changelog with sections (Core, Velox, specific plugins)
- Update `SUMMARY.md` to add new release at top of Releases section
- Current release branch: `release/v2025.1.5`
- Main branch for PRs: `master`

### Common Documentation Patterns

- **Plugin docs**: Overview, configuration reference, usage examples, RPC methods (if applicable)
- **Driver docs** (queues/kv): Installation, configuration, driver-specific features, limitations
- **Integration docs**: Installation steps, framework-specific worker implementation, configuration examples
- **Customization docs**: Interface definitions, implementation examples, registration/build instructions

### Reference Configuration

Primary config reference: `.rr.yaml` in roadrunner-server/roadrunner repository
Example configs in: `app-server/nginx+RR/.rr.yaml`

## Commands

This repository contains **documentation only** - no build, test, or lint commands apply. Documentation is version-controlled and rendered via GitBook.

To preview documentation locally, refer to GitBook's own documentation system (not part of this repository).

## Key Constraints

- **Dots in names**: Cannot use dots (`.`) in section names, queue names, etc. (separator conflicts)
- **No interactive git**: Avoid `-i` flags (rebase -i, add -i) - not supported in docs workflow
- **GitBook syntax**: Must use GitBook-specific markdown extensions (hints, code blocks with titles)
