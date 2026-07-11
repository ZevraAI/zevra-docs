# zevra-docs

The official documentation platform for the Zevra AI platform — built with
[MkDocs](https://www.mkdocs.org/) and
[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/), published
to https://docs.zevra.ai.

- **Writing or changing a page?** Start with [`CONTRIBUTING.md`](CONTRIBUTING.md).
- **Wondering what a good page looks like?** [`STYLE_GUIDE.md`](STYLE_GUIDE.md).
- **Starting a new page?** Copy a template from [`templates/`](templates/).

## Local development

Requirements: Python 3.11+ and Git.

```bash
# one-time setup
python -m venv .venv
.venv\Scripts\activate          # Windows
# source .venv/bin/activate     # macOS / Linux
pip install -r requirements.txt

# write with live reload at http://127.0.0.1:8000
mkdocs serve

# the CI gate — run before pushing
mkdocs build --strict
```

`mkdocs serve` rebuilds on save, including `mkdocs.yml` and theme overrides.
`mkdocs build --strict` is exactly what CI runs: broken links, orphaned pages,
and bad navigation entries fail the build.

### Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Config value 'plugins': The "…" plugin is not installed` | Dependencies out of date — `pip install -r requirements.txt` inside the venv |
| Page exists but is missing from navigation | Not listed in the folder's `.pages` and no `- ...` rest entry — see §8 of the style guide |
| Revision dates look wrong locally | The git-revision plugin falls back to build time for uncommitted files; dates are correct once committed (CI checks out full history) |
| Mermaid block renders as a plain code block | Fence must be ` ```mermaid ` exactly; check `pymdownx.superfences` custom fence config in `mkdocs.yml` |

## Repository map

| Path | Purpose |
|---|---|
| `docs/` | All published content — one folder per top-level section, each with an `index.md` charter and a `.pages` nav file |
| `docs/assets/` | Published static assets (images, `extra.css`, `extra.js`, logo) |
| `overrides/` | Material theme template overrides (`main.html` is the seam) |
| `templates/` | Authoring templates — one per page type |
| `assets/` | Source-of-truth design files (diagram sources, brand masters) — not published |
| `scripts/` | Docs automation (conventions in its README) |
| `mkdocs.yml` | Site configuration — contains **no `nav:`**; navigation is assembled from `.pages` files |
| `STYLE_GUIDE.md` | Documentation standards, folder conventions, navigation philosophy |
| `CONTRIBUTING.md` | How changes get proposed, reviewed, and shipped |
| `.github/workflows/` | `ci.yml` (strict build on PRs) and `deploy.yml` (publish on merge to `main`) |

## How the site is organized

Ten fixed top-level sections, ordered as a reader journey — orientation
(**Getting Started**), understanding (**Platform**, **Capabilities**,
**AI Platform**, **Runtime**, **Architecture**), the decision record
(**ADR**), lookup (**API**), operating (**Operations**), and direction
(**Roadmap**). Each section's `index.md` is its charter: what belongs there
and where everything else goes. Content grows *inside* sections; the top
level stays fixed.

## Deployment and versioning

Merging to `main` publishes the site to GitHub Pages via
[mike](https://github.com/jimporter/mike) as the **latest** version
(`.github/workflows/deploy.yml` documents the one-time Pages setup and the
custom-domain step for docs.zevra.ai). When platform releases warrant
versioned docs, deploy releases with
`mike deploy --push --update-aliases <version> latest` — the version selector
is already configured in `mkdocs.yml`.
