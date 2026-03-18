# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Documentation site for the BDesk REST API (an ITSM/service-desk platform). This repo contains **no application code** — only MkDocs Material documentation, an OpenAPI 3.1 spec, and a Postman collection. The site is published to GitHub Pages at `https://bdeskapp.github.io/api`.

All documentation is written in **Brazilian Portuguese (pt-BR)**.

## Build & Serve Commands

```bash
# Install dependencies (Python 3.12+)
pip install mkdocs-material

# Local development server with live reload
mkdocs serve

# Production build (same as CI) — run before pushing to catch errors
mkdocs build --strict
```

The `--strict` flag is used in CI and will fail on warnings (broken links, missing pages, etc.). Always run `mkdocs build --strict` locally before pushing to avoid failed deployments.

## Deployment

Pushing to `main` triggers `.github/workflows/deploy-docs.yml`, which builds with `mkdocs build --strict` and deploys to GitHub Pages via `actions/deploy-pages`. No manual deploy needed.

## Architecture

- `mkdocs.yml` — Site configuration, nav structure, theme, and markdown extensions
- `docs/` — All source markdown files, organized by nav structure:
  - `docs/index.md` — Landing page
  - `docs/guias/` — How-to guides (creating requests, attachments, workflow actions, etc.)
  - `docs/referencia/` — Reference docs (errors, pagination, glossary, limits)
  - `docs/exemplos/` — Code examples (cURL, Python, PowerShell)
- `docs/openapi.yaml` — OpenAPI 3.1.0 spec (72 endpoints), served as a downloadable asset and rendered via Redoc
- `docs/redoc.html` — Standalone HTML page that renders the OpenAPI spec using Redoc CDN (not managed by MkDocs)
- `docs/postman_collection.json` — Postman collection for the API
- `site/` — Built output (gitignored — do not edit directly)

## Key Conventions

- The nav hierarchy in `mkdocs.yml` is the source of truth for page organization. Keep it in sync when adding/removing pages.
- Markdown extensions in use: admonition, details, superfences, highlight, tabbed, tables, attr_list, md_in_html. Use these features freely in docs.
- The API base URL pattern is `https://{cliente}.bdesk.com.br/askrest` with a per-customer subdomain variable.
- The API uses a custom response envelope `RetornoApiRest<T>` with `_metadata` and `records` fields. The login endpoint is a special case returning `RetornoApi<T>` with an escaped JSON string in `Dados`.

## Gotchas

- **Redoc ↔ OpenAPI coupling**: `docs/redoc.html` loads the spec via relative path `spec-url="openapi.yaml"`. If `openapi.yaml` is renamed or moved, the Redoc page breaks silently (no build-time check).
- **redoc.html is not in mkdocs nav**: It's a raw HTML file copied to `site/` during build as a static asset — MkDocs doesn't process or validate it.
- **Postman collection is manual**: `docs/postman_collection.json` is not auto-generated from the OpenAPI spec. Changes to endpoints must be reflected in both files.
