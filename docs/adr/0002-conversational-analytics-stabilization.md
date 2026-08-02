---
description: ADR-0002 — Capability Stabilization Decision Record for Conversational Analytics: the final KEEP/STABILIZE/DEFER decisions, the protected architecture, the accepted production work, and the exit criteria for declaring the capability STABLE.
---

# ADR-0002: Conversational Analytics — Capability Stabilization Decision

| | |
|---|---|
| **Status** | Proposed (becomes Accepted on approval of the story set in §9) |
| **Date** | 2026-07-11 |
| **Deciders** | Zevra platform team |
| **Scope** | Conversational Analytics only. The Semantic Foundation, Autonomous Agents, Scheduled Reports, Executive Brief, and Workflow Automation are referenced only where a decision requires it; their own stabilization records govern them. |
| **Inputs** | `capabilities/conversational-analytics.md`, `architecture/semantic-foundation.md`, the capability inventory, and the two prior reviews (enterprise review CA-1..; AI Workforce review R1–R16/ST-1–15). Findings are cited, not re-argued. |
| **Supersedes** | — |
| **Superseded by** | — |

Once accepted, this capability is not reopened for architecture review unless implementation changes significantly; changes to these decisions supersede this record, never edit it.

---

## 1. Executive Decision

## **STABILIZE BEFORE PRODUCTION.**

- **Not Stable:** four independently blocking production defects — plaintext connection secrets, insecure security defaults (CORS `*`, default JWT secret, public actuator), the ungoverned agent-routing exit at the pipeline's front door, and unbounded synchronous execution with no run deadline or cancellation — plus one unverified potentially-critical exposure (masking in snapshots/traces).
- **Not Major Rework:** the intelligence core — resolution-first assembly, the bounded reasoning loop, per-step governance, the evidence/trace model — survived two adversarial reviews with its ownership boundaries intact and its claims matching implementation. Every blocker is configuration, gating, verification, or bounded engineering *around* an untouched core. The prior workforce review's decomposition test (every envisioned analyst maps onto the existing loop with zero loop changes) stands as the decisive evidence that this architecture is the right foundation.

The accepted work is eight stories owned by this record (§9) plus two **external blocking dependencies** owned by other capability records — connection-secret encryption (Connection Registry stabilization) and platform security defaults (Authentication / Platform Security stabilization) — tracked in §5. On their completion and the exit criteria (§10), this capability is declared **STABLE**.

## 2. Vision Alignment (ownership, not design)

**Does it support the AI Workforce vision? Yes — it is the only candidate.** It is the sole execution path where the Semantic Foundation's full contract and the governance chain are enforced end to end; an analyst whose answers cannot carry receipts is not sellable under this vision, and no other component can produce them.

**Responsibilities that permanently remain here** (the engine's charter):
language resolution consumption and retrieval expansion; context assembly with its budgets; the orchestration decision; the bounded reasoning loop with literal validation; governance-chain invocation per step; evidence, trace, and audit persistence; learning-signal capture; progress streaming.

**Responsibilities that must never live here:**
analyst identity, persona, and voice; domain/connection grants as product objects; playbook authoring and KPI watchlists; schedules, triggers, and delivery channels; work-product composition style; business meaning (Foundation-owned); governance policy (governance-owned); the autonomous agent runtime (routed to, never absorbed).

This split is recorded now so that workforce-era stories build analysts *on* the engine, never *into* it. The engine-surface fusion (the engine is reachable only as a chat call) is acknowledged and **deliberately deferred** (§6) — it is vision enablement, not production risk.

## 3. Protected Architecture

The following are **protected architectural decisions** — not to be redesigned without overwhelming evidence, and specifically not reopenable by future cleanliness arguments:

1. **Semantic Foundation integration as implemented** — resolution runs once, first, after scope; fail-open to a byte-identical baseline; annotation-never-substitution; referents-only targets; provenance tiers in prompts and traces.
2. **Single-assembler context ownership** — one seam decides what every model stage sees, with every section question-filtered and capped; prompt cost bounded by configuration, never tenant size.
3. **The reasoning loop** — plan → literal-validate → govern → execute → evaluate, ≤ 6 steps, rejection-with-reason and bounded replan; the runtime never edits SQL or the question.
4. **The governance chain ordering** on every executed step (safety → route → contract → RLS → mask → limit → audit) and its invocation-not-implementation relationship to this capability.
5. **DLR's enforcement ladder** — reject-once with the legal list, hard-block on authoritative repeat, advisory on observed; enforcement strength follows metadata honesty.
6. **The evidence model and reconstructable trace** — every answer rebuildable from stores alone: routing evidence, resolutions with tiers, per-step SQL/policies/verdicts, snapshot.
7. **The deterministic/AI responsibility split** — AI confined to the five named judgments over offered sets; model output never becomes stored truth without a deterministic gate.
8. **The failure posture** — advisory layers fail open toward the baseline; enforcement layers fail closed. (Stabilization adds *telemetry* to this posture, never changes its direction.)
9. **Schema-per-tenant isolation** with per-request routing for every store this capability writes.

## 4. Stabilization Decisions by Area

| Area | Decision | Reason / Business impact / Risk if ignored |
|---|---|---|
| **Chat runtime (surface: controller, SSE, conversations UI)** | **KEEP** + STABILIZE bounds | Works, differentiates (live trace), and is one surface among future ones. *Impact:* none to customers. *Risk if ignored:* n/a — the stabilized bounds are the run-lifecycle and rate-limit stories, which live at the engine layer. |
| **Context assembly** | **KEEP (protected)** | §3.2. Budget-bounded by construction with executable proofs. *Risk if reopened:* regression of the platform's best-verified property for zero customer value. |
| **Orchestrator** | **KEEP; STABILIZE observability only** | The single-LLM decision design is acceptable *if visible*: its fail-open default can mask outages. Decision: instrument (fail-open counters, decision-mix metrics), do **not** add a deterministic second-guessing layer (§7-R2). *Risk if ignored:* silent quality drift discovered by customers instead of dashboards. |
| **Reasoning engine** | **KEEP (protected); STABILIZE run lifecycle** | The loop is untouched; add deadline, cancellation, and orphaned-run reconciliation around it. *Impact:* investigations become bounded and recoverable. *Risk if ignored:* LB-timeout orphaned runs, thread exhaustion under modest concurrency, no operator recourse mid-run. |
| **Governance integration** | **KEEP chain (protected); STABILIZE the front door** | The chain is complete on this path; the Zevra-Agent dispatch exit falsifies the capability's central promise and is inherited by reports. Decision: per-tenant gate (default off) + explicit ungoverned-run marking in audit and trace. Governing the agent runtime itself belongs to the **Autonomous Agents** stabilization record, not this one. *Risk if ignored:* the first enterprise security review fails the platform's flagship claim. |
| **Conversation model** | **KEEP as the chat abstraction; STABILIZE retention** | Conversations are right for ad-hoc Q&A; the defect is evidence/audit purging on the 3-day conversation clock. Decision: split retention clocks (evidence/sessions/audit ≥ 90d configurable). Work-product abstractions (investigations, findings) are workforce-era (§6). *Risk if ignored:* unreconstructable answers after 3 days — an audit-trail failure by policy. |
| **Evidence model** | **KEEP (protected); STABILIZE = verify masking** | One open verification: snapshots/evidence/SSE must provably store post-masking rows (prior-result answers and UI restore reuse snapshots). *Risk if ignored:* a potential masking bypass shipping unexamined — upgrade to Critical if the test fails. |
| **Memory integration** | **KEEP; STABILIZE fail-open wrap** | Consumption is clean; retrieval is the one advisory stage not wrapped fail-open. Small fix, owned by story S8. *Risk if ignored:* an embeddings outage fails every question instead of degrading. |
| **Charts** | **KEEP as-is; first-class charts DEFERRED** | Auto-rendered tables/charts are adequate for the chat surface. Chart-as-work-product is analyst feature work (§6). *Risk if ignored:* none for production. |
| **Attachments** | **KEEP flow; STABILIZE screening** | The enriched-question separation is correct; the content is an unscreened injection surface. Deterministic screening + hardened framing + steerability fixtures. *Risk if ignored:* documented prompt-injection vector reachable by any uploader. |
| **Reports integration** | **KEEP (tolerated as-is)** | Creator-impersonation is ugly but attributed, audited, and functional; replacing it requires the deferred engine contract. Decision: tolerate, document, revisit at extraction. *Risk if ignored:* none new — the mechanism is already in production shape. |
| **Learning integration** | **KEEP consumption** | Capture/injection on this path conforms to the Foundation's contracts. The promotion review gate and vocabulary tier provenance are **Foundation-record items**, out of this capability's scope; this record notes the dependency only. |

## 5. Production Stabilization Work (accepted)

Only work required for production readiness. Eight stories owned by this record, full definitions in §9:

| # | Work | Area | Severity |
|---|---|---|---|
| S1 | Gate + audit-mark the agent-routing exit (default off) | Governance | Critical |
| S2 | Verify masking in snapshots, evidence, SSE (fix if breached) | Auditability/Security | High |
| S3 | Run deadline, cancellation, orphaned-run reconciliation | Resilience | High |
| S4 | Pipeline observability: fail-open telemetry, decision metrics, job/health indicators | Observability | High |
| S5 | Per-user rate limits + per-tenant token budgets at the ask entry | SaaS readiness | High |
| S6 | Retention split (evidence vs conversations) + async completion notification | Auditability/UX | Medium |
| S7 | Attachment content screening + injection fixtures | Security | Medium |
| S8 | Fail-open wrap for memory retrieval | Resilience | Low |

### External blocking dependencies

Production blockers this capability depends on but does not own. They remain §1 blockers — this capability cannot be declared STABLE until they close in their owning records:

| Dependency | Owning record | Why it blocks this capability |
|---|---|---|
| Connection secrets encrypted at rest | **Connection Registry stabilization** | Every investigation executes against customer databases through these credentials; plaintext secrets fail any enterprise security review of this capability's data path. |
| Platform security defaults — JWT boot-gate, CORS allowlist, actuator authentication | **Authentication / Platform Security stabilization** | The ask endpoint and SSE stream ship behind these defaults, and S4's metrics are exposed via the actuator and must not be public. |

Load evidence is an **exit criterion** (§10), not a story: measure first; tune (pool size, budgets) only on evidence.

## 6. Deferred Work (valuable, intentionally postponed)

| Item | Why deferred |
|---|---|
| **Investigation Engine extraction** (headless `InvestigationRequest → InvestigationResult` contract) | The workforce keystone — but vision enablement, not production risk. Deferral is *safe by construction*: everything in §5 lands at or below the future seam (deadline/cancel/budgets/metrics all attach to the engine, not the chat shell), so extraction later is additive and behavior-preserving. First story of the workforce phase. |
| **Analyst identity consolidation** (NexusAgent/ZevraAgent convergence) | Requires the Agents capability record and the extraction above; consolidating before the engine contract exists would consolidate onto the wrong seam. |
| **Structured findings as engine outputs** | The workforce's work-product primitive; no production consumer exists yet. Depends on extraction. |
| **Workforce APIs / analyst framework / pre-built analyst content** | Product evolution, explicitly out of stabilization scope. |
| **Background execution with run handles** | The scheduling half of the run lifecycle; S3 delivers the production-safety half (deadline/cancel). Continuous analysts need this; chat does not. |
| **Clarification linkage** | UX polish; single-shot clarifications are functional. |
| **First-class charts** | Analyst work-product feature. |
| **Agent-concept renaming in UI** | Cosmetic churn ahead of the consolidation decision; S1's ungoverned-answer marking delivers the trust-relevant part now. |
| **PRODUCT_VISION.md** | Author it at workforce-phase kickoff, where it governs real stories — not as stabilization paperwork. |

## 7. Rejected Recommendations (from the prior reviews — not to be implemented in stabilization)

| Rejected | Why |
|---|---|
| **Deterministic cross-check on the orchestrator decision** (prior R-CA-A2, guardrail half) | Adds a second decision system to police the first — complexity with unproven correctness gain, and it erodes protected decision §3.7 (AI judges; runtime verifies *proposals*, not judgments). The accepted remedy is telemetry (S4): make fail-open and decision-mix visible, then decide on evidence. |
| **`ChatService` decomposition / stage extraction** | Churn on the platform's most load-bearing class for zero behavior change; the assembler-ownership concentration is a protected decision. Revisit only at Investigation Engine extraction, which restructures the seam for a *product* reason. |
| **Hikari pool resizing / preemptive performance tuning** | No load evidence exists. Measurement is an exit criterion; tuning without it is guesswork. |
| **Governing the agent runtime's SQL as part of *this* capability's stabilization** | Correct work, wrong record: the runtime is out of scope here. Moved to the Autonomous Agents stabilization review — this record's S1 makes the boundary safe from this side regardless. |
| **Memory-chunk and data-value injection screening** | Defer pending the steerability measurements S7 produces on the attachment vector; screening every context source without evidence of steerability is speculative hardening. |
| **Orchestrator/pipeline retry mechanisms** | The no-run-retry posture is coherent (bounded replans exist inside the loop); adding run-level retry invites duplicate side effects (learning capture, audit) for marginal value. |

## 8. SaaS Product Assessment

- **Customer value:** yes — governed answers with receipts to questions asked in the tenant's own language; the honest-failure modes are a value statement, not a limitation.
- **Differentiation:** real and defensible — per-step governance, deterministic resolution with provenance, and reconstructable audit are claims generic enterprise chat (and the Genie/Cortex class) does not make. This is the demo that closes enterprise deals: *watch the reasoning, open the receipts*.
- **Would customers trust it?** After S1/S2/S4: yes — trust is exactly what the trace-first design manufactures, provided routed-out answers are labeled (S1) and the masking guarantee is proven (S2).
- **Does it strengthen the AI Workforce future?** It *is* that future's engine; §2 records the ownership so the workforce builds on it. Its receipts are what will make unattended analysts sellable.
- **Commercially valuable / would it help sell Zevra?** Yes, with one honest caveat recorded for the workforce phase: the empty-chat entry experience undersells the engine (the vision's own critique). Selling "an AI workforce" on a blank text box is a positioning problem — a *product* problem, deliberately not a stabilization one.
- **Enterprise-ready?** Not today (§1 blockers); yes upon §10.

## 9. Linear Stories (accepted work only)

Connection-secret encryption and platform security defaults are **not stories in this record** — they are external blocking dependencies owned by the Connection Registry and Authentication / Platform Security stabilization records (§5).

**S1 — Gate and audit-mark the Zevra Agent routing exit**
*Business Objective:* make the governed pipeline's governance claim true per tenant; end silent ungoverned inheritance by reports.
*Technical Scope:* per-tenant `agent-routing.enabled` (default **off**); routed runs marked "answered outside the governance chain" in run evidence, reasoning trace, and the chat UI answer; report execution routes with dispatch disabled unless a report opts in.
*Acceptance Criteria:* disabled tenant → dispatcher never invoked (test); routed runs carry the marker end-to-end; default report runs unaffected by agent matches; capability page's governance section matches the shipped behavior.
*Dependencies:* none. *Priority:* Critical. *Estimate:* Small.

**S2 — Verify masking in snapshots, evidence, and SSE**
*Business Objective:* prove governed data cannot leak through stored or streamed artifacts (prior-result answers and UI restore reuse snapshots).
*Technical Scope:* trace masking-application vs. snapshot/evidence/SSE capture points; regression tests asserting masked values in all three; relocate capture if pre-masking.
*Acceptance Criteria:* executable tests prove masked-at-rest/streamed for snapshot, evidence, SSE; prior-result answers reason over masked rows only.
*Dependencies:* none. *Priority:* High (Critical if breached). *Estimate:* Small.

**S3 — Run lifecycle: deadline, cancellation, orphan reconciliation**
*Business Objective:* bounded, recoverable investigations; no orphaned or unkillable runs.
*Technical Scope:* configurable run deadline enforced across loop iterations → graceful `FAILED` + SSE close; owner-verified cancel endpoint effective mid-loop; startup reconciliation marking stuck `RUNNING` runs `FAILED`. (Background execution handles: deferred.)
*Acceptance Criteria:* slow-run fixture terminates at deadline with intelligible trace; cancel ends a mid-loop run cleanly; restart reconciles orphans; defaults documented.
*Dependencies:* none. *Priority:* High. *Estimate:* Medium.

**S4 — Pipeline observability**
*Business Objective:* make silent degradation visible — the operational precondition for trusting a fail-open architecture in production.
*Technical Scope:* Micrometer metrics: decision-type mix, per-decision latency, fail-open activations (resolution, orchestrator default, literal-validator error), governance/literal block rates, SSE failures, background-job outcomes; health indicators for the AI client and async queue; exposed via the secured actuator.
*Acceptance Criteria:* forced resolver failure, forced orchestrator failure, and a literal hard-block each observable as metrics without log access; a failed nightly job visible; metric names documented.
*Dependencies:* actuator authentication (Authentication / Platform Security record, §5). *Priority:* High. *Estimate:* Medium.

**S5 — Rate limits and token budgets**
*Business Objective:* cost governance and abuse protection on LLM-metered entry points.
*Technical Scope:* per-user request rate limit and per-tenant daily token budget enforced at ask-time using existing usage data; clear 429/limit responses; per-tenant configurability; report/brief identities budget under their tenant.
*Acceptance Criteria:* limits enforce and reset correctly under test; limit events visible in usage metrics; defaults documented.
*Dependencies:* none. *Priority:* High. *Estimate:* Small.

**S6 — Retention split + async completion notification**
*Business Objective:* audit-grade evidence retention independent of chat ephemerality; heavy governed queries stop dead-ending.
*Technical Scope:* separate retention clocks — run evidence, reasoning sessions, executions ≥ 90d configurable; conversations keep their window; async completion posts into the conversation and the notification bell (existing mechanisms).
*Acceptance Criteria:* purge test: conversations deleted, evidence retained within its window; an async result appears in-conversation and in the bell; retention matrix documented.
*Dependencies:* none. *Priority:* Medium. *Estimate:* Small.

**S7 — Attachment content screening**
*Business Objective:* close the documented prompt-injection vector.
*Technical Scope:* deterministic screening (instruction-shaped content detection, length/structure caps) + hardened prompt framing of attachment blocks; steerability fixture suite (hostile attachments) with measured results.
*Acceptance Criteria:* hostile fixtures no longer alter planner behavior; flagged content visible in trace; steerability results recorded.
*Dependencies:* none. *Priority:* Medium. *Estimate:* Small.

**S8 — Fail-open wrap for memory retrieval**
*Business Objective:* an embeddings/memory outage degrades one advisory context section instead of failing every question — closing the one advisory stage that violates the protected failure posture (§3.8, §4 Memory integration).
*Technical Scope:* wrap memory retrieval in the same advisory fail-open used by the other context sections; degradation logged and counted in the S4 fail-open metrics.
*Acceptance Criteria:* forced retrieval-failure test — the run completes with no-memory context and the answer is produced; the fail-open activation is visible in metrics; no behavior change when retrieval is healthy.
*Dependencies:* none (its metric lands with S4). *Priority:* Low. *Estimate:* Small.

## 10. Exit Criteria — declaring Conversational Analytics **STABLE**

All criteria are objective and testable. When every box is checked, the inventory's coverage row flips (Architecture Reviewed ✓, Enterprise Reviewed ✓, Stabilized ✓) and this capability is closed to further architecture review absent significant implementation change.

1. **Secrets (external dependency, Connection Registry record):** direct database inspection finds zero plaintext connection secrets; rotation procedure executed once successfully.
2. **Defaults (external dependency, Authentication / Platform Security record):** deployed environment verified — boot-fails on default JWT secret; CORS allowlist active; actuator authenticated.
3. **Governance boundary:** with routing disabled (default), a suite proves the dispatcher is never invoked; with it enabled, routed runs carry the ungoverned marker in evidence, trace, and UI.
4. **Masking:** regression tests assert masked rows in snapshots, evidence, and SSE payloads; prior-result answers verified against masked snapshots.
5. **Run lifecycle:** deadline, cancel, and orphan-reconciliation tests green; no code path can hold a request thread past the configured deadline.
6. **Observability drills:** three forced-failure drills (resolver down, orchestrator down, literal hard-block) each visible in metrics within one scrape interval, without log access.
7. **Budgets:** rate-limit and token-budget enforcement tests green, including reset behavior and report/brief attribution.
8. **Retention:** purge test demonstrates conversations deleted / evidence retained per the split clocks; async completion notification observed in-conversation.
9. **Injection:** the hostile-attachment fixture suite passes; results recorded.
10. **Load evidence:** a recorded load test — ≥ 25 concurrent live-data investigations on a reference tenant — completes within deadlines with no pool exhaustion; p50/p95 latency by decision type published in the operations docs. (Tuning decisions, if any, are made from this evidence.)
11. **Regression:** the full backend suite green with zero removed tests; the capability page's stabilization-checklist items covered by S1–S8 and the external dependencies in §5 are checked with links to their tests/evidence.
12. **Audit reconstruction:** one production-shaped answer reconstructed end-to-end from stores alone (routing evidence → resolutions/tiers → step SQL/policies/verdicts → snapshot) as a documented exercise.

## 11. Future Evolution Contract

Conversational Analytics is the **shared governed intelligence engine** for future AI Workforce capabilities — AI Analysts (Sales, Inventory, Finance, Vendor, Promotion), the Executive Advisor, the Root Cause Investigator, Scheduled Investigations, Executive Dashboards, and future analyst personas.

These future capabilities **may consume and extend this engine**: invoke it, layer analyst identity, scope, playbooks, scheduling, and composition on top of it, and add new surfaces over it. They **must not modify the protected architecture defined in §3**. Extension happens above the engine's contracts, never inside them; the ownership charter in §2 draws the line.

Any structural change to the protected architecture requires a **future ADR that supersedes this record**. Absent such a record, a workforce capability that cannot be built without altering §3 is, by definition, mis-scoped.

---

## Consequences

**Positive:** production risk is reduced to a bounded, sequenced story set; the platform's best architecture is explicitly protected from future re-litigation; the workforce vision gets a recorded ownership charter and a safe deferral path whose stabilization work all lands beneath the future engine seam.

**Negative:** reports continue to consume the engine via impersonation until the workforce phase; the two-agent-system confusion persists (labeled but unresolved) until the Agents capability record; deferring the engine extraction means the next capability reviews (Reports, Brief) will each record a dependency on it.

## Alternatives considered

- **Declare Stable now** — rejected: four blockers are objectively disqualifying for enterprise SaaS.
- **Major rework toward the workforce architecture during stabilization** — rejected: it would couple production readiness to the largest structural change on the roadmap, invert the review philosophy, and discard the safe-deferral property established in §6.
- **Stabilize the agent runtime within this record** — rejected: scope discipline; S3 secures this capability's boundary regardless of the Agents record's timeline.
