---
description: The governing implementation roadmap for converging Zevra's Chat, Agent, Workflow, Reports, and Brief execution paths onto a single Unified Answer Engine — one shared spine of single-owner stages, with experiences differing only by execution policy. Planning document; supersedes ad-hoc convergence work and must be referenced by every future migration phase.
---

# Unified Answer Engine — Master Implementation Plan

**Status:** Approved architecture, pre-implementation. This document is the single source of truth for the migration. Every implementation phase must reference it. It changes only by explicit amendment.

**Scope of this document:** planning only. No code, no refactoring, no architectural change is authorized by this document itself — it authorizes the *sequence* in which future, separately-approved phases proceed.

---

## 1. Executive Summary

### Why
Zevra answers a user question through **two independent cores** that were built at different times and have drifted:

- an **Agent core** (`AgentRunner`, ReAct tool-calling) with the ADR-0003 `ExecutionContract` grounding and a pre-governance business-object gate, but **no** Semantic Foundation (business-language resolution, value domains, literal validation, learning); and
- a **Conversational core** (`ChatService` → `ReasoningEngine` → `ReasoningPlanner`, iterative single-step planning) with the full Semantic Foundation but **no** `ExecutionContract` grounding and **no** business-object gate.

The same responsibilities — grounding, schema rendering, SQL generation, validation, response composition — exist twice, in incompatible forms. A fix to one (e.g. the identifier-fidelity hardening) must be re-applied to the other by hand. This is the direct cause of the recurring identifier-substitution class of defect and of every "we fixed it in Chat but not in Agent" gap.

### Business objectives
- **Fix once, everywhere.** A new grounding rule, validation, or safety control is added in one owner and is immediately live for Chat, Agent, Workflow, Reports, and Brief.
- **Consistent answers** across experiences for the same question and tenant.
- **Lower change cost and defect rate** by removing duplicated logic.

### Architectural objectives
- One shared **Answer Engine spine** of single-owner stages.
- Experiences (Chat, Agent, Workflow, Reports, Brief) differ **only by execution policy** — turn model, loop shape, tool set, streaming, output format.
- Preserve ADR-0003: `AgentBrain` = sole business-reasoning owner; deterministic execution + governance = sole execution owner; `ExecutionContract` immutable; `SqlGovernancePipeline` unchanged.

### Success criteria
The migration is successful when a capability added once to a spine owner is automatically available to all experiences, no answer-producing responsibility has more than one implementation, and production behavior is preserved at every deployable step (verified by parity tests and live probes).

---

## 2. Current State

Established by the Architecture Discovery. There are two answer cores; the other "paths" wrap one of them.

| Experience | Entry | Underlying core | Notes |
|---|---|---|---|
| **Chat** | `ChatController` → `ChatService.ask` | Conversational core (routes to Agent core when `ZevraAgentRouter` matches) | Owns routing + decision modes |
| **Agent** | `ZevraAgentController` → `AgentRunner.run` | Agent core | ReAct loop |
| **Reports** | `ScheduledReportService` (@Scheduled) | **Delegates to Chat** (`chatService.ask`) | schedule + formatting only |
| **Brief** | `MorningBriefService` | **Delegates to Agent** (`agentRunner.run`) + `openAi.chatWithJson` synthesis | schedule + synthesis only |
| **Workflow / Alerts** | `AutomationController` / `AlertController` | Steward-authored SQL via `DbQueryExecutor` → `DynamicSqlService`; raw `openAi.chat*` for reasoning/prose | **Bypasses `SqlGovernancePipeline`** for steward SQL |

**Shared today:** `SqlGovernancePipeline.governSql`, `GovernanceAuditService`, `DynamicSqlService`, `AzureOpenAiClient`, the Enterprise Map, and `SqlIdentifierGuidance` (prompt text, recently extracted).

**Duplicated today:**

| Responsibility | Agent core | Conversational core |
|---|---|---|
| Business resolution | `AgentBrain` (+ `EnterpriseSemanticAssembler`) | `BusinessLanguageResolver` + `SemanticService` |
| Grounding / schema render | `PromptContextBuilder` + `PromptAssembler` (backticked, from `ExecutionContract`) | `EnterpriseMapService` blocks + `ChatService.assembleEntityContext` |
| SQL generation | `AzureOpenAiClient.chatWithTools` (ReAct) | `ReasoningPlanner.nextStep` (single-step) |
| Validation (pre-governance) | business-object gate (`ExecutionBindings.approvedAssets`) | `LiteralValidator` |
| Execution + audit | `AgentToolRegistry.execQueryDatabase` | `ReasoningEngine.reason` |
| Response composition | `AgentRunner` final-answer prose | `ChatService.composeAnswer` (+ Brief synthesis, `AlertComposerService`) |

---

## 3. Target State

A single **Answer Engine spine** of single-owner stages, consumed by thin **execution policies**.

**Shared spine (fixed order, identical for every experience):**
```
AgentBrain.resolve
  → EnterpriseSemanticAssembler.assemble
  → ExecutionContractBuilder.compile           (immutable ExecutionContract)
  → PromptContextBuilder.build
  → PromptAssembler.assemble                   (+ SqlIdentifierGuidance)
  → [ generation/execution loop — POLICY ]
      per candidate SQL → Runtime.execute       (gate → literal validation → SqlGovernancePipeline → execute → audit)
  → ResponseComposer.compose
```

**Experience-specific (execution policy only):** intake/turn model, the loop shape (ReAct vs single-step vs steward-SQL), tool set, iteration budget, streaming, output format, delivery.

**Single ownership boundaries:** every answer-producing responsibility resolves to exactly one owner (see §6). `AgentBrain` owns reasoning; the deterministic execution owner (`Runtime`) owns gate+validation+governance+execute+audit with `SqlGovernancePipeline` unchanged inside it.

**Deliberate minimalism:** the target is reached by **sharing stage owners**, not by building a facade. `AnswerEngine`/`ExecutionStrategy` are *deferred, optional* consolidations (§5 Phase 6) introduced only if the two experience shells still duplicate the stage sequence after all owners are shared.

---

## 4. Guiding Principles (non-negotiable)

1. **`AgentBrain` is the only business-reasoning owner.** `BusinessLanguageResolver`/`SemanticService` become its *inputs*, never independent reasoners.
2. **One deterministic execution owner.** Gate, literal validation, governance invocation, execution, and audit occur in exactly one place; `SqlGovernancePipeline` is unchanged (ADR-0003 A2).
3. **`ExecutionContract` stays immutable**, fully compiled before the loop runs; the loop and Runtime derive nothing from persistence.
4. **Every responsibility has exactly one owner** (§6). No duplicated execution paths.
5. **Experiences differ only by execution policy.** Reasoning, grounding, SQL generation, validation, governance, and response generation are shared-core.
6. **Build on the existing architecture.** Reorganize existing components before introducing any new one; a new abstraction requires a clear, single architectural responsibility that no existing component can host.
7. **No big-bang rewrite.** Every phase is independently deployable and reversible.
8. **Every phase preserves production behavior**, proven by parity tests and live probes before it is considered done.

---

## 5. Migration Phases

Ordering rationale: establish the shared execution floor first (P1), then move the Conversational core onto the spine bottom-up (P2 reasoning inputs → P3 grounding), then dedup the tail (P4 composer), then bring the steward-SQL lane under governance (P5); consolidate the seam last and only if warranted (P6). Each phase retires a distinct duplication class and can ship and stop.

> **New-abstraction discipline (applies to every phase):** the default is to *reorganize existing components*. Only two new components are pre-justified: a shared execution **Runtime** (P1) and a shared **ResponseComposer** (P4), each a clear single responsibility currently duplicated. All other objectives are met by re-pointing existing classes.

### Phase 1 — Shared deterministic execution ("Runtime")
- **Objective:** One code path for gate → literal validation → `SqlGovernancePipeline` → execute → audit, called by both cores.
- **Scope:** Extract the shared execution orchestration currently duplicated in `AgentToolRegistry.execQueryDatabase` and `ReasoningEngine.reason` into a single collaborator (candidate name `GovernedSqlRuntime`; embodies the ADR-0003 "Runtime" responsibility). It *wraps* existing logic — it introduces no new governance behavior.
- **Components affected:** `AgentToolRegistry`, `ReasoningEngine` (re-pointed to the shared component). New: the extracted Runtime collaborator.
- **Intentionally excluded:** `SqlGovernancePipeline` internals (unchanged); grounding; SQL generation; response composition; the business-object gate *semantics* (moved verbatim, not redesigned).
- **Dependencies:** none (foundational).
- **Risks:** subtle divergence between the two current execution paths (e.g. audit-recording conditions, async handling) merged incorrectly.
- **Rollback:** the two callers keep their current inline paths behind a flag; flip back to inline on regression. Pure extraction ⇒ revert is mechanical.
- **Validation:** `GovernanceParityIntegrationTest` and `GovernanceAuditRecordOutcomeTest` must pass unchanged; payload/execution capture on both cores byte-identical; audit rows identical for representative queries.
- **Exit criteria:** both cores execute SQL exclusively through the shared Runtime; no behavioral or audit diff.
- **Architectural impact:** Medium surface, **high** duplication removed (execution + audit unified; foundation for the gate to reach Chat).

### Phase 2 — `AgentBrain` absorbs Semantic Foundation inputs
- **Objective:** Make `AgentBrain` the single reasoning owner for *both* cores, consuming `BusinessLanguageResolver` resolutions, literal scope, value domains, and expansion tokens, and resolving for any connection/domain scope (not only a `ZevraAgent`).
- **Scope:** Additive inputs to `AgentBrain.resolve`; `ResolvedBusinessModel` carries the resolution/literal signals. Chat continues to *render* via its current context builder for now (grounding swap is P3).
- **Components affected:** `AgentBrain`, `ResolvedBusinessModel`; `BusinessLanguageResolver`/`SemanticService` become collaborators of `AgentBrain`.
- **Intentionally excluded:** grounding render (P3), the loop, the composer. No change to how Chat currently prompts yet.
- **Dependencies:** none hard; independent of P1 but naturally sequenced before P3.
- **Risks:** `AgentBrain` accreting orchestration that isn't reasoning; agentless chat scope resolution differing from today's `domainKeys` behavior.
- **Rollback:** additive; Chat keeps its existing resolution wiring until P3 consumes the new output. Revert = stop calling the new inputs.
- **Validation:** `AgentBrainTest`, `BusinessLanguageResolverTest`, `DeterministicLiteralResolutionTest` green; resolution provenance identical for a corpus of questions across both cores.
- **Exit criteria:** `AgentBrain` produces a `ResolvedBusinessModel` for chat-scope questions equivalent in resolution content to today's Conversational pipeline.
- **Architectural impact:** Medium surface, prepares D1/D2 removal.

### Phase 3 — Conversational core adopts `ExecutionContract` grounding
- **Objective:** Chat grounds via `ExecutionContractBuilder` → `PromptContextBuilder` → `PromptAssembler` (shared spine) and executes through the shared Runtime (P1), gaining the business-object gate; retire the duplicate chat grounding.
- **Scope:** `ReasoningEngine`/`ReasoningPlanner` consume the spine's grounding + shared Runtime instead of `EnterpriseMapService` blocks + `ReasoningPlanner.SYSTEM_PROMPT` schema. Preserve the single-step loop shape and all Semantic Foundation richness (resolutions, literal candidates, value domains, knowledge-graph context) — now carried through the contract/prompt from P2.
- **Components affected:** `ReasoningEngine`, `ReasoningPlanner`, `ChatService.buildContextSummary`.
- **Retired:** `ChatService.assembleEntityContext`/`rankEntityBlockKeys`, `EnterpriseMapService` per-table block rendering, `ReasoningPlanner.SYSTEM_PROMPT` schema portion.
- **Intentionally excluded:** the loop shape (stays single-step); response composition (P4).
- **Dependencies:** **P1 and P2.**
- **Risks:** **Highest.** This touches the most-tested surface; token-budget/ranking behavior and Semantic-Foundation signal fidelity must be preserved; regression here degrades chat answer quality.
- **Rollback:** feature-flag the grounding source per request; flip to legacy context builder on regression. Keep the retired renderers in place (dormant) until the flag is removed.
- **Validation:** prompt byte-diff harness (legacy vs spine) over a question corpus; re-baseline `PlannerContextAssemblyTest`, `EntityBlocksRenderingTest`, `RetrievalExpansionTest`; live A/B on identifier fidelity and answer quality; SSE trace parity.
- **Exit criteria:** Chat prompts are produced by the spine, gate active, quality parity confirmed live; legacy renderers unreferenced.
- **Architectural impact:** **High** surface, **high** duplication removed (D1, D3, D4).

### Phase 4 — Shared `ResponseComposer`
- **Objective:** One owner for answer generation.
- **Scope:** Introduce `ResponseComposer` (justified new component: a single responsibility duplicated four ways). Re-point `ChatService.composeAnswer`, `AgentRunner` final-answer prose, `MorningBriefService` synthesis, and `AlertComposerService` to it; experiences pass a format policy.
- **Components affected:** the four composition sites; `AzureOpenAiClient` (transport, unchanged).
- **Intentionally excluded:** grounding, loop, Runtime.
- **Dependencies:** independent of P3 in principle; sequence after P3 to avoid churn.
- **Risks:** answer tone/format regressions per experience (Chat summary vs Agent factual vs Brief synthesis vs Alert prose).
- **Rollback:** per-experience flag to the legacy composer.
- **Validation:** output snapshot tests per experience; human review of representative answers; Brief/Alert format preserved.
- **Exit criteria:** all four sites call `ResponseComposer`; legacy composition methods unreferenced.
- **Architectural impact:** Medium surface, removes D6.

### Phase 5 — Steward SQL under shared governance
- **Objective:** Workflow/Alert steward-authored SQL executes through the shared Runtime, closing the governance bypass.
- **Scope:** `DbQueryExecutor` (and alert trigger SQL) routes through `Runtime.execute` (generation skipped — SQL is given; governance/RLS/masking/audit applied).
- **Components affected:** `DbQueryExecutor`, alert evaluation path.
- **Intentionally excluded:** automation *design-time* generation (`AutomationGeneratorService`); LLM reasoning steps.
- **Dependencies:** **P1.**
- **Risks:** previously-permitted steward SQL now blocked by governance (RLS/masking/row limits) — surfaces latent policy gaps.
- **Rollback:** flag to direct execution for steward SQL; triage blocked cases before enforcing.
- **Validation:** run existing automations/alerts through governance in shadow mode; compare results; audit coverage on workflow SQL.
- **Exit criteria:** no steward SQL executes outside the Runtime; blocked cases dispositioned.
- **Architectural impact:** Low-Medium surface, removes D8 (governance asymmetry).

### Phase 6 — *(Optional, conditional)* Sequence consolidation
- **Objective:** Only if, after P1–P5, `AgentRunner` and `ChatService`/`ReasoningEngine` still duplicate the stage-sequencing prefix, extract a thin `AnswerEngine` sequencer + `ExecutionStrategy` seam (ReAct / single-step / steward-SQL).
- **Scope:** No new behavior — pure sequencing extraction; the two loops become strategies.
- **Components affected:** the two shells.
- **Intentionally excluded:** everything already shared.
- **Dependencies:** P1–P4.
- **Risks:** introducing a facade that adds indirection without removing real duplication (see §Final Review). **Do not start unless measured duplication justifies it.**
- **Rollback:** keep the two shells; the facade is additive.
- **Validation:** full suite + live probes unchanged; the facade must delete more lines than it adds.
- **Exit criteria:** a single sequencer, or an explicit decision to *not* build it because the shells no longer duplicate.
- **Architectural impact:** Low; **explicitly optional.**

---

## 6. Component Ownership Matrix (definitive)

| Responsibility | Single owner | Absorbs / retires |
|---|---|---|
| Business reasoning & resolution | **`AgentBrain`** | `BusinessLanguageResolver`, `SemanticService` become inputs (P2) |
| Semantic assembly (Map → model) | **`EnterpriseSemanticAssembler`** | — |
| Execution bindings / immutable contract | **`ExecutionContractBuilder`** | — |
| Prompt context construction | **`PromptContextBuilder`** | — |
| Prompt rendering / grounding | **`PromptAssembler`** (+ `SqlIdentifierGuidance`) | retires `EnterpriseMapService` blocks, `ChatService.assembleEntityContext`, `ReasoningPlanner` schema (P3) |
| Generation loop shape | **Execution policy** (`AgentRunner` ReAct / `ReasoningEngine` single-step / steward) | — |
| Deterministic execution: gate → literal validation → governance → execute → audit | **Runtime** (extracted; wraps `SqlGovernancePipeline` **unchanged**, `DynamicSqlService`, `GovernanceAuditService`, gate, `LiteralValidator`) | merges `AgentToolRegistry` execution + `ReasoningEngine` execution (P1); `DbQueryExecutor` (P5) |
| Answer generation | **`ResponseComposer`** | merges `composeAnswer`, `AgentRunner` prose, Brief synthesis, `AlertComposerService` (P4) |
| LLM transport | **`AzureOpenAiClient`** | — |
| Experience orchestration (intake + delivery) | **each execution policy** (`ChatService`, `AgentRunner` shell, `ScheduledReportService`, `MorningBriefService`, `AutomationController`) | — |
| Sequencing | **shell today; optional `AnswerEngine` (P6)** | — |

Every responsibility has exactly one owner.

---

## 7. Validation Strategy (per phase)

| Layer | Requirement (applies to every phase) |
|---|---|
| **Unit** | New/changed owner has hand-rolled-fake tests (no DB/Mockito, per repo convention); shared components tested once, not per experience. |
| **Integration** | Governance parity (`GovernanceParityIntegrationTest`), read-only agent flow, chat planner context assembly must pass; re-baseline only where format legitimately changed. |
| **Regression** | Full suite green before merge; no test deleted without a documented behavioral reason. |
| **Live validation** | Payload/execution capture harness on **both** cores (opt-in `-Dnexus.capture.payload.dir`, abort-before-send); identifier-fidelity live probe on the affected experience; prompt byte-diff (legacy vs spine) for grounding phases. |
| **Performance** | Token-count and latency comparison legacy vs spine (grounding phases must not inflate prompt size beyond the existing budget); step-count parity for the loops. |
| **Production readiness checklist (per phase)** | ① Behind a flag or strangler swap. ② Rollback verified in a lower environment. ③ Parity evidence attached. ④ Audit rows unchanged (execution phases). ⑤ Runbook + owner sign-off. ⑥ Legacy path left dormant, not deleted, until the following phase confirms stability. |

---

## 8. Risk Register

| Class | Risk | Mitigation |
|---|---|---|
| **Technical** | Merging two execution paths (P1) drops an audit/async edge case | Extraction-only; parity tests + audit-row diff; flag rollback |
| **Technical** | Chat grounding swap (P3) loses Semantic-Foundation signal (resolutions, value domains, graph) | Carry all signals through `AgentBrain`/contract in P2 first; prompt byte-diff; live A/B; flag |
| **Architectural** | `AgentBrain` accretes orchestration that isn't reasoning | Keep BLR/Semantic as *input collaborators*; reasoning ownership reviewed at P2 exit |
| **Architectural** | New facade (`AnswerEngine`) adds indirection without removing duplication | P6 is optional and gated on measured duplication; must net-delete code |
| **Architectural** | Drift back to two paths if a phase ships half-done | Each phase has hard exit criteria and retires a specific duplication class before the next begins |
| **Migration** | Steward SQL (P5) newly blocked by governance | Shadow-mode comparison; disposition blocked cases before enforcing |
| **Migration** | Long-lived flags become permanent forks | Each flag has a removal step tied to the *next* phase's exit criteria |
| **Testing** | Re-baselining format tests masks real regressions | Byte-diff harness + human review gate for any re-baseline; live probes independent of unit tests |
| **Operational** | Changes don't reach the running app (stale build) | Deploy/restart verification in the readiness checklist; capture harness confirms the live prompt |

---

## 9. Definition of Done

The Unified Answer Engine migration is complete when **all** hold:

1. Every answer-producing responsibility in §6 has exactly one implementation; the retired components are deleted (not merely dormant).
2. Chat and Agent both ground via the shared spine (`AgentBrain` → … → `PromptAssembler`) and execute exclusively through the shared **Runtime**; Reports and Brief inherit via their delegated policy; Workflow/Alert steward SQL runs through the Runtime.
3. `AgentBrain` is the sole reasoning owner; `SqlGovernancePipeline` is unchanged; `ExecutionContract` remains immutable — verified.
4. A demonstrated "add-once" change (e.g. a new grounding rule or validation) is shown live in all experiences without per-experience edits.
5. Full test suite green; live identifier-fidelity and answer-quality probes pass on every experience; audit parity confirmed.
6. All migration flags removed; no legacy answer path remains reachable.

---

## Final Review

**Unnecessary abstractions.** Only two new components are authorized: the shared **Runtime** and **ResponseComposer** — each a single responsibility duplicated today. `AnswerEngine` and `ExecutionStrategy` are deliberately **deferred to optional P6** and gated on a net-code-deletion test; the convergence goal is met by sharing stage owners, so a facade is not assumed. Reports and Brief require **no** new abstraction — they already delegate to Chat/Agent.

**Architectural drift.** The plan moves the Conversational core onto Agent-core components already in the tree; it never stands up a third stack. `AgentBrain` and the execution owner boundaries are re-asserted at each phase's exit criteria.

**Duplicated responsibilities.** §6 maps every responsibility to one owner; §5 retires each duplication class in a specific phase (D1/D3/D4 in P3, D2 in P2, D6 in P4, D8 in P5, execution in P1).

**Missing validation.** Every phase carries a prompt/execution byte-diff or parity gate plus a live probe; grounding phases add token/latency performance checks; a production-readiness checklist gates each deploy.

**Implementation risks.** The highest-risk step (P3, chat grounding) is explicitly sequenced behind P1+P2 so the Semantic-Foundation signal is preserved before the render swaps, and is flag-guarded with legacy fallback.

**Simplification opportunities taken.** (a) No facade unless measured; (b) BLR/Semantic folded as *inputs*, not rewritten; (c) legacy paths kept dormant one phase for cheap rollback; (d) Reports/Brief untouched beyond inheritance; (e) steward-SQL governance reuses the same Runtime rather than a workflow-specific gate.

**Net:** a strangler-fig migration where each phase is independently deployable, retires exactly one duplication class, preserves production behavior under a flag, and re-asserts ADR-0003 ownership — with new abstractions held to the two that are genuinely justified.
