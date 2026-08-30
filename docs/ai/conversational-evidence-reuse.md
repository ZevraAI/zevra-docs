---
description: The rule governing when a follow-up question may reuse a prior turn's query result instead of running a new one, and why "answerable" is not the same test as "correct result set."
---

# Conversational evidence reuse

When a follow-up question arrives in an existing conversation, Zevra can either reuse the previous turn's query result or run a new query. This page documents the correctness rule that decision must follow, and the defect it fixes.

## The rule

> Previous evidence may be reused only when it satisfies the current request's result-set requirements. If the follow-up introduces or changes a filter, grouping, ordering, aggregation, limit, or other result-set requirement, the previous unmodified result set must not be treated as sufficient merely because an answer can be computed from it.

Two different questions are involved, and both must be true before reusing prior evidence:

1. **Answerable** — can this question be answered, at all, using the evidence already gathered?
2. **Correct result set** — are the rows already gathered actually the rows this question is asking for?

An answer can be computed from evidence that is nonetheless the wrong result set — for example, counting how many rows in an unfiltered 12-row result have a given status does not make that same 12-row result correct to display when the user asked to see only the rows with that status. Both checks must pass, not just the first.

## Where this is decided

`ReasoningEngine.reason()` seeds a conversation's prior `ExecutionReference` into the current turn's `EvidenceStore` as a synthetic "step 0," then asks `ReasoningEvaluator.evaluate()` whether that seeded evidence is sufficient. A `SUFFICIENT` (or `DEAD_END`) verdict skips the planning loop entirely — `ReasoningPlanner` is never invoked, no new SQL is generated or executed, and the seeded rows become the turn's displayed result. A `NEED_MORE_DATA` verdict lets the loop run as normal, invoking `ReasoningPlanner` for a new, targeted query.

This is a single LLM judgment, made once per turn that has prior evidence to seed — no additional model call was introduced to enforce the rule above; the existing evaluator call was made to reason about both checks instead of only the first.

## The defect this fixes

Before this fix, `ReasoningEvaluator`'s `SUFFICIENT` criterion asked only whether the evidence was about the right subject and could produce a substantive answer — the "correct result set" check did not exist as a distinct, required condition. A follow-up on the *same* subject (e.g. "purchase orders" in both turns) that asked for a different result set (e.g. "only the submitted ones") could be marked `SUFFICIENT` against the prior turn's full, unfiltered rows: an LLM could read through those rows, compute the right count or describe the right subset, and produce an accurate-sounding natural-language answer — while the actual rows returned to the user remained the original, unfiltered set. The prose and the displayed table disagreed.

This was not a defect in Persistent Knowledge, native OpenAI File Search, `previous_response_id` conversation chaining, Stage 1 concept selection, Decision Router, or SQL generation — all of those were confirmed, by source tracing, to have no path into this decision. The defect was entirely in how "sufficient" was defined for evidence carried over from a prior turn.

## What changed

A prose-only fix was tried first: restating the two-part test above directly in `ReasoningEvaluator`'s `SYSTEM_PROMPT`, still asking the model for a single holistic `decision` field. Live-validated directly against the real model (no fakes, the actual production method, seeded with data shaped exactly like the reported conversation), this was **not reliably followed** — the model still returned `SUFFICIENT` for "I want only submitted" and "What about the closed ones?", and its own `rationale` repeated precisely the conflation the prose forbade (e.g. *"...which is already present in the result set"*).

The fix that actually holds, live-validated the same way: the model is asked to answer the two checks as **separate, explicit fields** rather than folding both into one label —

```json
{
  "resultSetMatches": true,
  "decision": "SUFFICIENT | NEED_MORE_DATA | DEAD_END",
  "rationale": "one sentence explaining your decision"
}
```

— and `ReasoningEvaluator.evaluate()` adds one small, deterministic clamp: if the model reports `resultSetMatches: false`, `decision` is forced to `NEED_MORE_DATA` in code, regardless of what `decision` value the model also produced. This is not a keyword rule or a second LLM call — it reads one boolean the model already computed as part of the same, single evaluator call, and never inspects the question or evidence text itself. An absent `resultSetMatches` (e.g. a degraded/malformed response) is treated as unknown and never triggers the clamp, so behavior for such a response is unchanged from before this fix.

`ReasoningEngine` gained one observability log line, `CONVERSATION_EVIDENCE_REUSE` (conversation id, the evaluator's final decision, and whether the prior result set was reused — never the question text or the evaluator's rationale, both of which may carry business/user data), so a real request's reuse-vs-reexecute decision is visible after the fact without inspecting a database row.

## What did not change

- Reuse of evidence that genuinely still answers the question — a count, a total, a description, or an explanation of the same rows already gathered — is preserved. Only the definition of "sufficient" changed; the mechanism (seed as step 0, ask the evaluator, skip the loop on `SUFFICIENT`/`DEAD_END`) is unchanged.
- No additional LLM call. The same single `ReasoningEvaluator.evaluate()` call per seeded turn now reasons about both checks instead of one.
- Persistent Knowledge, native File Search, `previous_response_id` chaining, Stage 1's output contract, Decision Router, Planner's SQL-generation responsibilities, and the SQL governance pipeline are all unchanged — this defect and its fix sit entirely inside `ReasoningEvaluator`'s prompt and `ReasoningEngine`'s existing seed/skip logic.

## Relationships

- [Persistent tenant knowledge (Stage 1)](persistent-tenant-knowledge.md) — the concept-selection step this page's defect was initially (incorrectly) suspected to involve; source tracing confirmed it has no path into the evaluator's decision.
- `ExecutionReference` / execution continuity — the mechanism that carries a prior turn's result into the current turn's `EvidenceStore` as seeded evidence; unchanged by this fix.
