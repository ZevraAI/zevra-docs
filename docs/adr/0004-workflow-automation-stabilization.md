---
description: ADR-0004 — Capability Stabilization Decision Record for Workflow Automation: KEEP/STABILIZE/DEFER decisions, the protected workflow-as-data and deterministic-traversal model, the accepted production work, the execution-surface identity guardrail, and the exit criteria for declaring the capability STABLE.
---

# ADR-0004: Workflow Automation — Capability Stabilization Decision

| | |
|---|---|
| **Status** | Proposed (becomes Accepted on approval of the story set in §9) |
| **Date** | 2026-07-11 |
| **Deciders** | Zevra platform team |
| **Scope** | Workflow Automation (Automations) only. Conversational Analytics, Autonomous Agents, the Semantic Foundation, and the Connection Registry are referenced only where a decision requires it. |
| **Inputs** | `capabilities/workflow-automation.md` (source of truth), the capability inventory, ADR-0002 and ADR-0003, and code verification of the execution path (`WorkflowExecutionEngine`, `DbQueryExecutor`, `VariableResolver`, `DynamicSqlService`). |
| **Relationship to prior ADRs** | ADR-0002 established Conversational Analytics as the platform's governed intelligence engine; ADR-0003 established Autonomous Agents as the analyst identity layer, not a second intelligence foundation. This record positions Workflow Automation against both: it is the **execution surface**, and it must never become a third place where business reasoning lives. |
| **Supersedes** | — |
| **Superseded by** | — |

Once accepted, this capability is not reopened for architecture review unless implementation changes significantly; changes to these decisions supersede this record, never edit it.

---

## 1. Executive Decision

## **STABILIZE BEFORE PRODUCTION.**

- **Not Stable:** code verification confirms the capability page's worst-case reading. `DbQueryExecutor` string-substitutes `{{variable}}` values — including values arriving in **unauthenticated public webhook payloads** — directly into SQL text, then executes through `DynamicSqlService.executeQuery`, the same seam ADR-0003 proved accepts `UPDATE … RETURNING` and CTE-wrapped writes. The composite is an **unauthenticated SQL-injection path into customer databases, on a nominally read-only platform, invisible to governance audit**. Secondary blockers: slug-as-only-secret on a public endpoint with tenant selection by unauthenticated header; no run deadline (the schema's `TIMEOUT` status is never set), no cancellation, orphaned `RUNNING` executions; no graph validation at save or activation; unbounded execution-table growth; silent-misclassification condition semantics.
- **Not Major Rework:** the architecture underneath is correct and mirrors the platform's best pattern. **A workflow is entirely data** — a tenant-owned JSONB graph executed by graph-agnostic machinery. Traversal is deterministic; AI never chooses the next node; branching is owned by explicit `CONDITION` operators; AI appears only as bounded, inspectable steps (draft-only generation at design time, leaf judgments at run time); every run is replayable evidence. The blockers are input handling, enforcement, lifecycle, and gating around a sound core.

**Position verdict (the question this review was asked):** Workflow Automation is the AI Workforce's **execution and action surface** — the hands, not the mind. It owns *doing* (system-to-system flows, lookups, assessments-as-steps, structured responses), never *deciding what is true about the business* (the governed engine's job) and never *being someone* (the analyst layer's job). It is **not** a reasoning engine today — single-path deterministic traversal with AI as a step is categorically different from a loop where a model chooses actions — and §11 makes never-becoming-one a standing constraint.

## 2. Vision Alignment (ownership, not design)

Answers of record to the framing questions:

1. **Permanent responsibility:** deterministic execution of tenant-authored, event-triggered operational flows, with AI judgment available as bounded, inspectable steps — the workforce's sanctioned path from *insight* to *action*.
2. **Business reasoning or only execution?** Execution only. `AI_REASON` is a leaf judgment over resolved inputs inside a human-authored graph — an assessment step, not reasoning ownership. Investigation, resolution, and answer synthesis belong to the governed engine (ADR-0002).
3. **Should workflows make business decisions?** Deterministic branching on explicit conditions — yes, that is the product. Open-ended business judgment — only as an explicit `AI_REASON` step whose output a human-authored graph routes. A workflow must never *own* a decision that requires governed investigation; it may *carry* one.
4. **Should workflows generate SQL?** At run time, **never** — runtime SQL is tenant-authored configuration, and the engine authoring SQL at run time is precisely what would make it a reasoning engine. At design time, AI-generated SQL inside **draft** workflows for human review is acceptable and kept. Variable values must bind as parameters, never splice into SQL text (W1).
5. **Should workflows invoke Analysts?** In the future, yes — a workflow step that asks a governed question is the correct integration direction. Deferred (§6): it requires the Investigation Engine contract ADR-0002 deferred. It must never be approximated by an `AI_REASON` node imitating an analyst.
6. **Should Analysts invoke workflows?** In the future, yes — an analyst that finds a problem triggering a remediation workflow is the workforce's action loop, and chat's read-only boundary already points humans this way. Deferred (§6): it requires the findings/action contracts deferred in ADR-0002/0003.
7. **Does the current implementation introduce another reasoning engine?** **No.** No loop, no model-chosen next action, no model-authored runtime SQL. The traversal decision belongs to `CONDITION` nodes with explicit operators. This must remain true (§11).
8. **Does it violate ADR-0002/0003 responsibilities?** No ownership violation — it performs no resolution, orchestration, or investigation, and duplicates neither engine. It shares two *defects* with the agent runtime (the ungoverned `executeQuery` seam; the compiled-bounds posture), and it exposes one **promise conflict**: chat's read-only boundary directs users to automations as the sanctioned operational path, while this path enforces read-only nowhere. W1/W2 resolve the conflict in favor of the promise.
9. **Is it becoming a generic low-code tool?** Not yet — the discipline is visible (seven node types, no scheduler, no connector zoo, no loops). The gravitational pull is real, so §11 fixes its identity: it is the AI Workforce's execution surface; node-palette or connector expansion beyond that identity requires a superseding ADR, not incremental accretion.
10. **Protected / stabilized / deferred:** §3, §5, §6 respectively.

**Responsibilities that permanently belong here:** the workflow definition model and lifecycle; deterministic traversal and variable resolution; step execution bounds; execution records and step traces; the external trigger surface; design-time AI generation as drafts.

**Responsibilities that must never live here:** business investigation and answer synthesis (engine-owned); analyst identity (agent-model-owned); business meaning (Foundation-owned); connection credentials (registry-owned); runtime SQL authorship by AI (nobody's, on this surface); governance enforcement (governance-owned — invoked, not implemented).

## 3. Protected Architecture

Not to be redesigned without overwhelming evidence:

1. **Workflow-as-data.** The graph is a tenant-owned JSONB document; the engine is graph-agnostic machinery; no workflow logic in platform code. The exact mirror of ADR-0003's protected agent-as-data property, and the same workforce enabler.
2. **Deterministic traversal with explicit branching.** The AI never decides which node runs next; `CONDITION` nodes with explicit operators own routing. This is the line between an execution engine and a third reasoning engine.
3. **AI as bounded, inspectable steps.** Design-time generation produces **drafts only**, grounded in real catalogs, always behind human review; run-time AI is confined to `AI_REASON`/`IMAGE_ANALYSE` leaf judgments whose inputs and outputs land in the trace.
4. **The execution record as replayable evidence.** Trigger payload, per-step resolved inputs, output, executed SQL, duration, error — the run's full story, queryable per tenant.
5. **Human-gated activation.** Drafts are never triggerable; nothing executes on save; only `ACTIVE` workflows resolve by slug. (The gate gains validation in §5; the human ownership is protected.)
6. **Bounded execution.** The 50-step cap, per-node row caps, and query timeouts — cost and blast radius bounded by construction.
7. **Shared platform primitives.** One connection registry, one SQL execution seam, one AI client — no capability-private duplicates.

## 4. Stabilization Decisions by Area

| Area | Decision | Reason · Business impact · Risk if ignored |
|---|---|---|
| **Workflow definition model** (graph-as-data, statuses, slugs) | **KEEP (protected)** | §3.1. Correct, tenant-owned, canvas round-trips it. *Risk if reopened:* destroying the property the workforce's action layer will build on. |
| **Execution engine (traversal, dispatch, step cap)** | **KEEP** | Deterministic single-path traversal is a feature, not a limitation, at this maturity — it is what keeps this from being a reasoning engine. Parallelism/loops are deferred product questions (§6). |
| **Variable resolution → SQL** | **STABILIZE — the record's critical work** | Verified: `{{values}}` (incl. unauthenticated webhook payload) string-splice into SQL text before execution. Decision: parameterized binding (W1) — template placeholders become bind parameters; splice-into-text is removed. *Risk if ignored:* unauthenticated SQL injection into customer databases. |
| **SQL execution posture** | **STABILIZE** | Same unvalidated, write-capable `executeQuery` seam as ADR-0003; same remedy, reusing A1's validator and A2's chain integration (W2). The read-only promise chat makes on this path becomes true. *Risk if ignored:* the platform's read-only guarantee is false on its own sanctioned write-adjacent surface. |
| **Public webhook trigger** | **STABILIZE** | Slug-as-secret plus unauthenticated tenant-selection header is not an enterprise posture for an endpoint that reaches SQL. Decision: per-workflow secret verification, hardened tenant resolution, rate limiting, uniform unknown-slug responses (W3). The unauthenticated-endpoint *pattern* is kept — external systems must call it. *Risk if ignored:* slug enumeration + payload injection = remote data access. |
| **Execution lifecycle** | **STABILIZE** | No deadline (schema's `TIMEOUT` never set), no cancellation, orphaned `RUNNING` forever, synchronous callers held indefinitely. W4 mirrors ADR-0002 S3 / ADR-0003 A3. *Risk if ignored:* wedged executions, thread exhaustion, webhook callers timing out against completed-but-unreported runs. |
| **Graph validation & activation** | **STABILIZE** | Structural defects (no trigger, unreachable nodes, missing condition edges) surface at run time; activation is a bare status edit on a webhook-exposed artifact. Decision: validation at save + activation gate (W5). *Risk if ignored:* externally triggered flows failing at run time for authoring errors a save-time check would catch. |
| **Condition & variable semantics** | **STABILIZE** | Case-insensitive `eq`, unparseable-numeric→`0` coercion, and unresolvable-variable→null are silent-misclassification machinery in operational flows that take actions. Decision: strict evaluation — resolution failures and type mismatches fail the step with a clear trace (W6). *Risk if ignored:* wrong branch taken silently in a flow that returns decisions to external systems. |
| **Execution records & retention** | **KEEP (protected) + STABILIZE growth** | The trace model is right; nothing bounds table growth under production trigger rates, and the API caps reads at 50 with no archival. Decision: retention policy aligned with the platform retention split (W7). *Risk if ignored:* unbounded growth on the hottest write path. |
| **AI generation (design time)** | **KEEP** | Draft-only, catalog-grounded, human-reviewed — the correct pattern, and the workforce's future "describe an automation" surface. Generation quality measurement belongs to the existing checklist, not a story. |
| **`AI_REASON` / `IMAGE_ANALYSE` nodes** | **KEEP** | Bounded leaf judgments with recorded inputs/outputs. Injection screening for payload-fed prompts is deferred pending the steerability evidence pattern (§6) — these nodes hold no tool authority, and W1/W2 remove the SQL blast radius. |
| **Conversational platform integration** | **KEEP (absent) — deliberately** | No runtime integration exists in either direction; both future directions (workflow→analyst, analyst→workflow) are deferred contracts (§6), not stabilization work. |
| **Semantic layer integration** | **KEEP (absent) — deliberately** | Literal, tenant-authored SQL and prompts are the *point* of this surface; semantic machinery belongs to the engine. Not a gap. |
| **Tenancy & data placement** | **KEEP with verification** | Double scoping (tenant column + `search_path`) is sound; the webhook's unauthenticated tenant resolution is the weak edge and is covered by W3 + exit criteria. |
| **Configuration** | **KEEP compiled bounds, document them** | Same posture as ADR-0003: constants acceptable at this maturity; W4 externalizes only the deadline it introduces; a configuration reference entry is story DoD. |
| **Demo seeding** | **KEEP** | Explorability out of the box; verify demo workflows are tenant-safe as part of regression, not a story. |

## 5. Production Stabilization Work (accepted)

Seven stories owned by this record, full definitions in §9:

| # | Work | Area | Severity |
|---|---|---|---|
| W1 | Parameterized variable binding for `DB_QUERY` (no string splice into SQL) | Security | Critical |
| W2 | Read-only enforcement + governance chain for workflow SQL | Governance | Critical |
| W3 | Webhook hardening: per-workflow secret, tenant resolution, rate limiting | Security | High |
| W4 | Execution deadline (`TIMEOUT` assigned), cancellation, orphan reconciliation | Resilience | High |
| W5 | Graph validation at save + activation gate | Operational maturity | Medium |
| W6 | Strict condition & variable semantics (fail loud, trace why) | Correctness | Medium |
| W7 | Execution retention + runtime observability | Ops/SaaS readiness | Medium |

### External blocking dependencies

| Dependency | Owning record | Why it blocks this capability |
|---|---|---|
| Connection secrets encrypted at rest | **Connection Registry stabilization** | Every `DB_QUERY` authenticates with these credentials. |
| SQL safety validator + governance-chain entry point for non-conversational callers | **ADR-0003 A1/A2 (Autonomous Agents)** | W2 reuses the same seam-level machinery; building it twice is the anti-goal. |
| Actuator authentication for metrics exposure | **Authentication / Platform Security stabilization** | W7's metrics ship through the secured actuator. |

## 6. Deferred Work (belongs to AI Workforce evolution)

| Item | Why deferred |
|---|---|
| **Workflow → Analyst invocation** (a step that asks a governed question) | Requires the Investigation Engine contract ADR-0002 deferred; the correct integration, adopted by future ADR — never approximated with `AI_REASON` in the interim. |
| **Analyst → Workflow invocation** (findings trigger remediation flows) | Requires the findings/action contracts deferred in ADR-0002/0003; the workforce's insight-to-action loop, designed once, with governance, not ad hoc. |
| **Scheduling** (timer-triggered workflows) | A real product need with an honest current answer (none exists, schema forbids it). Deserves a deliberate decision against the platform's existing schedulers rather than a fourth scheduler; deferred to a scheduling-consolidation decision. |
| **Retries / resumption / idempotency contracts** | Same posture as prior records — retries invite duplicate side effects, which for *action* workflows means duplicate real-world writes; requires an idempotency design, not a flag. |
| **Parallel branches, joins, loops** | Expands the execution model materially; no current consumer needs it; loops in particular sit next to the reasoning-engine line and need the §11 test applied at design time. |
| **Workflow versioning** (executions pinned to the graph that ran) | Valuable for audit correlation; not blocking because each execution's trace stores its own resolved inputs and SQL — the run itself remains reconstructable. |
| **Async execution with callbacks** | The right long-term shape for slow flows; today's consumers are synchronous and W4's deadline bounds the exposure. |
| **AI-node prompt-injection screening** | Deferred pending steerability evidence (the ADR-0002/0003 pattern): these nodes hold no tool authority and W1/W2 remove the SQL blast radius; measure before hardening. |
| **Node palette / connector expansion** | Any growth beyond the execution-surface identity requires a superseding ADR (§11), not incremental accretion. |

## 7. Rejected Recommendations (not to be implemented in stabilization)

| Rejected | Why |
|---|---|
| **Making execution asynchronous/queued now** | Synchronous-with-deadline (W4) is sufficient for current consumers; an execution queue is infrastructure for the deferred async shape, premature today. |
| **Adding retry mechanisms** | Duplicate side effects on an action surface are worse than honest failure; consistent with ADR-0002/0003 posture. |
| **Building a workflow scheduler now** | The platform already has three schedulers (inventory finding); adding a fourth before the consolidation decision compounds the defect. |
| **Routing workflow SQL/prompts through the semantic layer** | Literal tenant authorship is this surface's contract; semantic resolution belongs to the engine. Deliberate absence, not a gap. |
| **Merging Automations into the agent runtime** (or vice versa) | They own different things — identity vs. execution — and both records protect that separation; merging re-couples what ADR-0002/0003 just finished separating. |
| **Authenticating the webhook with platform user auth** | External systems are not platform users; the fix is per-workflow secrets + rate limiting (W3), preserving the unauthenticated-caller pattern the surface exists for. |
| **Backward-compatible lenient mode for condition/variable semantics** | Preserving silent misclassification as an option preserves the defect; W6 fails loud with clear traces, and existing workflows surface their latent typos as fixable errors, not wrong actions. |

## 8. SaaS Product Assessment

- **Customer value:** real and distinct — the only surface where Zevra *does* things for the business rather than answering; "webhook in, decision out, every step traced" is a concrete integration story.
- **Differentiated:** the builder itself is not (visual workflow tools are commodity); **AI judgment as an inspectable step inside a traced, tenant-owned flow** is the differentiator, and it compounds with the platform's governance story once W2 makes the inheritance real.
- **Understandable by business users:** the canvas and the AI generator are the platform's most approachable authoring experience; the trace panel makes runs legible.
- **Would customers trust it?** Not today, knowingly — an unauthenticated endpoint reaching injectable SQL fails any review instantly. After W1–W3: yes, with evidence.
- **Strengthens the AI Workforce:** structurally — it is the action half the workforce needs (§6's deferred invocation contracts are the payoff), and the generator previews "describe the automation you need."
- **Commercially valuable / helps sell Zevra:** yes, as the operational-integration wedge: it answers "can Zevra act, not just analyze?" in demos.
- **Enterprise-ready:** no (§1); yes upon §10.

## 9. Linear Stories (accepted work only)

**W1 — Parameterized variable binding for `DB_QUERY`**
*Business Objective:* eliminate SQL injection from webhook payloads and node variables — the platform's most exposed defect.
*Technical Scope:* `{{variable}}` placeholders in `DB_QUERY` SQL resolve to **bind parameters** (PreparedStatement) instead of string splice; structural SQL (table/column names) remains static authored text — placeholders are legal only in value positions, enforced at save-time validation (with W5) and at run time; the step trace records the SQL template plus bound values separately; existing workflows migrate transparently where placeholders are value-position (the common case), and value-position violations fail validation with a clear message.
*Acceptance Criteria:* injection fixture suite — hostile payloads (`' OR 1=1 --`, stacked statements, `UNION`, comment tricks) through webhook and manual triggers — executes harmlessly as bound literals or fails validation; traces show template + bound values; legitimate workflows unaffected.
*Dependencies:* W5 for the save-time half (run-time enforcement lands regardless). *Priority:* Critical. *Estimate:* Medium.

**W2 — Read-only enforcement + governance chain for workflow SQL**
*Business Objective:* the platform's read-only guarantee and governance audit become true on the surface chat itself calls the sanctioned operational path.
*Technical Scope:* workflow `DB_QUERY` execution reuses the seam-level machinery from ADR-0003 A1/A2 — statement safety validation (non-SELECT incl. `RETURNING` DML and writing CTEs rejected), read-only transaction, row security, masking, row limits, governance audit attributed to workflow + execution; verdicts recorded in step traces; blocked steps fail the step with the reason in trace.
*Acceptance Criteria:* the A1 fixture suite passes on this path; an RLS-blocked query and a masked column behave per policy with both traces recording it; every workflow query findable in governance audit; capability page's governance-gap statement deleted truthfully.
*Dependencies:* ADR-0003 A1/A2 (shared machinery). *Priority:* Critical. *Estimate:* Small (given A1/A2 exist).

**W3 — Webhook hardening**
*Business Objective:* an enterprise-defensible public trigger surface.
*Technical Scope:* per-workflow secret token generated at activation, verified via header (constant-time compare); slug remains the identifier, stops being the secret; hardened tenant resolution (unknown/mismatched `X-Nexus-Tenant` → uniform 404, no information leakage; slug+tenant must co-resolve); per-tenant rate limiting on the webhook endpoint; caller metadata (source IP, received-at) recorded on the execution.
*Acceptance Criteria:* trigger without/with-wrong secret → uniform 404/401 per design, with no timing or body divergence between unknown-slug and bad-secret; rate limit enforces and resets; existing workflows require secret regeneration on next activation edit (documented migration); caller metadata visible in execution history.
*Dependencies:* none. *Priority:* High. *Estimate:* Medium.

**W4 — Execution lifecycle: deadline, cancellation, orphan reconciliation**
*Business Objective:* bounded, recoverable runs; the schema's `TIMEOUT` status becomes real.
*Technical Scope:* configurable wall-clock deadline enforced between steps → execution ends `TIMEOUT` with traces preserved; owner-verified cancel for manual/UI runs; startup reconciliation marking stale `RUNNING` executions `FAILED`; webhook callers receive the terminal status within the deadline bound.
*Acceptance Criteria:* slow-run fixture ends `TIMEOUT` at deadline with intelligible partial traces; cancel ends a run cleanly; restart reconciles orphans; deadline documented and configurable.
*Dependencies:* none. *Priority:* High. *Estimate:* Small.

**W5 — Graph validation at save + activation gate**
*Business Objective:* authoring errors surface at authoring time; nothing externally triggerable without passing structural checks.
*Technical Scope:* save-time validation (exactly one `TRIGGER`, reachable nodes, condition nodes with both edges, known node types, `DB_QUERY` placeholder positions per W1) returning actionable errors to the canvas; activation additionally requires a successful test-panel run against a sample payload; existing ACTIVE workflows grandfathered with a flagged warning until re-validated.
*Acceptance Criteria:* each structural defect class blocks save/activation with a specific message; valid graphs save unchanged; activation without a passing test run is blocked; grandfathering behaves as specified.
*Dependencies:* none (W1 adds one rule). *Priority:* Medium. *Estimate:* Small.

**W6 — Strict condition and variable semantics**
*Business Objective:* no silent misclassification in flows that return decisions to external systems.
*Technical Scope:* unresolvable `{{paths}}` fail the step with the unresolved path named in the trace; numeric operators on unparseable values fail the step (no `0` coercion); `eq` semantics made explicit (documented case behavior, with a per-node case-sensitivity option); failure messages name node, path, and value.
*Acceptance Criteria:* typo'd path fails its step with the path in the trace; `"abc" > 5` fails rather than evaluating as `0 > 5`; documented `eq` behavior matches implementation; regression fixtures for every operator.
*Dependencies:* none. *Priority:* Medium. *Estimate:* Small.

**W7 — Execution retention + runtime observability**
*Business Objective:* bounded storage growth and operational visibility on an externally triggered surface.
*Technical Scope:* retention policy for execution records aligned with the platform retention split (ADR-0002 S6; configurable window, default ≥ 90d) with a cleanup job; metrics — run outcomes by status (incl. `TIMEOUT` and reconciled orphans), webhook invocation/rejection/rate-limit counts, run latency distribution, per-workflow trigger volume; exposed via the secured actuator.
*Acceptance Criteria:* purge test removes expired executions only; a failed run, a timeout, an orphan reconciliation, and a rate-limit rejection are each observable in metrics without log access; metric names documented.
*Dependencies:* actuator authentication (external). *Priority:* Medium. *Estimate:* Small.

## 10. Exit Criteria — declaring Workflow Automation **STABLE**

1. **Injection closed:** the W1 hostile-payload fixture suite passes through both webhook and manual triggers against a live PostgreSQL connection; traces show template + bound values separated.
2. **Write-path closed:** the shared read-only fixture suite (DML, `RETURNING` DML, writing CTEs, multi-statement) rejected on the workflow path before execution.
3. **Governance drill:** an RLS-blocked workflow query and a masked column behave per policy, recorded in both the step trace and governance audit; a sampled week of workflow queries fully findable in governance audit with workflow + execution attribution.
4. **Webhook posture:** secret verification, uniform unknown-slug/bad-secret responses (no information leakage), tenant-mismatch tests, and rate-limit enforcement all green; slug enumeration probes recorded as rejected.
5. **Lifecycle:** deadline (`TIMEOUT` assigned), cancel, and orphan-reconciliation tests green; kill-mid-run drill leaves no `RUNNING` execution after restart.
6. **Authoring gate:** every structural-defect class blocked at save/activation with actionable messages; zero un-validated ACTIVE workflows remaining (grandfathered set re-validated or flagged).
7. **Semantics:** strict-evaluation fixtures green for every operator and resolution form; no code path coerces unparseable numerics to `0`.
8. **Isolation:** cross-tenant workflow/execution/connection-key tests green, including identical slugs across tenants and webhook tenant-resolution edge cases (missing, unknown, mismatched headers).
9. **Retention & growth:** purge test green; execution-table growth bounded under a recorded sustained-trigger test.
10. **Observability:** forced drills (failed run, timeout, orphan reconciliation, rate-limit rejection) visible in metrics without log access.
11. **Performance evidence:** recorded webhook latency distribution for realistic graphs (AI nodes dominating) and a concurrency test establishing safe trigger volume before thread exhaustion, published in operations docs.
12. **Checklist closure:** the capability page's stabilization-checklist items covered by W1–W7 checked with links to tests/evidence; full backend suite green with zero removed tests; the capability page's governance and injection statements updated truthfully.

## 11. Future Evolution Contract

Workflow Automation is the AI Workforce's **execution and action surface** — the sanctioned path from insight to operational action. Future workforce capabilities build on it in two directions, both by future ADR against the deferred contracts: **workflows invoking Analysts** (a step that asks a governed question through the Investigation Engine contract) and **Analysts invoking workflows** (findings triggering remediation flows through the findings/action contracts).

Two standing constraints, violable only by a superseding ADR:

1. **It must never become a reasoning engine.** No model-chosen traversal, no model-authored runtime SQL, no autonomous loops inside a workflow run. AI participates as draft-time generation and bounded leaf-judgment steps only. A proposed feature that requires the model to decide *what happens next at run time* belongs to the intelligence engine, not here.
2. **It must remain the AI Workforce's execution surface, not a generic low-code platform.** Node-palette growth, connector catalogs, or embedding-third-party-app ambitions beyond what the workforce's action loop requires are identity changes requiring a superseding ADR, not incremental accretion.

---

## Consequences

**Positive:** the platform's most exposed vulnerability (unauthenticated SQL injection) is closed at its root (parameterization) rather than patched at its symptom; the read-only promise chat makes on this surface becomes enforced truth; the governance chain gains its third and final SQL caller, completing one-posture-for-all-SQL across the platform; the workforce's action layer gets a recorded identity and two named future contracts instead of drift.

**Negative:** webhook integrations require a one-time secret adoption on activation edit (documented migration); strict semantics (W6) will surface latent typos in existing workflows as errors — deliberate, but visible to tenants; scheduling remains honestly absent until the consolidation decision; the analyst↔workflow loop stays unbuilt until the deferred contracts exist.

## Alternatives considered

- **Declare Stable now** — rejected: an unauthenticated endpoint reaching injectable, write-capable, unaudited SQL is the most disqualifying finding of the stabilization program to date.
- **Fix injection by escaping/sanitizing interpolated strings** — rejected: escaping is the historically failure-prone half-measure; parameterized binding removes the vulnerability class, and value-position enforcement keeps templates honest.
- **Take the capability offline until the workforce phase** — rejected: it is the platform's only action surface, two demo workflows and the chat boundary handoff depend on its existence, and W1–W3 are bounded work.
- **Accept the governance gap as a bounded exception** — rejected for the same reason as ADR-0003: the boundary is not read-only in practice, so "bounded" is false; and here the exposure is unauthenticated.
