# Source assets

This folder holds **source-of-truth design assets** that are *not* served by the
documentation site:

- Editable diagram sources (`.drawio`, `.excalidraw`, Figma exports)
- Brand source files (logo masters, color specifications)
- Raw images before optimization

**Anything the site serves lives under [`docs/assets/`](../docs/assets/)** —
MkDocs only publishes files inside `docs/`.

Convention: when you export a diagram into `docs/assets/images/`, commit its
editable source here under the same relative name, so the next author can
modify it instead of redrawing it.
