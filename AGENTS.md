# Repository Guidelines

## Project Structure & Module Organization
- Docs are organized by topic directories (e.g., `intro/`, `php/`, `http/`, `plugins/`, `queues/`, `kv/`, `app-server/`, `integration/`, `releases/`). Each contains Markdown pages.
- Navigation is controlled by `SUMMARY.md`; the GitBook entry points are set in `.gitbook.yaml` (`readme: README.md`, `summary: SUMMARY.md`). Update `SUMMARY.md` when adding pages.
- Shared assets live in `resources/` or are referenced via external URLs. Prefer HTTPS links and stable, permalinks when possible.

## Build, Test, and Development Commands
- Local preview: use any Markdown preview (e.g., VS Code) — GitBook builds on push; no custom local build required.
- Lint Markdown (uses `.markdownlint.json`):
  - `npx markdownlint-cli "**/*.md"` or `markdownlint .` if globally installed.
- Optional link check (if you have it installed): `lychee . --verbose --no-progress`.

## Coding Style & Naming Conventions
- Filenames and directories: kebab-case (`example-page.md`), keep section folders flat and topical.
- Headings: sentence case for page titles, concise `##`/`###` sections; avoid deep nesting beyond `###`.
- Code fences: always add language tags (`php`, `go`, `yaml`, `bash`, `json`). Prefer minimal, runnable snippets.
- Lists/indentation: 2 spaces; wrap lines naturally (MD013 disabled) but keep paragraphs short.
- Links: use relative links within the repo; use absolute HTTPS for external resources. Avoid bare URLs in text when anchor text is clearer (MD034 disabled but clarity preferred).

## Testing Guidelines
- Run markdown lint before opening a PR and fix reported issues.
- Validate code/config samples by running or syntax-checking where feasible (e.g., `php -l`, `yamllint`).
- Images: ensure they render in GitBook; prefer externally hosted, permanent URLs or add assets under `resources/`.

## Commit & Pull Request Guidelines
- Commit messages follow a conventional style: `docs(scope): short summary`, `fix:`, `chore:`, `release:`. Reference issues with `[#123]` when applicable.
- PRs must include: purpose, scope of changes, screenshots for visual updates, and affected pages list. Update `SUMMARY.md` when adding/moving pages.
- Keep PRs focused and small; link related issues and releases notes where relevant.

## Agent-Specific Instructions
- Only modify Markdown, `SUMMARY.md`, and assets; do not introduce build tooling or restructure without justification.
- Adhere to `.markdownlint.json`; prefer relative links and language-tagged fences.
- When adding pages, place them in the appropriate topic folder and register them in `SUMMARY.md`.

