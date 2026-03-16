# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is the **official documentation** for [RoadRunner](https://github.com/roadrunner-server/roadrunner) — a high-performance PHP application server and process manager written in Go. The docs are published via [GitBook](https://docs.gitbook.com/) and cover all RoadRunner plugins, configuration, deployment, and integrations.

This is a **pure Markdown documentation repo** — there is no build system, no tests, and no application code.

## Repository Structure

- `SUMMARY.md` — GitBook table of contents; **must be updated** when adding/removing/renaming pages
- `.gitbook.yaml` — GitBook configuration pointing to `README.md` and `SUMMARY.md`
- `.markdownlint.json` — disables MD013 (line length) and MD034 (bare URLs)
- `resources/` — PSD source files for diagrams (not referenced directly by docs)

### Documentation Sections

| Directory | Topic |
|---|---|
| `intro/` | About, quick start, install, config, compatibility, contributing |
| `php/` | PHP workers, pool, RPC, scaling, debugging, environment |
| `http/` | HTTP plugin config, headers/CORS, static files, gzip, streaming |
| `grpc/` | gRPC plugin, proto file registration |
| `queues/` | Jobs plugin + per-driver docs (AMQP, Kafka, NATS, SQS, Beanstalk, BoltDB, Memory, Google Pub/Sub) |
| `kv/` | KV plugin + per-driver docs (Redis, Memcached, BoltDB, Memory) |
| `plugins/` | Other plugins: Centrifuge, Service, Server, Config, Locks, TCP |
| `lab/` | Observability: OpenTelemetry, health checks, access logs, metrics, logger, Grafana dashboards |
| `workflow/` | Temporal.io integration |
| `customization/` | Building custom plugins, middleware, jobs drivers, embedding, events bus |
| `app-server/` | Deployment: Docker, K8s+ArgoCD, Nginx, systemd, Lambda, CLI, production |
| `integration/` | Framework integrations: Spiral, Laravel, Symfony, Yii, ChubbPHP, migration |
| `community-plugins/` | Third-party plugins |
| `experimental/` | Experimental features |
| `known-issues/` | Known issues and error codes |
| `releases/` | Release notes (one file per version) |

## Markdown Conventions

- **GitBook directives** are used throughout: `{% code title=".rr.yaml" %}`, `{% endcode %}`, `{% hint style="info" %}`, `{% endhint %}`, etc. Preserve these when editing.
- YAML config examples use fenced code blocks with `yaml` language tag, often annotated with a title: `` ```yaml .rr.yaml ``
- Cross-links use relative paths: `[text](../section/page.md)`
- Images are referenced via GitHub URLs (not local files)

## Release Notes

Release notes live in `releases/` with naming pattern `v{year}-{major}-{patch}.md` (e.g., `v2025-1-10.md`). The versioning scheme is **CalVer**: `v{YYYY}.{major}.{patch}`.

Format for release notes:
- Title: `# 🚀 v{version} 🚀`
- Section per affected plugin: `### 📦 \`{plugin}\` plugin`
- Core changes: `### 🎯 Core`
- Each entry uses emoji prefixes: ✨ (feature), 🐛 (bug fix)
- Entries link to GitHub issues with `[FR]` (feature request) or `[BUG]` tags

When adding a release, also add its entry to `SUMMARY.md` under the `📚 Releases` section.

## Commit Message Style

Commits follow this pattern:
- `chore: {description}` — routine updates (releases, minor fixes)
- `fix: {description}` — corrections
- `[#{issue}]: {description}` — issue-linked changes
