# Page templates

Start every new page by copying the template matching its type (see §1 of
`STYLE_GUIDE.md` for how to choose):

| Template | Page type | Reader's question |
|---|---|---|
| `tutorial.md` | Tutorial | "Teach me, I'm new" |
| `how-to.md` | How-to | "Help me do this task" |
| `concept.md` | Concept | "Help me understand this" |
| `reference.md` | Reference | "Give me the exact facts" |
| `adr.md` | Architecture Decision Record | "Why is it like this?" |
| `section-index.md` | Section/sub-section charter (`index.md`) | "What lives here?" |

Each template's `<!-- comments -->` explain how to fill it in — **delete them
all before committing**. Headings marked *(optional)* may be removed; the
others are the page's contract with its readers and stay.
