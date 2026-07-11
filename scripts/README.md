# Scripts

Automation for the documentation platform (link audits, reference generation,
asset optimization). None exist yet; when adding one, follow these conventions:

- **Idempotent** — safe to run twice.
- **Runnable from the repo root** — `python scripts/<name>.py`, no `cd` required.
- **Python preferred** — the docs toolchain already requires it; avoid adding a
  second runtime. If a script needs extra packages, add them to
  `requirements.txt` with a comment.
- **Wired into CI, not memory** — if a script guards quality (e.g. a link
  audit), call it from `.github/workflows/ci.yml` so it cannot be forgotten.
- **Documented** — a docstring or header comment stating purpose, inputs, and
  an example invocation.
