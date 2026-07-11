# Zevra Documentation Style Guide

The standards for everything published on the Zevra documentation site. `CONTRIBUTING.md` covers *how to submit* a change; this guide covers *what a good page is*. CI enforces structure (strict builds, link validation); this guide covers what CI cannot check.

---

## 1. The documentation model

Every page is exactly one of five types. The type determines the template, the section it may live in, and the voice it is written in. Mixed-type pages are the primary way documentation rots — when instructions, explanation, and reference share a page, none can be updated safely.

| Type | Reader's question | Template | Typical home |
|---|---|---|---|
| **Tutorial** | "Teach me, I'm new" | `templates/tutorial.md` | Getting Started |
| **How-to** | "Help me do this task" | `templates/how-to.md` | Getting Started, Capabilities, Operations |
| **Concept** | "Help me understand this" | `templates/concept.md` | Platform, Capabilities, AI, Runtime, Architecture, Roadmap |
| **Reference** | "Give me the exact facts" | `templates/reference.md` | API, Operations, AI, Runtime |
| **ADR** | "Why is it like this?" | `templates/adr.md` | ADR section only |

If a page needs two types, it is two pages that link to each other.

## 2. Writing style

- **Present tense, active voice.** "The planner proposes SQL", not "SQL will be proposed by the planner."
- **Second person for the reader** ("you configure"), never "the user" when addressing them.
- **State facts plainly.** No marketing language ("powerful", "seamless", "simply"), no filler openers ("In this document we will…"). Start with the substance.
- **Define terms at first use** and use them identically everywhere afterward. One concept, one name — synonyms in documentation are bugs.
- **Spell out acronyms on first use** per page, except ones the whole site assumes (API, SQL, ADR).
- **Be honest about limits.** Documented behavior includes failure modes, caps, and known constraints. A page that only describes the happy path is incomplete.
- **Write for the section's reader.** Operations assumes an operator under time pressure; Platform assumes a newcomer building a mental model. The same fact can appear in both — worded for each.

## 3. Page structure

- **Exactly one H1**, and it is the page title. Heading levels never skip (H2 → H4 is a build smell even though it builds).
- **An intro paragraph directly under the H1**, before any heading: what this page covers and who it is for. This paragraph doubles as the page's search result context.
- **Frontmatter `description`** on every page — one sentence, used for search previews and social cards.
- **Heading depth stops at H4.** Needing H5 means the page needs splitting.
- **Sentence case for headings** ("Value domains and trust", not "Value Domains And Trust").
- **A page does one job.** When a page answers more than one reader question, split it and cross-link.

## 4. Formatting conventions

- **Code blocks** always declare a language (` ```sql `, ` ```json `, ` ```text ` for plain output).
- **Admonitions** are for genuinely out-of-band content only — `!!! warning` for data-loss/security hazards, `!!! note` sparingly. A page that is half admonitions has a structure problem.
- **Tabs** (`===`) only for true alternatives (OS-specific commands, language-specific clients) — never for sequential steps.
- **Tables** for enumerable facts with uniform columns; prose for anything needing explanation. No walls of prose where a table is honest, and no tables tortured into holding paragraphs.
- **Keys** with `++ctrl+c++` syntax, UI elements in **bold**.

## 5. Diagrams and images

- **Mermaid is the default** for architecture, flows, and sequences — it diffs, it themes, and it cannot go stale in a binary blob. Fence with ` ```mermaid `.
- When a diagram exceeds Mermaid (dense component maps), export to SVG in `docs/assets/images/<section>/` and commit the editable source to `/assets` (repo root) under the same relative name.
- Every image has meaningful alt text.
- Screenshots are a last resort: they rot fastest. Prefer describing the flow; when unavoidable, crop tightly and never embed secrets or real tenant data.

## 6. Files and folders

- **File and folder names**: `kebab-case.md`, short, noun-based; the name should survive a page retitle. No dates, versions, or type prefixes in filenames — except ADRs (`NNNN-slug.md`).
- **Every folder has an `index.md`** — its charter: what belongs, what does not, pointing elsewhere. Material's `navigation.indexes` renders it as the section landing page.
- **Every folder has a `.pages` file** declaring its `title` and `nav` order.
- **Sub-sections** appear when a topic outgrows a single page: create `section/topic/` with its own `index.md` + `.pages`. Depth cap: two folder levels below a top-level section. Deeper nesting means the information architecture is wrong, not the site.
- **Published assets** live in `docs/assets/` (`images/<section>/…`, `stylesheets/`, `javascripts/`); source-of-truth design files live in `/assets` at the repo root.
- **Deleting or moving a page** requires a redirect or updating every inbound link — strict CI will catch internal ones; check external deep links for popular pages before moving them.

## 7. Links

- **Internal links are relative paths to `.md` files** (`../platform/index.md`), never absolute URLs, never `.html` paths. MkDocs validates and rewrites them; absolute URLs silently break on domain or version changes.
- **Link text says where it goes** ("see [Value Domains](…)"), never "click here".
- **External links** point at stable, canonical sources, not blog posts.

## 8. Navigation philosophy

Navigation is federated, not centralized. `mkdocs.yml` deliberately contains no `nav:` — the awesome-pages plugin assembles navigation from per-directory `.pages` files:

- **`docs/.pages`** owns the top-level section order — the site's spine. It changes rarely, and changing it is an information-architecture decision, not a formatting edit.
- **`docs/<section>/.pages`** owns that section's title and page order. Section owners manage their own ordering without touching global config.
- **`- ...` (rest entry)** in a `.pages` nav keeps future pages auto-included alphabetically after the pinned entries — pin what has a deliberate reading order, let reference material sort itself.

Principles:

1. **The top level is fixed and small.** New content lands inside an existing section. A new top-level section is a rare, deliberate act — it must have a distinct reader and a distinct question, and it is proposed in a PR that adds its charter (`index.md`), its `.pages`, and its entry in `docs/.pages` together.
2. **Sections are ordered by reader journey**, not alphabet: orientation → understanding → doing → reference → operating → future.
3. **Navigation shows a section's *shape*, not its inventory.** If a section's nav no longer fits on one screen, introduce sub-sections rather than a longer list.
4. **A page lives in exactly one place.** When two sections both seem right, the section charters decide; if they don't, fix the charters in the same PR.

## 9. Versioning and change

- The site is versioned with mike; `main` publishes continuously as `latest`. Release versioning starts when the platform's own versioning demands it (see `.github/workflows/deploy.yml`).
- Page creation and revision dates come from git history — never write "last updated" dates into page bodies.
- Documentation changes ship with the change they document, in the same review cycle — documentation that trails the product becomes fiction.

## 10. Meta-documentation

This guide, `CONTRIBUTING.md`, `README.md`, and `templates/` live at the repository root, outside `docs/` — they are for authors and are not published to readers. Changing the *standards* (this file, templates, navigation spine) is a platform change: flag it as such in the PR.
