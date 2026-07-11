# Contributing to zevra-docs

How changes reach the Zevra documentation site. What a good page *is* lives in
[`STYLE_GUIDE.md`](STYLE_GUIDE.md); this file covers the workflow around it.

## Ground rules

1. **Everything ships through a pull request.** No direct pushes to `main` —
   `main` deploys to the public site on every merge.
2. **CI is the floor, review is the bar.** `mkdocs build --strict` must pass
   (broken links and orphaned pages fail the build), and at least one reviewer
   approves for accuracy — CI cannot check whether a page is *true*.
3. **Docs ship with the change they document.** If a platform change alters
   documented behavior, the documentation PR is part of that change's
   definition of done, not a follow-up.

## Setup

See the [README](README.md#local-development) — in short: Python 3.11+,
`pip install -r requirements.txt`, `mkdocs serve`.

## Making a change

### Editing an existing page

1. Branch from `main` (`docs/<short-topic>` is the convention).
2. Edit; keep the page true to its type (concept pages don't grow steps,
   how-tos don't grow essays — split instead).
3. Preview with `mkdocs serve`, then gate with `mkdocs build --strict`.
4. Open a PR using the template; state who the affected reader is.

### Adding a page

1. Pick the page type and section using §1 and §8 of the style guide; the
   section's own `index.md` charter confirms the fit.
2. Copy the matching template from `templates/`, name the file in
   `kebab-case.md`, and write.
3. Order it in the section's `.pages` file if position matters (pages after
   `- ...` sort alphabetically on their own).
4. Link it from at least one related page — a page only reachable through
   navigation is a page nobody finds.

### Adding a sub-section

When a topic outgrows one page: create `docs/<section>/<topic>/` with an
`index.md` charter (template: `templates/section-index.md`) and a `.pages`
file. Move the original page's content in; leave links behind.

### Adding a top-level section

Rare and deliberate — see §8 of the style guide. The PR must add the charter,
the `.pages` file, and the `docs/.pages` entry together, and explain what
reader and question the existing sections could not serve.

### Recording a decision (ADR)

Follow [the ADR process](docs/adr/index.md): next number, `templates/adr.md`,
status `Proposed`, reviewed by someone who owns an affected component.

### Changing the platform itself

`mkdocs.yml`, the theme, workflows, templates, and the style guide are
infrastructure. Changes to them affect every author and every page — label
the PR as a platform change and expect review to weigh precedent, not just
the diff.

## Review expectations

Reviewers check, in order:

1. **Accuracy** — is it true of the system as shipped?
2. **Placement** — right section, right page type, per the charters?
3. **Standards** — style guide followed; template contract intact?
4. **Connectedness** — linked from somewhere; links out where readers will
   need them?

Authors: respond to every comment, but push back where you disagree —
documentation review is a technical discussion, not a formality.

## Deployment

Merging to `main` triggers `.github/workflows/deploy.yml`, which publishes to
GitHub Pages via [mike](https://github.com/jimporter/mike) as the `latest`
version. There is no manual publish step, and no way to publish without
merging.
