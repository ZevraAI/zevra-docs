---
description: Execution guide for Phase 4 of the Unified Answer Engine migration — converging the duplicated response-composition responsibilities (execution-outcome interpretation and evidence→answer generation) into two small shared collaborators, while every experience keeps its own presentation policy (Chat markdown, Agent ReAct output, Brief JSON, Report HTML, Alert prose).
---

# Phase 4 — Unified Response Composition: Implementation Plan

**Status:** planning only. No code changes are authorized by this document. It is the execution guide for Phase 4 and must be read with the [Master Implementation Plan](unified-answer-engine-implementation-plan.md), whose Phase 4/5 entries it expands.

**Prerequisites (complete):** Phase 1 (`GovernedSqlRuntime`), Phase 2 (`AgentBrain` owns reasoning), Phase 3 (shared grounding + canonical executable identity).

> **Thesis.** Phase 4 is *not* a `ResponseComposer` monolith. The survey below shows the actual duplication is narrow: **row/outcome interpretation** and the **evidence→answer LLM call**. Those become two small, single-purpose shared collaborators.
 Everything downstream — how each experience *shapes* the answer — is presentation policy and stays where it is. The Agent path composes inline during its ReAct loop and is deliberately left alone.

---

## 1. Current Response Pipelines

| Experience | Execution result | NL generation | Evidence | Follow-ups | Output formatting |
|---|---|---|---|---|---|
| **Chat** | `ReasoningEngine` evidence → `evidenceToExecResults` | `composeAnswer` — `buildRowSummary` → **LLM analyst-summary** (two prompts: rows / zero-rows). Also `answerFromMemory`, `answerFromPriorResults` | `reasoningSteps` trace from evidence; `queryData` rows | `buildQuickRefinements` (canned per decision type) | `buildResponse` → `ChatResponse` (markdown answer + steps + quickRefs) |
| **Agent** | `GovernedSqlRuntime` via `AgentToolRegistry` | **Model writes the answer inline** — the ReAct loop's `final_answer` tool; `salvageAnswer` at max-iter. No separate compose method | `ZevraSession.steps` (tool-call trace) | none | `completeSession` → `ZevraSession` (→ `ChatResponse` on routed chat) |
| **Workflow (Automation)** | `DbQueryExecutor` rows | `AiReasonExecutor` — raw `openAi.chat` per AI step | per-step outputs | none | automation step output contract |
| **Brief** | `agentRunner.run` per agent → `finalOutput` (+ injected concrete numbers) | `synthesise` — **single `openAi.chatWithJson`** formatting all agent outputs | agent outputs | none | structured brief JSON |
| **Reports** | `chatService.ask` per question → `ChatResponse` | inherited from Chat | `ChatResponse.queryData` → `ReportSection[]` | inherited | `ReportHtmlComposer.composeEmail` / `composeSlackText` (HTML / Slack) |

**Observations that drive the design:**
- Four experiences run a **separate evidence→answer LLM call** with different prompts: Chat (`composeAnswer`), Brief (`synthesise`), Alert (`AlertComposerService.compose`), and Workflow AI steps (`AiReasonExecutor`). The **prompt/tone is the only real difference** — the call mechanics are duplicated.
- **Execution-outcome interpretation** (rows → counts/distributions/sums) for *response composition* lives in `ChatService.buildRowSummary` (extracted to `ExecutionOutcomeInterpreter` in Step 1); Brief re-invents a slice of it ("inject concrete numbers"); Agent feeds raw rows to the model.

> **Amendment (post-Step 1): a second, independent summarizer exists.** `EvidenceStore.buildRowSummary` (reasoning package) is a *reasoning-oriented* row summarizer that feeds the **planner/evaluator mid-loop** (`buildContextForLlm`), not the final response. It is **not** a byte-identical duplicate of the response summarizer — it uses a different format (`"N row(s). Columns: …"`), a different cardinality threshold (≤ 10 vs ≤ 8), different numeric formatting, and different value filtering. It is therefore **out of Phase 4 scope**: merging it into `ExecutionOutcomeInterpreter` would reconcile two intentionally-different formats — a behavior change requiring its own semantic review, not the byte-identical consolidation this phase performs. Recorded here as an independent consolidation candidate for a future, separately-reviewed step.
- **Follow-ups**, **traces**, and **output shaping** are all experience-specific.
- **Citations** have **no current implementation** anywhere (grep confirms) — so there is nothing to *converge*; it is a potential future shared capability, out of scope for a dedup phase.

---

## 2. Shared Responsibilities (justified)

Only two responsibilities are genuinely duplicated and presentation-free. These become the shared core.

| Shared responsibility | Justification | Owner (new, small) |
|---|---|---|
| **Execution-outcome interpretation** — rows/evidence → outcome class (rows / empty / blocked / error / async) + compact statistical summary (totals, low-cardinality distributions, numeric sums/averages, non-ID) | Deterministic, no presentation, and reusable by every experience that has execution results. Currently in `ChatService.buildRowSummary` + `evidenceToExecResults`; Brief re-derives "concrete numbers"; Agent would benefit (summary beats raw rows for the model). | `ExecutionOutcomeInterpreter` |
| **Evidence→answer generation** — given a question/context, the interpreted evidence, and a **composition policy** (system prompt + response-format hint), produce the natural-language answer via `AzureOpenAiClient` | The LLM-composition call is duplicated across Chat/Brief/Alert/Workflow with only the prompt differing. Consolidating the *mechanics* (message assembly, model call, error fallback) removes duplication; the **prompt stays a per-experience policy input**. | `AnswerComposer` |

**Explicitly NOT shared (and why):**
- **Follow-up recommendations** (`buildQuickRefinements`) — deterministic, tied to Chat decision types and the `/async` slash command. Chat UX policy.
- **Reasoning/step traces** — each experience has its own trace shape (`reasoningSteps` vs `ZevraSession.steps`); the underlying evidence is shared, the *rendering* of it is not.
- **The Agent's inline composition** — the ReAct `final_answer` is the model's own output mid-loop, not a separable compose method. Forcing it through a post-hoc composer would change agent behavior for no dedup gain. It stays; it *may* later consume `ExecutionOutcomeInterpreter` to feed the model better summaries, but that is optional and out of scope.

---

## 3. Experience Responsibilities (presentation policies — preserved)

| Experience | Presentation policy that stays |
|---|---|
| **Chat** | Markdown answer + `ChatResponse` assembly (`buildResponse`) + `buildQuickRefinements` + `reasoningSteps`; the analyst-summary system prompt (markdown rules) is passed to `AnswerComposer` as policy, not owned by it |
| **Agent** | ReAct inline `final_answer` + `ZevraSession` step persistence |
| **Workflow** | Automation step output contract; each step's role |
| **Brief** | Structured brief JSON shape; the synthesis system prompt is Brief's policy |
| **Reports** | `ReportHtmlComposer` email HTML / Slack text |
| **Alert** | Alert prose string; the alert system prompt is Alert's policy |

The invariant: **the shared core emits structured interpretation + natural language; it never emits markdown, HTML, JSON shape, or slash-command text.** All of that is presentation, owned by the experience.

---

## 4. Component Mapping

| Current component | Verdict | Target |
|---|---|---|
| `ChatService.buildRowSummary` | **Merge** | into `ExecutionOutcomeInterpreter` |
| `ChatService.evidenceToExecResults` | **Modify** | feed the interpreter; keep chat's `execResults` shape |
| `ChatService.composeAnswer` | **Modify** | delegate the LLM call to `AnswerComposer` (chat prompt passed as policy); keep row/zero-row prompt selection in Chat |
| `ChatService.answerFromMemory` / `answerFromPriorResults` | **Modify (low priority)** | may delegate to `AnswerComposer` with a memory policy; not required for dedup |
| `ChatService.buildQuickRefinements` | **Retain** | chat presentation |
| `ChatService.buildResponse` / `ChatResponse` | **Retain** | chat presentation |
| `MorningBriefService.synthesise` | **Modify** | delegate LLM call to `AnswerComposer` (brief policy); use `ExecutionOutcomeInterpreter` for the injected numbers |
| `AlertComposerService.compose` | **Modify** | delegate to `AnswerComposer` (alert policy) |
| `AiReasonExecutor` (workflow) | **Modify (optional)** | route AI reasoning steps through `AnswerComposer`; low value, sequence last |
| `AgentRunner` `final_answer` / `salvageAnswer` | **Retain** | inline ReAct composition |
| `ReportHtmlComposer` | **Retain** | report presentation |
| `ExecutionOutcomeInterpreter`, `AnswerComposer` | **New (small)** | the two shared collaborators |

Two new components only — each a single responsibility currently duplicated. No monolith.

---

## 5. Migration Plan (incremental, independently deployable, no large refactor)

**P4.1 — Extract `ExecutionOutcomeInterpreter`.**
- *Objective:* one owner for rows→outcome-class + statistical summary.
- *Files:* new `ExecutionOutcomeInterpreter`; `ChatService` (delegates `buildRowSummary`/classification).
- *Validation:* golden-master unit test — the interpreter's summary is **byte-identical** to the current `buildRowSummary` on a fixture corpus.
- *Rollback:* keep `buildRowSummary` until parity confirmed, then delete.
- *Risk:* Low (pure deterministic extraction).

**P4.2 — Extract `AnswerComposer`; Chat delegates.**
- *Objective:* one owner for the evidence→answer LLM call; prompt passed as policy.
- *Files:* new `AnswerComposer`; `ChatService.composeAnswer` delegates.
- *Validation:* with the same prompt + evidence, output is equivalent; capture-harness on the request payload (message assembly unchanged); live A/B on answer quality.
- *Rollback:* Chat retains its inline call behind a flag until parity confirmed.
- *Risk:* Medium (LLM output; mitigated by unchanged prompt/evidence).

**P4.3 — Brief consumes the composer (JSON mode).**
- *Objective:* `synthesise` uses `NaturalLanguageComposer` in **JSON mode** (brief's synthesis prompt stays as its policy).
- *Files:* `MorningBriefService`.
- *Validation:* brief JSON shape unchanged; the synthesis system prompt is unchanged; a failure still propagates to mark the brief FAILED.
- *Rollback:* per-service revert. *Risk:* Low-Medium.

> **Amendment (during P4.3): two implementation realities.** (1) Brief's `synthesise` has **no fallback** — a failure must propagate so the caller marks the brief *failed*. This drove the composer API to make the fallback optional (`CompositionRequest.jsonPropagating(...)` → propagate; a supplied fallback → mask). (2) Brief's "concrete numbers" come from `extractQueryResults`, which dumps **raw** result JSON from the agent trace — this is **not** the statistical summarization `ExecutionOutcomeInterpreter` performs. Swapping it for the interpreter would change what the synthesis LLM sees (raw JSON → stats), a *behavior change* out of a dedup phase's scope. So Brief consumes **only** the composer in P4.3; the interpreter is intentionally not forced onto Brief. Recorded like the `EvidenceStore` finding: a distinct responsibility, not a duplicate.

> **API evolution (per review guidance):** the composer moved from an overload list to a small request object (`CompositionRequest`) with an explicit `ResponseMode {TEXT, JSON}` rather than boolean flags — introduced here because Brief adds JSON-mode composition, keeping a single `compose(CompositionRequest)` method as more experiences migrate.

**P4.4 — Alert consumes the composer (TEXT mode). ✅ done.**
- *Files:* `AlertComposerService`. *Validation:* alert prose equivalence. *Rollback:* revert. *Risk:* Low.
- *Outcome:* the clean case — a single TEXT-mode call with a deterministic template fallback. `AlertComposerService` swapped `AzureOpenAiClient` for `NaturalLanguageComposer`; its system prompt is now a named `ALERT_SYSTEM_PROMPT` constant passed as **policy**; the template becomes the composer's **lazy** fallback (built only on failure — behavior preserved). The prior `catch (Exception)` narrows to the composer's `catch (RuntimeException)`; equivalent in practice since `chat` throws only unchecked exceptions (it already backs Chat and Brief through the composer). Unit test wires a fake AI through a real composer and proves both the policy-passthrough success path and the template fallback.

> **Invariant recorded (per review guidance): `ResponseMode` is protocol, not presentation.** `ResponseMode` (`TEXT`/`JSON`) represents the expected **protocol** of the model response. It must **never** evolve into a presentation format. Presentation concerns — Markdown, HTML, Slack formatting, email layout, report layout — remain owned by the consuming experience and never enter the shared composer. A new `ResponseMode` constant is justified only by a new model *response protocol*, not by a new way of rendering an answer. Documented on the enum itself in `NaturalLanguageComposer`.

**P4.5 — Evidence-driven review of Workflow and Agent. ✅ done (partial migration).**
- *Method:* migrate only where **measurable duplication of the evidence→answer mechanic** exists without altering semantic responsibility; otherwise record the finding and conclude.
- *Workflow (`AiReasonExecutor`) — MIGRATED.* It reproduced the composer's mechanic exactly: `List.of(ChatMessage.user(...))` + a text/JSON branch over `chat`/`chatWithJson`. Real duplication. Now composes through `NaturalLanguageComposer` in the mode picked from `outputFormat`. Semantics preserved and still owned by the executor: `{{variable}}` templating, the default system prompts, the JSON→`Map` parsing (incl. the `{raw: ...}` fallback for unparseable output), and **propagate-on-failure** (an AI step has no answer fallback — a failed model call fails the automation; expressed via `CompositionRequest.propagating(...)`, null fallback). Unit test covers text passthrough, JSON→Map, the raw-wrap fallback, failure propagation, and the missing-prompt guard.
- *Agent (`AgentRunner`) — NOT MIGRATED (no duplication found).* The Agent's model call is `openAi.chatWithTools(messages, systemPrompt, tools)` over `AgentMessage` (not `ChatMessage`), returning an `AgentToolResponse` (a tool call or a terminal answer) inside the multi-turn ReAct loop. This is a **different mechanic** — tool-calling orchestration — not the evidence→answer `chat`/`chatWithJson` the composer owns. There is nothing to deduplicate; routing it through the composer would change agent behavior for zero gain. Confirms §2's original stance. The optional "feed the interpreter's summary to the model" idea remains genuinely optional and is **not** done: it would *change* what the model sees (a behavior change), which is out of a dedup phase's scope.
- *Also reviewed and excluded (not evidence→answer mechanics):* `ReasoningPlanner`, `ReasoningEvaluator`, `CorrectionDetector`, `TermExtractor`, `PackEntityMapper`, `EnterpriseMapService`, `OnboardingService`, `AutomationGeneratorService`, and Chat's own `getLlmDecision`/`resolveAgent` — all are structured decision/reasoning/extraction calls, explicitly outside the composer per §2/§3.
- *API:* added `CompositionRequest.propagating(userPrompt, systemPrompt, mode)` (mode-agnostic, no-fallback) to serve the executor's dynamic text/JSON mode; `jsonPropagating` now delegates to it.

**P4.6 — Retire duplicated mechanics. ✅ done.**
- *Objective:* no legacy inline row-summarization or evidence→answer LLM-call mechanics survive; nothing retained behind a flag.
- *Outcome:* because every step migrated by **direct replacement** (not flag-gated dual paths), there was no parallel legacy to delete — the duplication was removed at each step. Post-migration sweep confirms:
  - **Row summarization:** the response-side `ChatService.buildRowSummary` is gone (moved to `ExecutionOutcomeInterpreter` in P4.1). The only remaining `buildRowSummary` is `EvidenceStore`'s reasoning-side summarizer — a **distinct** responsibility, intentionally out of scope (see the post-Step-1 amendment), not a duplicate.
  - **Evidence→answer LLM call:** Chat's answer path (`composeAnswer`/`answerFromMemory`/`answerFromPriorResults`), Brief's `synthesise`, `AlertComposerService`, and `AiReasonExecutor` all route through `NaturalLanguageComposer`. None retains a direct `AzureOpenAiClient` for evidence→answer. Chat keeps `AzureOpenAiClient` only for `getLlmDecision`/`resolveAgent` (routing/decisions — out of scope).
  - **Dead code:** the now-unused `AzureOpenAiClient` and `ChatMessage` imports were removed from Brief, Alert, and the executor (grep-verified: zero `ChatMessage` references in all three).
- *Risk:* Low. Full suite green (215 tests).

---

## 5b. Phase 4 Completion Status (as built)

> The two shared collaborators shipped as **`ExecutionOutcomeInterpreter`** and **`NaturalLanguageComposer`** (the composer was named `AnswerComposer` in the tables above during planning; the built name reflects its narrower responsibility — generating natural language from already-interpreted evidence, not composing the overall response).

| Step | Scope | Status |
|---|---|---|
| P4.1 | `ExecutionOutcomeInterpreter` extracted (byte-identical) | ✅ done |
| P4.2 | `NaturalLanguageComposer` extracted; Chat delegates | ✅ done |
| P4.3 | Brief consumes composer (JSON mode; propagate-on-failure) | ✅ done |
| P4.4 | Alert consumes composer (TEXT mode; template fallback) | ✅ done |
| P4.5 | Workflow `AiReasonExecutor` migrated; Agent reviewed → no duplication, not migrated | ✅ done |
| P4.6 | Duplicated mechanics retired (direct replacement; no flags; dead code removed) | ✅ done |

**Composer consumers:** Chat, Brief, Alert, Workflow (`AiReasonExecutor`). **Interpreter consumers:** Chat (Brief intentionally excluded — its "concrete numbers" are raw extraction, not statistical summary). **Left inline by design:** the Agent ReAct loop (`chatWithTools`), and all routing/decision/extraction calls. Tests: 215 green, including golden-master (interpreter), policy-passthrough + mode + fallback/propagate (composer), and per-experience delegation (Alert, Workflow).

---

## 6. Response Parity Strategy

Parity is proven per dimension against a stored corpus, legacy-vs-shared:

| Dimension | Method | Pass condition |
|---|---|---|
| **Execution outcomes** | interpreter output vs `buildRowSummary` on fixtures | **byte-identical** (deterministic) |
| **Evidence / queryData** | unchanged data flow | identical rows/steps |
| **Citations** | n/a | no current implementation — nothing to regress |
| **Natural language** | live A/B, same prompt+evidence | equivalent answers (human review + no correctness regression) |
| **Follow-ups** | `buildQuickRefinements` untouched | byte-identical |
| **Formatting by experience** | ChatResponse / brief JSON / report HTML / alert string diffs | **byte-identical** (presentation unchanged) |

Automated subset (interpreter output, evidence, follow-ups, formatting) is hard pass/fail; only the NL dimension needs live judgment, and only because the model is nondeterministic — the *inputs* to it are proven unchanged.

---

## 7. Legacy Retirement

**Becomes obsolete after parity (delete in P4.6):**
- `ChatService.buildRowSummary` (moved to the interpreter)
- the inline LLM-call mechanics inside `composeAnswer`, `MorningBriefService.synthesise`, `AlertComposerService.compose` (moved to `AnswerComposer`; their **prompts remain** as policy inputs)

**Remains permanently (presentation policy):** `buildQuickRefinements`, `buildResponse`/`ChatResponse`, `ReportHtmlComposer`, brief JSON assembly, `ZevraSession` persistence, the Agent inline `final_answer`.

**Rule:** no deletion until the §6 battery passes and the shared path has run one validation window.

---

## 8. Exit Criteria

- **Architecture:** exactly two shared composition collaborators; every evidence→answer LLM call and every row-summarization routes through them; presentation stays per experience.
- **Validation:** §6 battery green; interpreter byte-identical; NL equivalence confirmed live; all experience outputs byte-identical in shape.
- **Testing:** unit tests for the interpreter (golden master) and composer (policy passthrough, fallback); existing experience tests green.
- **Cleanup:** duplicated row-summarization and LLM-call mechanics deleted; no dead code.
- **Production readiness:** shared path live one validation window with no regression in answer quality, latency, or error rate; rollback verified.

---

## Final Review

**Duplicated ownership.** After Phase 4, the interpreter is the **only** row-summarizer and `AnswerComposer` the **only** evidence→answer LLM-call mechanic. The §4 table maps every current composer to Merge/Modify/Retain so none survives twice.

**Unnecessary abstractions.** Two small single-purpose collaborators, each justified by real, measured duplication across ≥3 experiences — explicitly **not** a `ResponseComposer` monolith. The Agent is left inline (no dedup to gain). Citations are excluded (nothing to converge). Follow-ups/traces/formatting stay experience-owned.

**Hidden coupling.** The risk is the composer absorbing an experience's prompt or format. Mitigation: the system prompt and format hint are **inputs** (composition policy), never constants inside the composer; a test asserts the composer emits exactly the policy-supplied framing and nothing experience-specific.

**Presentation leakage.** Hard invariant: the shared core emits structured interpretation + plain NL only — no markdown, HTML, JSON shape, or slash-command strings. A review gate on the two new classes checks for any presentation string.

**Rollout risk.** Every step is additive and per-experience; the highest-risk step (P4.2, Chat's LLM call) ships behind a flag with legacy fallback and a live A/B before default.

**Rollback risk.** No legacy deleted before P4.6; each Modify step is a revertible delegation. The interpreter extraction is byte-identical, so its rollback is trivial.

**Simplifications taken.** (a) No monolith; (b) Agent untouched; (c) citations deferred; (d) Reports inherit entirely through Chat and need no Phase-4 work beyond what Chat's delegation gives them; (e) `answerFromMemory`/`answerFromPriorResults` and Workflow AI steps are optional/last, not forced.

**Constraint check.** No change to `AgentBrain`, `GovernedSqlRuntime`, `ExecutionContract`, `PromptContext`, or `PromptAssembler` — Phase 4 lives strictly in the response tail, downstream of execution.
