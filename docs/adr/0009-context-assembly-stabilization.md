---
description: ADR-0009 — Platform Integration Architecture Stabilization for Context Assembly: the reverse-engineered architecture by which all capabilities contribute the final LLM context, its de-facto owner (ChatService), the protected assembly order / service-boundary / fail-open / zero-cost model, the accepted production work (uniform fail-open seam, total-context budget guard + observability, documentation), the deferred component extraction, and the exit criteria for declaring the integration architecture STABLE.
---

# ADR-0009: Context Assembly — Platform Integration Architecture Stabilization

| | |
|---|---|
| **Status** | Proposed (becomes Accepted on approval of the story set in §9) |
| **Date** | 2026-07-11 |
| **Deciders** | Zevra platform team |
| **Scope** | The **integration architecture** by which platform capabilities contribute context to an LLM request — not any single capability. Covers assembly order, ownership, token budgeting, fail-open posture, and prompt construction across `ChatService` and the context-producing services. The individual capabilities (AI Memory, Semantic Foundation, Knowledge Graph, Enterprise Map, Baseline/Anomaly, Learning) are governed by their own records (ADR-0002/0006/0007/0008) and referenced here only for their contributor role. |
| **Inputs** | Code verification of the assembly path — `chat/ChatService` (`ask`, `buildContextSummary`, `assembleEntityContext`, `filterGraphContext`, `truncateEntityContext`, `composeAnswer`, `answerFromMemory`), and the contributor services it calls (`BusinessLanguageResolver`, `DocumentMemoryService`, `EnterpriseMapService`, `SemanticService`, `KnowledgeGraphService`, `BaselineService`, `LearningContextBuilder`), plus `ReasoningEngine`. **There is no documentation of the context assembly architecture as a whole — the architecture was reverse-engineered from code, which is the source of truth.** |
| **Relationship to prior ADRs** | ADR-0006, ADR-0007, and ADR-0008 each forward-referenced "a future Context Assembly ADR" that would define how the engine composes context from its governed sources. **This is that record.** It also finds that the assembly it describes *already exists implicitly inside `ChatService`* — the same engine-surface fusion ADR-0002 identified from the chat side and deferred (its Investigation Engine extraction). This record governs the integration architecture as it is; it does not build the extraction ADR-0002 defers. |
| **Supersedes** | — |
| **Superseded by** | — |

Once accepted, this integration architecture is not reopened for review unless implementation changes significantly; changes to these decisions supersede this record, never edit it.

---

## 1. Executive Decision

## **STABILIZE BEFORE PRODUCTION.** *(The integration architecture exists and is coherent; it is implicit, not isolated; it needs two bounded guards and a written record — not a redesign.)*

Reverse-engineering the implementation yields a clear and, in most respects, **sound** integration architecture: a single owner (`ChatService`) gathers context from each capability *through that capability's own service*, assembles it in a **fixed, legible order**, spends an explicit budget on the most expensive block (table schema), and renders optional blocks only when non-empty (the zero-cost guarantee). Most contributors fail open. This is a real architecture, and it works. But code verification surfaced two **architecture-level gaps** and one **record gap** that are production-relevant because they govern what every LLM request sees:

- **Fail-open is a per-contributor decision, not an assembler contract (architecture defect).** The gather seam (`ChatService.ask`, lines 281–294) calls each contributor directly: BLR resolution (line 281, fails open — verified), memory retrieval (line 284, **does not** fail open — verified, ADR-0006 M1), enterprise map (289), semantic (291), findings (293), and anomaly (294) — the latter four **unguarded**. Any one of them throwing fails the entire question before the model is ever called. The assembler has no uniform rule that a single contributor's failure degrades to its empty block; it inherits whatever posture each contributor happens to have. (CA1)
- **No component owns the total token budget (missing architectural responsibility).** Budget is *distributed*: each contributor self-caps (BLR ≤ 8 resolutions, memory top-K 6, graph keyword-filtered, history last-4/600-char), and only the **table-schema** block has an explicit char budget (`maxEntityContextChars`, default 1500, via `truncateEntityContext`). Nothing reconciles the *sum*. The assembled prompt's total size is an emergent property of independently-capped parts, unmeasured and unbounded in aggregate. (CA2)
- **The architecture is undocumented (record defect).** No page describes the assembly order, the contributor set, the budget model, or the fail-open contract; three prior ADRs pointed at a Context Assembly record that did not exist. A load-bearing integration architecture governed by no written record cannot be stabilized or safely evolved. (CA3)

**Not a redesign — and the isolation gap is already deferred, not owed here.** The assembly logic being *entangled inside `ChatService`* rather than a named `ContextAssembler` component is the engine-surface fusion ADR-0002 already identified and deferred to its Investigation Engine extraction. This record does **not** build that extraction (the brief forbids proposing a new architecture); it governs the architecture that exists and closes the two guards and the record that make it production-safe *as-is*.

**Position verdict (the question this review was asked):** Context Assembly is a **platform orchestration responsibility that already exists, owned de-facto by `ChatService`**. It is the seam where every capability's contribution becomes the single context the model reasons over. Its permanent identity: the orchestration layer owns *order, total budget, fail-open enforcement, cross-component transformation, and the prompt-construction seam*; each capability owns *only its own block and that block's internal cap*. §11 fixes that division permanently.

## 2. Vision Alignment (the integration architecture, reverse-engineered)

Permanent architectural positions, answering the eighteen questions of record. All answers are from code.

1. **What components contribute context?** In gather order: agent scope (routing); attachment text (prepended into `enrichedQuestion`, `raw` untouched); **BLR resolutions**; **DLR literal candidates**; **knowledge-graph JOIN guidance** (keyword-filtered); **enterprise-map TABLE SCHEMA** (ranked + truncated); **semantic context + bindings**; **prior findings**; **anomaly/baseline context**; **prior-result snapshot** (flag); **conversation history** (last 4 turns); and on the reasoning path, **agent playbook** and **learned vocabulary** (`LearningContextBuilder`). Live-data results feed answer composition, not planning context.
2. **In what order are they assembled?** Fixed in `buildContextSummary`: agent header → RESOLUTIONS → LITERAL CANDIDATES → filtered graph → TABLE SCHEMA → supporting markers (memory count, semantic-available, findings count, anomaly, prior-available) → conversation history; then the reasoning path appends playbook and learned vocabulary. Deterministic, hardcoded.
3. **Which component owns orchestration?** `ChatService` — specifically `buildContextSummary` plus the `ask` pipeline. Its own comment (lines 958–962) states it: "The assembler (this class) owns 'which metadata does the planner see': it holds the question, every context block, and the token budget." It is the de-facto Context Assembler.
4. **Which components merely contribute information?** Memory, Semantic, Knowledge Graph, Enterprise Map, Baseline/Anomaly, Findings, Learning, Playbook — each produces a pre-rendered block through its own service and decides nothing about order or total budget.
5. **Which components transform context?** The assembler (`ChatService`): `filterGraphContext` (keyword + expansion-token filtering of graph context), `assembleEntityContext`/`rankEntityBlockKeys` (reordering schema blocks by relevance using graph join-tables + semantic bindings + expansion tokens), `truncateEntityContext` (budget truncation with a provenance marker). BLR transforms the question into annotations (never substitution). Enterprise Map renders per-table blocks (question-agnostic).
6. **Which components validate context?** `LiteralValidator` (in the reasoning loop, *after* assembly) validates literal choices against persisted domains; the write-intent gate validates intent. **Nothing validates the assembled context itself** (its size, completeness) — the CA2 gap.
7. **Which components must fail open?** Every advisory contributor. Verified today: BLR fails open, graph fails open, learning swallows its own errors; **memory does not** (ADR-0006 M1), and the enterprise/semantic/findings/anomaly gather calls are unguarded. The assembler *as a whole* must fail open — CA1 makes that a contract.
8. **Which are mandatory vs optional?** Only the TABLE SCHEMA block (or its explicit "NO LIVE DATA SOURCES" fallback) is always emitted. Every other block is optional-additive (rendered only when non-empty — the zero-cost guarantee). No contributor is *architecturally* a hard dependency; the defect is that unguarded contributors can still fail the run despite being optional in principle (CA1).
9. **Which capabilities have direct dependencies?** `ChatService` depends on all contributor services and `ReasoningEngine`. Cross-component: schema-block ranking consumes graph join-tables *and* semantic bindings *and* BLR expansion tokens together — the assembler is where those three meet.
10. **Which boundaries are respected?** Contributors are consumed only through their owning services; the assembler never reaches into a contributor's store. Each capability owns its block's internal cap. Attachment is kept separate from `raw` (annotate-never-substitute extends across the pipeline).
11. **Which boundaries are violated?** (a) Memory does not honor the fail-open posture the assembler assumes (CA1). (b) Assembly is entangled in `ChatService` with no isolation — deferred to ADR-0002's extraction, not a new violation. (c) Answer-composition paths (`composeAnswer`, `answerFromMemory`) assemble context *divergently* from `buildContextSummary` — duplicated assembly logic (recorded; consolidation deferred).
12. **Where does prompt construction actually occur?** In multiple `ChatService` methods (`buildContextSummary` for planner/orchestrator context; `composeAnswer`, `answerFromMemory`, `answerFromPrior` for answer prompts; the orchestrator-decision wrapper) and in `ReasoningEngine` (the per-step planner prompt). It is **spread, not centralized**.
13. **Where is context ordering decided?** Hardcoded in `buildContextSummary`'s sequence of appends.
14. **Where are token-budget decisions made?** Per contributor (self-caps) plus `maxEntityContextChars` for the schema block only. **No global budget decision exists** (CA2).
15. **Which architectural responsibilities are missing?** A total-context budget owner; a uniform fail-open contract; a single prompt-construction seam; assembly observability (what blocks were present, total size, which contributors failed open).
16. **Which responsibilities are duplicated?** Context assembly across `buildContextSummary` and the divergent answer-composition methods; prompt construction across several `ChatService` methods and `ReasoningEngine`.
17. **Which responsibilities belong permanently to the orchestration layer?** Assembly order; total-budget reconciliation; fail-open enforcement; cross-component transformation (graph filtering, schema-block ranking); the prompt-construction seam; provenance/trace of what was assembled.
18. **Which responsibilities belong permanently to individual capabilities?** Producing their own context block and enforcing that block's internal cap. A capability owns its *content* and *self-limit* — never the order, never the total budget, never another capability's block.

## 3. Protected Architecture

Not to be redesigned without overwhelming evidence — these are the properties that make the implicit architecture sound:

1. **Service-boundary consumption.** The assembler gathers each contribution *through the owning capability's service*, never by reaching into its store. Verified across all contributors. This is what keeps context assembly an integration layer rather than a second implementation of every capability. Protected.
2. **A single fixed, legible assembly order.** One deterministic sequence in one place (`buildContextSummary`); not per-tenant, not pluggable, not model-decided. The order is explainable and stable. Protected.
3. **Distributed self-capping under a single ceiling.** Each contributor bounds its own block; the assembler owns the schema budget and (via CA2) the total ceiling. Prompt cost is bounded by construction and configuration, never by tenant size — the property ADR-0002 protects, here at the integration level. Protected.
4. **Fail-open / zero-cost posture.** Absent or failed advisory contributors produce no block, and the pipeline degrades to a smaller prompt — never a failed answer. Verified for BLR/graph/learning; CA1 extends it to the whole gather seam. Protected.
5. **Annotate-never-substitute across the pipeline.** `raw` is immutable; attachments and resolutions sit *beside* the question; the model reads the user's words verbatim. Verified (`enrichedQuestion` is separate; `resolved` annotates). Protected.
6. **The orchestration layer owns integration; capabilities own content.** The division in §2.17/§2.18 is the constitutional boundary — order/budget/fail-open/transformation are the assembler's; blocks and their caps are the capabilities'. Protected.

## 4. Stabilization Decisions by Area

| Area | Decision | Reason · Business impact · Risk if ignored |
|---|---|---|
| **Assembly ownership** (`ChatService` as de-facto assembler) | **KEEP; extraction DEFERRED** | A single owner holding question + blocks + budget is correct integration design. Its *isolation* into a named component is ADR-0002's deferred Investigation Engine extraction — not built here, not a production blocker. *Risk if reopened now:* redesigning the platform's most load-bearing class against the brief's explicit prohibition. |
| **Assembly order** (fixed sequence in `buildContextSummary`) | **KEEP (protected)** | §3.2. Deterministic, legible, single-sourced. *Risk if reopened:* a configurable/pluggable order trades explainability for flexibility no consumer needs. |
| **Service-boundary consumption** | **KEEP (protected)** | §3.1. Each contribution flows through its owning service. *Risk if reopened:* the assembler re-implementing capability internals — the anti-pattern this boundary prevents. |
| **Fail-open posture** (per-contributor today) | **STABILIZE — the record's headline** | Architecture defect: the assembler assumes advisory contributors fail open, but the gather seam is unguarded and memory throws (ADR-0006 M1). Decision: a uniform fail-open contract at the gather seam — any single contributor failure degrades to its empty block, never fails the run (CA1). *Risk if ignored:* one contributor's transient error takes down every question. |
| **Token budget** (schema-only; no total) | **STABILIZE** | Missing responsibility: no component reconciles total assembled size. Decision: measure the assembled total and apply a final ceiling with a provenance marker (reusing the existing truncate-with-marker pattern), plus assembly observability (CA2). *Risk if ignored:* prompt bloat as contributors grow, unmeasured, discovered as latency/cost/context-overflow in production. |
| **Cross-component transformation** (graph filter, schema-block ranking) | **KEEP** | Correctly lives in the assembler — it is precisely the integration logic that needs all blocks at once (graph join-tables + semantic bindings + expansion tokens ranking schema blocks). Verified. Not a capability's job. |
| **Prompt construction** (spread across methods + ReasoningEngine) | **KEEP behavior; consolidation DEFERRED** | The divergent answer-composition paths duplicate assembly, but consolidating them is refactoring toward the deferred extraction. Recorded as a finding; not a stabilization story (no behavior defect, only duplication). |
| **Attachment handling** (separate from `raw`) | **KEEP (protected)** | §3.5. Attachment text is per-conversation context injected as `enrichedQuestion`, never merged into the resolved/annotated question. Correct. |
| **Contributor fail-open specifics** (memory) | **KEEP; owned elsewhere** | Memory's own fail-open is ADR-0006 M1 / ADR-0002 S8; CA1 is the *seam-level* backstop above it. Both land together (memory stops throwing; the seam tolerates any contributor that still does). |
| **Assembly observability** | **STABILIZE** | No metrics on which blocks were present, total assembled size, or per-contributor fail-open activations. Folded into CA2. *Risk if ignored:* the integration layer is unobservable — silent degradation and budget drift. |
| **Documentation** (none exists) | **STABILIZE** | The architecture is undocumented and three ADRs pointed at a record that did not exist. Decision: author the Context Assembly architecture page; this ADR governs it (CA3). *Risk if ignored:* an unwritten load-bearing architecture cannot be safely stabilized or evolved. |

## 5. Production Stabilization Work (accepted)

Three stories owned by this record, full definitions in §9:

| # | Work | Area | Severity |
|---|---|---|---|
| CA1 | Uniform fail-open gather seam (any single contributor failure degrades to empty) | Resilience | High |
| CA2 | Total-context budget guard + assembly observability | Resilience/Cost/Observability | Medium |
| CA3 | Document the context assembly architecture | Governance-of-record | Medium |

### External blocking dependencies

| Dependency | Owning record | Why it relates |
|---|---|---|
| Memory retrieval stops throwing (contributor-level fail-open) | **ADR-0006 M1 / ADR-0002 S8** | CA1 is the seam-level backstop; the memory contributor's own fix is owned there. They land together. |
| Actuator authentication for metrics exposure | **Authentication / Platform Security stabilization** | CA2's assembly metrics ship through the secured actuator. |

### Deferred, not owed here

| Item | Owning record |
|---|---|
| Extraction of a standalone `ContextAssembler` / headless engine (isolating assembly from `ChatService`) | **ADR-0002 (Investigation Engine extraction)** — the brief forbids proposing it here; recorded as the natural home for the isolation and prompt-construction consolidation. |

## 6. Deferred Work (belongs to platform evolution)

| Item | Why deferred |
|---|---|
| **Standalone Context Assembler component** (isolate assembly from the chat surface) | This is ADR-0002's Investigation Engine extraction; building it is redesign, explicitly out of this brief. Every CA story here lands at or below that future seam (fail-open guard, budget ceiling, and metrics all attach to the assembly logic, which the extraction later relocates intact). |
| **Consolidating the divergent prompt-construction paths** (`composeAnswer`/`answerFromMemory` vs `buildContextSummary`) | Refactoring toward the same extraction; no behavior defect today, only duplication — consolidate when the assembler is isolated, not before. |
| **Non-chat consumers assembling context directly** (reports, agents) | Reports inherit assembly by replaying chat; agents bypass it (ADR-0003). A shared assembly contract for them is the extraction's payoff, deferred with it. |
| **Configurable / per-tenant assembly order or weighting** | No consumer needs it; a fixed order is protected (§3.2). Revisit only with evidence, via a superseding ADR. |
| **A context-provenance block in the trace** (what each block contributed to the answer) | Valuable for explainability; CA2 delivers the *presence/size* metrics, the richer per-answer provenance is an enhancement gated on the extraction. |

## 7. Rejected Recommendations (not to be implemented in stabilization)

| Rejected | Why |
|---|---|
| **Redesign assembly into a pluggable pipeline / context-rules engine / DSL** | The brief forbids redesign; a fixed legible order is protected (§3.2) and no consumer needs configurability. Complexity for its own sake. |
| **Build the standalone Context Assembler now** | It is ADR-0002's deferred extraction; building it here inverts the brief ("do not propose a better architecture") and couples this record's production safety to the roadmap's largest structural change. |
| **Make any advisory contributor a hard dependency** (fail-closed when a block is absent) | The opposite of the fail-open guarantee (§3.4); a missing block must shrink the prompt, never fail the answer. |
| **Move context ordering or budgeting into the individual capabilities** | Violates the ownership division (§2.17/§2.18): capabilities own their block and its cap, never the order or the total budget. |
| **Let the model decide assembly order or which blocks to include** | Order and inclusion are deterministic orchestration; the model reasons over the assembled result, it does not assemble. |
| **Reconcile the divergent answer-composition paths by forcing them through `buildContextSummary`** | Premature: those paths assemble a *different* prompt (answer vs plan); unifying them belongs to the extraction, not a point fix that risks answer-quality regressions. |

## 8. SaaS Product Assessment

- **Customer value:** invisible but foundational — this is the machinery that makes every answer draw on the tenant's meaning, documents, relationships, schema, and history *together*, in one coherent prompt. Its quality is the ceiling on answer quality.
- **Differentiated:** the *integration discipline* is the moat — governed, provenance-tagged, fail-open assembly of many tenant-owned sources is what competitors approximate with ad-hoc prompt stuffing. The value is that it is ordered, bounded, and (after CA2) observable.
- **Would customers trust it today?** The output is trustworthy, but the *resilience* is not yet: one contributor's transient failure can fail a question (CA1), and prompt size is unmeasured (CA2). After CA1/CA2: yes.
- **Strengthens the platform:** it is the join point of every foundation and capability the prior ADRs stabilized; each of those records made its own block trustworthy, and this record makes the *composition* of them resilient and legible.
- **Enterprise-ready:** not quite (§1 resilience + record gaps); yes upon §10. The architecture is sound; the gaps are guards and documentation, not design.

## 9. Linear Stories (accepted work only)

**CA1 — Uniform fail-open gather seam**
*Business Objective:* no single context contributor's failure can fail a question — the assembler degrades to a smaller prompt, always.
*Technical Scope:* guard each contributor call in `ChatService.ask`'s gather (lines ~281–294 — resolution, memory, enterprise map, semantic, findings, anomaly) and each block-producing call in `buildContextSummary` so a thrown exception is caught, logged, counted (CA2 metric), and yields that contributor's empty block; the assembly and the run proceed with whatever blocks succeeded; coordinates with ADR-0006 M1 / ADR-0002 S8 (memory's own fix) as the seam-level backstop above them; no behavior change when all contributors succeed.
*Acceptance Criteria:* a forced failure in each of memory, enterprise map, semantic, findings, and anomaly independently still yields a completed answer with that block absent and a recorded fail-open activation (tests); all-succeed path is byte-identical to today; no contributor failure surfaces as a failed run.
*Dependencies:* coordinates with ADR-0006 M1 / ADR-0002 S8. *Priority:* High. *Estimate:* Small.

**CA2 — Total-context budget guard + assembly observability**
*Business Objective:* the assembled prompt's total size is measured and bounded, and the integration layer is observable.
*Technical Scope:* after assembly, measure the total assembled context size and apply a final configurable ceiling that truncates with an explicit provenance marker (reusing the `truncateEntityContext` pattern, applied to the whole rather than only the schema block), preserving the highest-priority blocks in assembly order; metrics via the secured actuator — which blocks were present/absent per request, total assembled chars (distribution), per-contributor fail-open activations (from CA1), and truncation events.
*Acceptance Criteria:* an oversized assembly is truncated at the ceiling with the marker and the highest-priority blocks retained (test); block-presence, total-size, fail-open, and truncation metrics are observable without log access; the ceiling is configurable and documented; normal-size assemblies are unchanged.
*Dependencies:* CA1 (fail-open counts feed the metrics); actuator auth (external). *Priority:* Medium. *Estimate:* Small.

**CA3 — Document the context assembly architecture**
*Business Objective:* the platform's integration architecture has an accurate written record the prior ADRs can point to.
*Technical Scope:* author a Context Assembly architecture page capturing the contributor set, the fixed assembly order, the ownership division (orchestration vs capability, §2.17/§2.18), the budget model (per-contributor caps + schema budget + the CA2 total ceiling), the fail-open contract (CA1), and where prompt construction occurs; cross-link the capability records (ADR-0006/0007/0008) whose forward-references this record resolves; note the deferred extraction (ADR-0002).
*Acceptance Criteria:* the page exists and matches the implementation (verified against `buildContextSummary` and the gather seam); the ADR-0006/0007/0008 "future Context Assembly ADR" references resolve to this record; the ownership division and fail-open/budget contracts are stated.
*Dependencies:* CA1/CA2 (documents the contracts once they exist). *Priority:* Medium. *Estimate:* Small.

## 10. Exit Criteria — declaring the Context Assembly architecture **STABLE**

1. **Uniform fail-open proven:** an independently forced failure in each gather-seam contributor (memory, enterprise map, semantic, findings, anomaly, resolution, graph) yields a completed answer with that block absent and a recorded fail-open activation; no contributor failure produces a failed run.
2. **Total budget bounded:** an oversized assembly is truncated at the configured ceiling with a provenance marker, retaining the highest-priority blocks in order; total assembled size is measured and within the ceiling on a representative corpus.
3. **Assembly observable:** block-presence, total-size distribution, per-contributor fail-open, and truncation metrics are each visible without log access.
4. **Order intact:** a test asserts the fixed assembly order (resolutions → literals → graph → schema → supporting → history) and that resolution-free / contributor-absent questions produce the byte-identical baseline prompt (the zero-cost guarantee at the assembly level).
5. **Service-boundary intact:** a code audit confirms the assembler consumes every contribution through the owning service and reaches into no contributor's store.
6. **Annotate-never-substitute intact:** `raw` is immutable through assembly; attachments and resolutions annotate rather than rewrite; verified at the prompt level.
7. **Ownership division recorded and honored:** no capability decides assembly order or total budget; the assembler owns both; verified against §2.17/§2.18.
8. **Isolation deferral recorded:** the entanglement with `ChatService` is documented as ADR-0002's deferred extraction, with every CA guard confirmed to land at/below that future seam (no rework required by the extraction).
9. **Documentation reconciled:** the Context Assembly architecture page exists and matches code; the prior ADRs' forward references resolve here; the divergent prompt-construction paths are recorded as a deferred consolidation.
10. **Regression:** the full backend suite is green with zero removed tests; a gather → assemble → prompt round-trip is verified for each decision type (memory, prior, clarification, knowledge-gap, live-data, hybrid).

## 11. Future Evolution Contract

Context Assembly is a **platform orchestration responsibility** — the seam where every capability's contribution becomes the single context the model reasons over. Today it is owned by `ChatService`; it is an integration architecture, not a capability with its own stores, and it introduces no reasoning of its own (it orders, bounds, and renders what capabilities produce; the model reasons over the result).

Three standing constraints, violable only by a superseding ADR:

1. **The orchestration layer owns integration; capabilities own content.** Assembly order, total budget, fail-open enforcement, cross-component transformation, and the prompt-construction seam belong permanently to the orchestration layer. Each capability owns only its own block and that block's internal cap — never the order, never the total budget, never another capability's block.
2. **Deterministic, fail-open, annotate-never-substitute, bounded-by-construction.** The assembly order is fixed and legible; advisory contributors fail open to a smaller prompt; the user's question is never rewritten; prompt cost is bounded by configuration and caps, never by tenant size. The model reasons over the assembled context; it never decides the assembly.
3. **Consume through services, never through stores.** Every contribution flows through its owning capability's service. The assembler must never re-implement or reach past a capability to build a block.

When ADR-0002's **Investigation Engine extraction** lands, the assembly logic this record governs moves — intact — from `ChatService` into that headless component, and the extraction ADR supersedes this record's ownership clause (the *contracts* — order, budget, fail-open, service-boundary — carry forward unchanged). Until then, this record is the governing description of how context is assembled, and no capability may bypass it, duplicate it, or assume a block it did not contribute.

---

## Consequences

**Positive:** the platform's integration architecture is, for the first time, described and governed by a written record that the prior three ADRs can point to; the one behavior that can fail a question for a transient contributor error is closed with a uniform fail-open seam; total prompt size becomes measured and bounded; and the permanent division between orchestration-owned integration and capability-owned content is fixed so context assembly cannot drift into a pluggable engine, a reasoning layer, or a place capabilities reach past their own services.

**Negative:** the architecture remains entangled in `ChatService` until ADR-0002's extraction — CA1/CA2 harden it in place, which is correct now but means the isolation debt persists (documented, deferred, not owed here); the divergent answer-composition paths remain duplicated until that extraction; and CA1 depends on the memory contributor's own fail-open fix (ADR-0006 M1 / ADR-0002 S8) landing alongside the seam-level backstop.

## Alternatives considered

- **Declare Stable now** — rejected: an unguarded gather seam lets a single transient contributor error fail every question, and total prompt size is unmeasured — both are production-relevant for the machinery behind every answer, even though the assembled output is correct.
- **Build a standalone Context Assembler component as this record's work** — rejected: that is ADR-0002's deferred Investigation Engine extraction, and the brief explicitly forbids proposing a new architecture; CA1/CA2 make the existing architecture safe in place, at a seam the future extraction relocates intact.
- **Treat the missing documentation as out of scope** — rejected: three prior ADRs forward-reference a Context Assembly record; a load-bearing integration architecture governed by no written record cannot be stabilized, so the documentation is a deliverable, not a footnote.
- **Redesign the assembly order into a configurable/weighted pipeline** — rejected: no consumer needs it, the fixed order is protected for explainability, and configurability trades legibility for flexibility that would itself need governing.
- **Consolidate every prompt-construction path immediately** — rejected: the answer and plan prompts are deliberately different; unifying them risks answer-quality regressions and belongs to the extraction, not to stabilization.
