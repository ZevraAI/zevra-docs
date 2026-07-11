# zevra-docs

Documentation site for the Zevra platform, built with [MkDocs](https://www.mkdocs.org/) and the [Material theme](https://squidfunk.github.io/mkdocs-material/).

## Local development

```bash
pip install -r requirements.txt
mkdocs serve
```

The site is served at http://127.0.0.1:8000 with live reload.

## Structure

| Path | Purpose |
|---|---|
| `docs/` | All documentation content (Markdown) |
| `overrides/` | Material theme template overrides |
| `assets/` | Images, diagrams, and other static assets |
| `scripts/` | Build and maintenance scripts |
| `mkdocs.yml` | Site configuration and navigation |
| `.github/workflows/` | CI — build and deploy the site |

## Build

```bash
mkdocs build
```

Output is written to `site/` (not committed).
