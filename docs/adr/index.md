---
description: How Zevra records architecture decisions.
---

# Architecture Decisions

This section is the numbered, append-only record of architecturally significant decisions: choices that are expensive to reverse, constrain future work, or cross component boundaries. [Architecture](../architecture/index.md) describes the current state of the system; ADRs record how it got there and what was traded away.

## When to write an ADR

Write one when a decision:

- is costly or disruptive to reverse,
- constrains how future work must be built,
- resolves a disagreement that would otherwise resurface, or
- changes an ownership contract, trust boundary, or invariant.

Routine implementation choices do not need an ADR. When in doubt, ask: *will someone in two years wonder why this is the way it is?* If yes, record it.

## Conventions

- **Numbering** — sequential, zero-padded to four digits: `0001`, `0002`, …
- **File name** — `NNNN-short-kebab-slug.md` (e.g. `0007-adopt-event-sourcing.md`)
- **Template** — start from `templates/adr.md` in the repository root
- **One decision per record** — if you are writing "and", you are writing two ADRs

## Lifecycle

| Status | Meaning |
|---|---|
| **Proposed** | Under review in an open pull request |
| **Accepted** | Merged; the decision is in force |
| **Superseded** | Replaced by a later ADR, which it must link to |
| **Rejected** | Considered and declined; kept for the record |

**Accepted ADRs are immutable.** To change a decision, write a new ADR that supersedes the old one and update the old record's status and `Superseded by` link — nothing else in it may be edited. Fixing typos and broken links is allowed; rewriting history is not.

## Review

An ADR is proposed, discussed, and accepted through a pull request like any other page. Acceptance requires review by someone who owns or operates an affected component.
