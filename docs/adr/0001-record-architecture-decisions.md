---
description: ADR-0001 — record architecture decisions as numbered, immutable documents.
---

# ADR-0001: Record architecture decisions

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-07-10 |
| **Deciders** | Zevra platform team |
| **Supersedes** | — |
| **Superseded by** | — |

## Context

Zevra's architecture is governed by explicit contracts — ownership boundaries, trust tiers, invariants — and the reasoning behind them is as important as the contracts themselves. Without a durable record, decisions get re-litigated, context is lost when people move on, and the same trade-offs are re-argued from scratch. Design history scattered across chat threads, commit messages, and memory does not survive.

## Decision

We record every architecturally significant decision as an Architecture Decision Record in this section, following the conventions in [the section charter](index.md): sequential numbering, one decision per record, proposal and acceptance through pull-request review, and immutability after acceptance — decisions are changed by superseding, never by editing.

## Consequences

### Positive

- The reasoning behind the platform's shape survives team changes and time.
- "Why is it like this?" has a linkable answer; settled arguments stay settled.
- Reversing a decision requires engaging with the original trade-offs, not ignoring them.

### Negative

- Writing a record adds friction to significant decisions — deliberately.
- The record is only as good as the team's discipline in writing them; unrecorded decisions are the failure mode to watch for.

## Alternatives considered

- **Design docs only** — richer, but unversioned prose with no lifecycle; they describe intent at a point in time and silently rot.
- **Decision log in a wiki** — editable in place, which invites rewriting history instead of superseding it.
- **No formal record** — the status quo this ADR exists to end.
