---
description: The ExecutionContract — the immutable, compiled business artifact that Agent Brain produces and the deterministic runtime enforces, so autonomous agents can never invent business objects or physical tables.
---

# ExecutionContract — Agent Business-Object Grounding

Autonomous agents must never invent business objects or physical table names. The
platform guarantees this structurally by separating **business reasoning** from
**deterministic runtime**, with an immutable compiled artifact — the **ExecutionContract**
— as the boundary between them (ADR-0003 A9–A16).

```
User question
   │
   ▼  AgentBrain            business reasoning ONLY (Enterprise Map)
ResolvedBusinessModel       resolved business objects (Brain's output; not the contract)
   │
   ▼  ExecutionContractBuilder   deterministic COMPILER — no repository/service access
ExecutionContract           immutable, fully compiled (SemanticView + ExecutionBindings + identity)
   │                                   │
   │ (prompt path)                     │ (runtime path)
   ▼ PromptContextBuilder              ▼ AgentToolRegistry (runtime)
   (SemanticView + ExecutionBindings)  enforce approvedAssets  →  SqlGovernancePipeline (UNCHANGED)  →  execute
PromptContext → PromptAssembler
   → grounding text
```

## Ownership boundaries

| Component | Owns | Must never |
|---|---|---|
| **AgentBrain** | Enterprise Map lookup, business-object resolution, semantic/synonym/alias interpretation, deciding the execution scope, producing `ResolvedBusinessModel`. | Execute SQL; enforce; build prompts; compile the contract. |
| **ExecutionContractBuilder** | Deterministic compilation only: build `SemanticView` + `ExecutionBindings`, precompute `approvedAssets`, compute `semanticHash`, assign identity. | Query the Enterprise Map, `SemanticService`, or any repository; perform business reasoning. |
| **ExecutionContract** | The immutable compiled business artifact + identity. | Contain business logic, prompt text, or any runtime mutation. |
| **PromptContextBuilder / PromptAssembler** | Deterministically derive a model-agnostic `PromptContext` from the immutable contract — business meaning from `SemanticView`, physical bindings from `ExecutionBindings` — and render grounding text from that `PromptContext` alone. | Interpret business meaning; access the Enterprise Map, `EnterpriseSemanticAssembler`, any repository or persistence, runtime state, or any source other than the contract. |
| **Runtime** (`AgentRunner`, `AgentToolRegistry`) | Validation, deterministic enforcement of `approvedAssets`, governance invocation, execution, auditing, persistence. | Query the Enterprise Map; resolve or interpret business meaning; derive any projection. |
| **SqlGovernancePipeline** | SQL safety, contracts, RLS, masking, routing, row limits, auditing. | — **UNCHANGED** by this work. |

Dependency direction is strictly **Runtime → AgentBrain → (Enterprise Map)**. The runtime
holds no reference to `EnterpriseMapRepository` or `SemanticService`.

## The compiled artifact

`ExecutionContract` (immutable — treat it as a compiled execution plan):

- **Identity** — `contractId`, `createdAt`, `agentId`, `connectionKeys`, `semanticHash`
  (deterministic over the agent and its approved surface). Recorded in the agent session
  trace (`CONTEXT_RESOLVE` step) so every governance run is traceable to the exact contract
  that produced it — for audit, replay, lineage, and future caching.
- **`SemanticView`** (business plane) — `businessObjects`, each composing its
  `BusinessAttribute`s (business name + `AttributeRole`) and `relationships`. Business
  meaning only; **no physical detail**, no prompt text. Read by the prompt path for
  meaning; never by the runtime.
- **`ExecutionBindings`** (execution plane) — `objectBindings` (object → table),
  `attributeBindings` (attribute → physical column), and the **precomputed**
  `approvedAssets` set. Read by the runtime for enforcement (`approvedAssets`) **and** by
  the prompt path for the physical table/column a business attribute grounds to. Targets
  carry a `kind` (`SQL` today) so the same shape extends to REST/GraphQL/warehouse/vector/MCP
  later.

Both planes are read to construct the prompt: `SemanticView` supplies business meaning and
`ExecutionBindings` supplies the physical table/column each business attribute maps to, so
the model is grounded in real columns and never has to invent one. The runtime path reads
`ExecutionBindings` alone.

The contract is **fully compiled**: the runtime consumes `approvedAssets` via pure set
membership (`isApproved(connectionKey, canonicalIdentifier)`) and derives nothing.

## Enforcement

On every `query_database` call, the runtime — after the connection allow-list and
**before** `SqlGovernancePipeline` — extracts referenced tables with the shared
[`SqlTableReferenceExtractor`](#) (which also supplies the canonicalization the builder
used to precompute `approvedAssets`) and requires each to be in the contract's approved
surface. An unapproved reference (an invented or unauthorized business object) yields a
**deterministic, physical** observation the ReAct loop re-plans against; the business
explanation is the model's `final_answer`, grounded by the compiled `ExecutionContract`
(business meaning from its `SemanticView`, physical columns from its `ExecutionBindings`).
`describe_schema`
is likewise scoped to the contract — the runtime never exposes the raw `information_schema`.

Approved queries flow into `SqlGovernancePipeline` and read-only execution exactly as
ADR-0003 A2 — governance is untouched.

## What this replaces

Agents were previously grounded in the raw `information_schema` catalog, with no
business-object resolution and no gate: the model could emit `SELECT COUNT(*) FROM invoices`
against a source with no Invoice object, and the database was the first component to
reject it. Now the Enterprise Map is the sole source of business meaning, the model is
grounded only in approved business objects, and unapproved objects are rejected before
governance as a reasoning outcome — never a database error.

## Invariants

`SqlGovernancePipeline`, `GovernanceOutcome`, `GovernanceAuditService`, governance
semantics, and **ADR-0003 A2** are unchanged. The shared `SqlTableReferenceExtractor` is
the platform's canonical table-reference utility; governance keeps its own private
extractor for now, and its migration to the shared utility is a separate, governance-scoped
story.
