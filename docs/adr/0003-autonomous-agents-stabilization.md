---
description: ADR-0003 — Capability Stabilization Decision Record for Autonomous Agents (Zevra Agents): KEEP/STABILIZE/DEFER decisions, the protected agent-as-data model, the accepted production work, the future-evolution posture of the execution runtime, and the exit criteria for declaring the capability STABLE.
---

# ADR-0003: Autonomous Agents — Capability Stabilization Decision

| | |
|---|---|
| **Status** | Proposed (becomes Accepted on approval of the story set in §9) |
| **Date** | 2026-07-11 |
| **Deciders** | Zevra platform team |
| **Scope** | Autonomous Agents (Zevra Agent Runtime) only. The Conversational Analytics pipeline, Semantic Foundation, Executive Brief, Workflow Automation, and Connection Registry are referenced only where a decision requires it. |
| **Inputs** | `capabilities/autonomous-agents.md` (source of truth), the capability inventory, ADR-0002 (Conversational Analytics stabilization), and code verification of the SQL execution path (`AgentToolRegistry`, `DynamicSqlService`). |
| **Relationship to ADR-0002** | ADR-0002 §7 explicitly moved "governing the agent runtime's SQL" to this record; ADR-0002 §11 names Conversational Analytics the shared governed intelligence engine and defers analyst-identity consolidation to the workforce phase. This record is written to be consistent with both. |
| **Supersedes** | — |
| **Superseded by** | — |

Once accepted, this capability is not reopened for architecture review unless implementation changes significantly; changes to these decisions supersede this record, never edit it.

---

## 1. Executive Decision

## **STABILIZE BEFORE PRODUCTION.**

- **Not Stable:** the runtime executes model-authored SQL with **no code-level statement validation and no governance stage**. Code verification sharpened the documented gap: `AgentToolRegistry` passes raw model SQL to `DynamicSqlService.executeQuery`, whose contract comment requires "SQL that has already passed safety + governance checks" — a precondition this caller falsifies. Because JDBC `executeQuery` accepts any statement returning a ResultSet, **PostgreSQL `UPDATE … RETURNING` and CTE-wrapped writes execute successfully** through the platform's nominally read-only path. Combined with zero prompt-injection screening on a loop reachable by routed chat text and raw data values, this is a write-capable injection surface — the platform's single most severe production defect. Secondary blockers: no run-level timeout or cancellation, orphaned `RUNNING` sessions with no reconciliation, and ungated activation.
- **Not Major Rework:** the capability's defining property — **an agent is entirely data** (persona, goal, connections, budget), executed by persona-blind machinery — is exactly the property the AI Workforce vision requires, and it is already true. The loop is deterministic, bounded, cached, salvaged, and fully traced. The defects are a missing enforcement stage, a missing lifecycle, and a missing gate — engineering around a sound core, not a rebuild.

**Foundation verdict (the question this review was asked):** Autonomous Agents is **not** the foundation of the AI Workforce — ADR-0002 already assigned that role to the governed Conversational Analytics engine. Autonomous Agents is the workforce's **analyst identity and packaging layer**: the half of the vision that proves analysts can be configuration instead of code. **The current runtime is accepted for the current product generation** (§4, §11); future architectural evolution may replace the execution runtime while preserving the Agent model defined by this ADR, and any such change requires a superseding ADR. Stabilization makes the runtime safe to operate; it neither entrenches nor pre-commits its replacement.

## 2. Vision Alignment (ownership, not design)

**Does it align with the AI Workforce vision?** Half of it *is* the vision, proven: a Sales Analyst differs from a Claims Agent by rows in `nexus_zevra_agent`, not by Java. No agent-specific logic exists in code. That is the workforce's central product claim already working.

**Can it become the foundation for built-in analysts?** The **agent model** yes; the **agent runtime** no. The runtime is the platform's second reasoning engine — semantically shallow (keyword vocabulary matching, live value sampling instead of governed domains, no resolution layer) and ungoverned (no safety validation, contracts, RLS, masking, or governance audit). Analysts whose answers must carry the platform's receipts cannot stand permanently on this runtime as it is; ADR-0002 assigns the shared-intelligence role to the governed engine, with analyst identities defined here.

**Responsibilities that permanently belong to the Agent (the analyst layer):**
identity — persona, voice, goal; connection/scope grants as tenant-owned data; iteration/effort budgets; lifecycle (`DRAFT`/`ACTIVE`/`ARCHIVED`) under human stewardship; sessions as the per-run record; the management UX.

**Responsibilities that must never belong to the Agent:**
reasoning machinery and its guarantees (engine-owned); governance enforcement (governance-owned); business meaning — vocabulary, domains, bindings (Foundation-owned); connection credentials (registry-owned); scheduling infrastructure (scheduler/brief-owned); delivery channels (delivery-owned).

The runtime today violates none of these ownership lines *except* the first — it embodies its own reasoning machinery. That boundary is governed by the Future Evolution Contract (§11) and ADR-0002's deferred work.

## 3. Protected Architecture

Not to be redesigned without overwhelming evidence:

1. **Agent-as-data.** Persona, goal, connections, budget, status are tenant-owned rows; the runtime is persona-blind and identical for every agent. This is the workforce's enabling property and survives any future runtime replacement unchanged — the analyst definition outlives the runtime that executes it.
2. **Connection allow-lists as the tenant-owned access grant**, validated before execution — the correct permission primitive regardless of which engine executes.
3. **Deterministic context assembly with compiled bounds** — keyword filtering, ranked schema capped at 8 tables, pruned history (question + last 6 tool pairs): prompt cost bounded by construction. (The *sources* feeding assembly change under §4 decisions; the bounded-assembly discipline does not.)
4. **The bounded loop's control furniture** — iteration caps, repeat-call cache with the explicit "already ran this" note, the salvage call, and exactly-one-terminal-state session semantics.
5. **The step trace as the session's complete audit record** — every tool call with inputs, outputs, duration, cache flags; `SCHEMA_LOAD`/`TOOL_CALL`/`FINAL_ANSWER` typed steps. This remains the analyst-session record under any future runtime.
6. **Human-gated activation ownership** — nothing becomes routable or brief-eligible without a human setting `ACTIVE` (the gate gains validation in §5; the human ownership is protected).
7. **The scheduling boundary** — the runtime owns no scheduler; consumers (the brief) schedule. Correct and kept.

## 4. Stabilization Decisions by Area

| Area | Decision | Reason · Business impact · Risk if ignored |
|---|---|---|
| **Agent definition model** (persona/goal/connections/budget as data) | **KEEP (protected)** | §3.1. The workforce's proof of concept, already correct. *Risk if reopened:* destroying the one property the vision depends on. |
| **Investigation Runtime (the bounded reasoning loop)** | **KEEP, stabilized — accepted for the current product generation** | The runtime is sound machinery made unsafe by what surrounds it. Stabilize lifecycle (A3) and inputs (A5); do **not** rebuild. Future architectural evolution may replace the execution runtime while preserving the Agent model (§11); any such change requires a superseding ADR. *Risk if ignored:* two reasoning engines diverge with no recorded architectural boundary between them. |
| **Tool registry & SQL execution** | **STABILIZE — the record's critical work** | No statement validator; write path real (`UPDATE … RETURNING`, CTE writes) via JDBC `executeQuery`; violates `executeQuery`'s own approved-SQL contract; invisible to governance audit. Decision: enforce read-only in code immediately (A1) and route agent SQL through the governance chain (A2). The brief and routed chat inherit whatever posture this seam has. *Risk if ignored:* a routed chat message can steer a model into a successful write against a customer database, unaudited. |
| **Context assembly — vocabulary & schema ranking** | **KEEP** | Bounded, deterministic, adequate for the stabilization horizon. Keyword-matching shallowness is real but its fix (resolution-layer consumption) belongs to future runtime evolution (§11), not a bolt-on. |
| **Context assembly — live status-value sampling** | **STABILIZE** | Sampling live `DISTINCT` values by column-name convention bypasses the Foundation's discovery gates, authority levels, and content classification — the platform re-litigated exactly this problem in the metadata arc. Decision: persisted value domains first, sampling as flagged fallback (A5). *Risk if ignored:* two sources of truth for legal values; convention-matched columns can pull ungoverned (potentially sensitive) values into prompts. |
| **Session model & step trace** | **KEEP (protected) + STABILIZE lifecycle** | The trace is complete and correct; the lifecycle is not: no deadline, no cancel, orphaned `RUNNING` forever (brief's 10-minute timeout protects only briefs). A3 mirrors ADR-0002 S3. *Risk if ignored:* unkillable runs, permanently wedged sessions, thread exhaustion. |
| **Chat router entry** | **STABILIZE (bounded)** | Governance exposure is already gated default-off by ADR-0002 S1 (owned there). Remaining defects owned here: dispatcher cost on every message when enabled (live per-agent catalog fetches — A8 caches them) and injection reachability (A4). *Risk if ignored:* latency/cost tax on every chat message for routing-enabled tenants. |
| **Executive Brief integration** | **KEEP (consumer contract)** | The brief consumes sessions correctly and imposes its own caps. Its fidelity is bounded by this runtime's — every A-story automatically improves the brief. Nothing to change on the brief's side of the seam from this record. |
| **Direct agent chat + management UI** | **KEEP + STABILIZE activation** | CRUD, statuses, sessions UI work. Activation is a bare status edit with no validation while activation makes an agent routable and brief-eligible. Decision: lightweight activation gate (A6). *Risk if ignored:* misconfigured agents silently intercepting chat questions and feeding the morning brief. |
| **Semantic Foundation integration** | **STABILIZE inputs only** | Direct vocabulary reads are acceptable short-term; value domains move to governed sources (A5). Unused injected services (`SemanticService`, `KnowledgeGraphService`) are noted as hygiene, not a story. Full resolution-layer consumption belongs to future runtime evolution (§11). |
| **AI Memory / Alerts / Workflows / Reports integration** | **KEEP (absent) — deliberately** | No integration exists and none is added during stabilization; analyst memory and findings are workforce-phase contracts (§6). |
| **Tenancy & data placement** | **KEEP with documented exception + verification** | Agent/session tables live in `public` keyed by tenant column — an exception to schema-per-tenant. Repository-level filtering with cross-tenant 404s is verified acceptable; migration during stabilization is churn without a defect. Isolation tests (including brief-scheduler tenant-context handling) become exit criteria. |
| **Configuration** | **KEEP compiled bounds, document them** | No `nexus.agent.*` properties; bounds are constants. Acceptable at this maturity; A3 externalizes only what it touches (deadline). A configuration reference entry is part of story DoD, not a separate effort. |

## 5. Production Stabilization Work (accepted)

Eight stories owned by this record, full definitions in §9:

| # | Work | Area | Severity |
|---|---|---|---|
| A1 | Code-level read-only enforcement for agent-authored SQL | Security | Critical |
| A2 | Route agent SQL through the governance chain + governance audit visibility | Governance | Critical |
| A3 | Run deadline, cancellation, orphaned-session reconciliation | Resilience | High |
| A4 | Prompt-injection hardening + steerability fixture suite (message and data values) | Security | High |
| A5 | Persisted value domains for status context (sampling as flagged fallback) | Semantic consistency | Medium |
| A6 | Activation gate: validated transition to ACTIVE | Operational maturity | Medium |
| A7 | Agent runtime observability: session outcomes, orphans, router match rates, per-session cost | Observability | Medium |
| A8 | Router overhead reduction: catalog caching, skip conditions | Performance | Low |
| A9 | Enterprise Map repository support — approved Data Objects by connection keys | Correctness | Medium |
| A10 | ExecutionContract foundation — immutable compiled business artifact + builder | Correctness | High |
| A11 | AgentBrain business-object resolution (Enterprise Map) | Correctness | High |
| A12 | Prompt grounding pipeline (PromptContext/PromptAssembler); retire information_schema grounding | Correctness | High |
| A13 | Canonical SqlTableReferenceExtractor (shared infra) | Correctness | Medium |
| A14 | Runtime ExecutionContract enforcement gate (before governance) | Correctness | High |
| A15 | End-to-end validation of business-object enforcement | Correctness | High |
| A16 | Architecture documentation (ExecutionContract, ownership) | Governance-of-record | Medium |

**A9–A16 (business-object grounding).** A structural correction: autonomous agents performed no business-object resolution and were grounded in the raw `information_schema`, so the model could invent business objects and tables (`SELECT COUNT(*) FROM invoices`) with the database as the first rejector. The correction introduces **Agent Brain** (business reasoning over the Enterprise Map → `ResolvedBusinessModel`), a deterministic **ExecutionContractBuilder** (→ immutable **ExecutionContract** with a precomputed approved-asset surface), a prompt pipeline (**PromptContextBuilder** → **PromptAssembler**) that grounds the model only in approved business objects, and a **runtime enforcement gate** that rejects any table outside the contract **before** `SqlGovernancePipeline`. `SqlGovernancePipeline`, governance semantics, and **A2** are unchanged. See [ExecutionContract](../architecture/execution-contract.md). These IDs continue after A1–A8; no existing story is renumbered.

### External blocking dependencies

| Dependency | Owning record | Why it blocks this capability |
|---|---|---|
| Connection secrets encrypted at rest | **Connection Registry stabilization** | Every agent query authenticates with these credentials. |
| Agent-routing gate in chat (default off) + ungoverned-run marking | **ADR-0002 S1 (Conversational Analytics)** | Controls this runtime's exposure to arbitrary chat traffic until A2 lands. |
| Platform rate limits / token budgets applied to the `agent` usage context and the direct agent-chat endpoint | **ADR-0002 S5 (Conversational Analytics)** | Agent chat is an unmetered LLM entry point today; the budget mechanism is owned there and must cover this surface. |

## 6. Deferred Work (belongs to AI Workforce evolution)

| Item | Why deferred |
|---|---|
| **Unifying agent execution with the governed Investigation Engine** (a single reasoning engine) | A possible future evolution, not a commitment: it requires the engine contract ADR-0002 deferred, and any adoption happens through a superseding ADR (§11). Stabilization work is placed accordingly — A1/A2 harden the *seam* (SQL execution), which is preserved under any runtime evolution; nothing invests in the runtime's internals. |
| **Analyst identity consolidation** (NexusAgent scope/playbooks/KPIs + ZevraAgent persona/UI/sessions → one analyst entity) | Owned by the workforce phase per ADR-0002 §6; the agent-as-data model here is its foundation. |
| **Agent/analyst memory** (sessions that remember prior investigations) | Workforce capability; single-shot sessions are functional for current consumers. |
| **Findings as structured work products** | Shared contract with ADR-0002's deferred findings item; agents emit prose + trace today, which the brief consumes adequately. |
| **Tool registration / per-agent tool selection** | Four compiled tools suffice for current surfaces; extensibility is a workforce contract to design once, under §11's evolution decision, not twice. |
| **Agent collaboration** | No mechanism exists and no current consumer needs one; collaboration arrives via the shared engine and findings contracts, not agent-to-agent plumbing. |
| **Continuous/scheduled autonomous operation** | Requires background run lifecycle (deferred in ADR-0002) plus scheduling contracts; the brief's scheduler covers today's only scheduled need. |
| **Session archival/search/export beyond last-50** | Operational nicety; retention alignment is covered by the platform retention split (ADR-0002 S6) if sessions are included there. |

## 7. Rejected Recommendations (not to be implemented in stabilization)

| Rejected | Why |
|---|---|
| **Rebuilding or replacing the Investigation Runtime now** (planner/reflection stages, multi-agent orchestration) | The runtime is accepted for the current product generation and may be replaced by future architectural evolution (§11); deep investment in its internals ahead of that decision is waste. Stabilization hardens its boundaries only. |
| **Migrating agent/session tables into tenant schemas** | Churn without a demonstrated isolation defect; repository filtering + 404 semantics are verifiable. Recorded as a documented exception instead. |
| **Semantic-resolution integration into agent context assembly** (BLR, entity bindings, knowledge graph) | The governed engine already does this; rebuilding it inside this runtime ahead of the §11 evolution decision builds the same thing twice. A5 takes only the value-domain slice, where dual sources of truth are an active correctness risk. |
| **Removing the unused injected services as a story** | Routine engineering hygiene, not stabilization (same principle as ADR-0002's rejected cleanup batch). |
| **Per-table/column permissions on the allow-list model** | A2 delivers row security and masking through the governance chain — the platform's actual fine-grained model. A parallel permission system on the allow-list would duplicate governance. |
| **Retry mechanisms for model calls / tools / sessions** | Same posture as ADR-0002: bounded single-attempt with honest failure is coherent; retries invite duplicate side effects and mask real failure rates that A7 should instead expose. |

## 8. SaaS Product Assessment

- **Customer value:** real — "package a domain's investigative focus in minutes, reuse it in chat and the morning brief" is a concrete, demonstrable value loop, and the brief is the platform's most executive-visible output.
- **Differentiated:** partially. Configurable agents are table stakes in 2026; **agents whose every step is traced, bounded, and (after A2) governed** are not — the differentiation is inherited from the platform's governance story, which is precisely why A2 is the record's center of gravity.
- **Understandable by business users:** yes — persona/goal/connections is the most business-legible surface in the platform. The *two-agent-systems* confusion (chat routing agents vs. Zevra Agents) is the legibility defect, and its resolution is the deferred consolidation.
- **Would customers trust it?** Not today, knowingly: unvalidated SQL against their databases outside governance audit fails any security review. After A1/A2/A4: yes, with the same receipts story as chat.
- **Commercially valuable:** yes, primarily as the *workforce preview* — this is the surface that lets sales say "your analysts, defined by you, in minutes" while the governed engine matures beneath it.
- **Enterprise-ready:** no (§1); yes upon §10.

## 9. Linear Stories (accepted work only)

**A1 — Enforce read-only agent SQL in code**
*Business Objective:* close the write-capable path from model-authored SQL to customer databases — the platform's most severe production defect.
*Technical Scope:* statement-level safety validation on the agent execution path (reject non-SELECT, including `RETURNING`-bearing DML and writing CTEs); execute agent queries in a read-only transaction as defense in depth; honor `DynamicSqlService.executeQuery`'s approved-SQL contract by validating before the call (a dedicated validated entry point at the seam is acceptable); Workflow Automation's identical exposure is noted to its owning record, not fixed here.
*Acceptance Criteria:* fixture suite — plain `UPDATE`/`DELETE`/`INSERT`, `UPDATE … RETURNING`, `WITH … DELETE RETURNING`, multi-statement — all rejected before execution with the rejection visible in the session trace; plain SELECTs unaffected; suite green.
*Dependencies:* none. *Priority:* Critical. *Estimate:* Small.

**A2 — Route agent SQL through the governance chain**
*Business Objective:* one governance posture for every SQL execution in the platform; the brief and routed chat inherit it automatically.
*Technical Scope:* agent `query_database` execution passes the governance chain — safety validation, row security, masking, row limits, governance audit (contracts where a mapped entity exists; applicability documented) — with each verdict recorded in the session step trace; agent queries appear in governance audit surfaces attributed to agent + session; blocked steps return to the loop as observations (the model can re-plan), mirroring the conversational loop's rejection semantics.
*Acceptance Criteria:* an agent query violating an RLS policy is blocked and both traces (session + governance audit) record it; masked columns return masked in agent results and the brief; every agent query is findable in governance audit; capability page's "does not pass the governance chain" limitation deleted truthfully.
*Dependencies:* A1 (subsumed by the chain's safety stage; A1 ships first as the immediate stopgap). *Priority:* Critical. *Estimate:* Medium.

**A3 — Session lifecycle: deadline, cancellation, orphan reconciliation**
*Business Objective:* bounded, recoverable agent runs; no permanently wedged sessions.
*Technical Scope:* configurable wall-clock deadline enforced across loop iterations → salvage attempt, then terminal state; owner-verified cancel endpoint effective mid-loop; startup reconciliation marking stale `RUNNING` sessions `FAILED`; brief and routed-chat callers behave correctly on each outcome.
*Acceptance Criteria:* slow-run fixture terminates at deadline (salvaged or `FAILED`) with intelligible trace; mid-loop cancel clean; restart reconciles orphans; brief run unaffected by a reconciled session.
*Dependencies:* none. *Priority:* High. *Estimate:* Medium.

**A4 — Prompt-injection hardening and steerability fixtures**
*Business Objective:* measured, hardened resistance on a loop that authors SQL and is reachable by routed chat text and raw data values.
*Technical Scope:* hardened framing of untrusted content (user message, query-result rows, sampled values) in the system prompt and observations; deterministic screening of instruction-shaped content in tool results; hostile fixture suite — instructions embedded in user messages, in status values, and in returned rows — with measured steer rates recorded.
*Acceptance Criteria:* fixture suite demonstrates hostile content does not redirect the loop's goal or its SQL targets (with A1/A2 as backstop, demonstrated in the same tests); flagged content visible in trace; results recorded in the capability page's security section.
*Dependencies:* A1 (backstop must exist before results are meaningful). *Priority:* High. *Estimate:* Medium.

**A5 — Persisted value domains for agent status context**
*Business Objective:* one source of truth for legal status values; the Foundation's discovery gates and authority levels apply to agent prompts.
*Technical Scope:* context assembly reads persisted value domains (authoritative first, observed advisory) for granted-connection columns where they exist; live `DISTINCT` sampling only as fallback, flagged as sampled in the trace; column-name convention list retired where domains cover.
*Acceptance Criteria:* a column with a persisted domain contributes domain values (test); fallback sampling flagged in trace; no live sampling for domain-covered columns.
*Dependencies:* none. *Priority:* Medium. *Estimate:* Small.

**A6 — Activation gate**
*Business Objective:* nothing routable or brief-eligible without passing minimum viability.
*Technical Scope:* transition to `ACTIVE` requires: ≥1 connection granted and reachable, goal/persona present, and one successful smoke session (bounded, e.g. 3 iterations) whose result the activating user sees; failures block activation with reasons; existing ACTIVE agents grandfathered with a flagged warning until re-validated.
*Acceptance Criteria:* activation without connections/goal fails with a clear reason; smoke-run failure blocks with the trace shown; passing agent activates; grandfathering behaves as specified.
*Dependencies:* none. *Priority:* Medium. *Estimate:* Small.

**A7 — Agent runtime observability**
*Business Objective:* operational visibility for a runtime two other capabilities depend on.
*Technical Scope:* metrics — session outcomes by terminal state (incl. forced/`MAX_ITER` rates), orphan reconciliation count, session latency and iteration distributions, router match/fallthrough rates, per-session token cost by agent; exposed via the secured actuator alongside ADR-0002 S4's pipeline metrics.
*Acceptance Criteria:* a `FAILED` session, a reconciled orphan, and a router fallthrough are each observable in metrics without log access; per-agent cost visible; metric names documented.
*Dependencies:* actuator authentication (Authentication / Platform Security record). *Priority:* Medium. *Estimate:* Small.

**A8 — Router overhead reduction**
*Business Objective:* remove the per-message latency/cost tax for routing-enabled tenants.
*Technical Scope:* cache per-tenant agent table catalogs with TTL + invalidation on agent/connection edits; skip dispatcher entirely when no `ACTIVE` agents or routing disabled; measure before/after per-message overhead.
*Acceptance Criteria:* no catalog fetch on cache-warm messages (test); zero dispatcher work when routing is off or no agents are active; measured overhead reduction recorded.
*Dependencies:* none. *Priority:* Low. *Estimate:* Small.

## 10. Exit Criteria — declaring Autonomous Agents **STABLE**

1. **Write-path closed:** the A1 fixture suite (DML, `RETURNING` DML, writing CTEs, multi-statement) rejected before execution, verified against a live PostgreSQL connection.
2. **Governance drill:** an agent query blocked by RLS, a masked column returned masked (in the session and in a brief run), and both events present in governance audit surfaces.
3. **Audit completeness:** every agent-issued query in a sampled week of sessions is findable in governance audit with agent + session attribution.
4. **Lifecycle:** deadline, cancel, and orphan-reconciliation tests green; a kill-mid-run drill leaves no `RUNNING` session after restart.
5. **Injection:** the hostile fixture suite (message, status values, result rows) passes with recorded steer rates and the A1/A2 backstop demonstrated.
6. **Domain conformance:** domain-covered columns draw prompt values from persisted domains (test); all sampling fallbacks flagged in traces.
7. **Activation:** gate tests green; zero un-validated ACTIVE agents remaining (grandfathered set re-validated or flagged).
8. **Isolation:** cross-tenant agent/session access tests green, including identical slugs across tenants and brief-scheduler runs setting/clearing tenant context with no leakage between consecutive tenants.
9. **Observability:** forced drills (failed session, orphan reconciliation, router fallthrough) visible in metrics without log access.
10. **Performance evidence:** router per-message overhead measured before/after A8; session latency/iteration distributions published; per-session token cost by agent visible.
11. **Consumer regression:** Executive Brief and routed chat run green end-to-end against the stabilized runtime; the brief's numbers still reconstruct from session traces.
12. **Checklist closure:** the capability page's stabilization-checklist items covered by A1–A8 are checked with links to tests/evidence; the full backend suite green with zero removed tests.

## 11. Future Evolution Contract

Autonomous Agents is the AI Workforce's **analyst identity and packaging layer** — the proof that analysts are tenant-owned configuration, not code. Future workforce capabilities (pre-built analysts, Executive Advisor, Root Cause Investigator, scheduled investigations) **build on the agent model** — definitions, grants, budgets, lifecycle, session records — which is protected (§3).

**The current execution runtime is accepted for the current product generation.** Future architectural evolution may replace the execution runtime while preserving the Agent model defined by this ADR; any such structural change happens through a superseding ADR, never by incremental drift. Until such a record exists, no capability may deepen its dependence on the runtime's internals (consuming session traces — the brief's pattern — is fine; extending the runtime is not), and workforce capabilities must not add a third reasoning engine under any circumstances.

**Intentionally open architectural decision** — recorded, not answered:

> Can an Analyst contain multiple reusable skills/playbooks (Trend Analysis, Root Cause, Forecasting, KPI Monitoring, …), or should each skill be modeled as an independent Analyst?

This decision is deliberately left open. Future development must not assume either answer — no schema, API, or UI built on the agent model may hard-code one structure before a future ADR decides it.

---

## Consequences

**Positive:** the platform's most severe defect (write-capable ungoverned SQL) is closed in two bounded steps; the brief and routed chat inherit governance without changes of their own; the agent-as-data model is protected as the workforce's identity foundation; the second-engine boundary gets an explicit architectural posture (§11) instead of indefinite drift.

**Negative:** two reasoning engines continue to operate, with A2 making the second one governable; agents remain semantically shallow (keyword matching, no resolution) — accepted because that machinery belongs to the shared engine, not this runtime; Workflow Automation's identical SQL exposure remains open until its own record (noted, not silently inherited).

## Alternatives considered

- **Declare Stable now** — rejected: a write-capable, injection-reachable, unaudited SQL path is disqualifying on its own.
- **Unify execution onto the governed engine during stabilization** — rejected: requires the Investigation Engine extraction ADR-0002 deliberately deferred; coupling this capability's production safety to the roadmap's largest structural change inverts the philosophy. A1/A2 make the runtime safe *now* at a seam preserved under any future runtime evolution.
- **Retire the runtime immediately and route everything through conversational chat** — rejected: destroys the working brief and the agent product surface, and chat's synchronous model cannot host the brief's scheduled runs today.
- **Accept the governance gap as a documented bounded exception** (the checklist's alternative disposition) — rejected: code verification shows the boundary is not read-only in practice, so the exception's premise ("bounded") is false.
