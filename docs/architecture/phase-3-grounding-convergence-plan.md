---
description: Execution guide for Phase 3 of the Unified Answer Engine migration — moving the Conversational (Chat) path off its bespoke schema rendering and onto the shared AgentBrain → ExecutionContract → PromptContextBuilder → PromptAssembler grounding pipeline, without regressing grounding quality or production behaviour.
---

# Phase 3 — Grounding Convergence: Implementation Plan

**Status:** planning only. No code changes are authorized by this document. It is the execution guide for Phase 3 and must be read with the [Unified Answer Engine Master Implementation Plan](unified-answer-engine-implementation-plan.md), whose Phase 3 entry it expands.

**Prerequisites (complete):** Phase 1 (shared `GovernedSqlRuntime`) and Phase 2 (`AgentBrain` absorbs the Semantic Foundation; `AgentBrain.resolve(agentId, connectionKeys, domainKeys, question)` already exists and already returns resolutions + literal scope).

> **The headline finding.** Phase 3 is *not* a renderer swap. Chat's grounding carries several load-bearing facts that the agent's `PromptAssembler` does not render today — above all `connection_key`, which `ReasoningPlanner` is contractually required to emit in its JSON plan. A naive swap would break conversational SQL generation on the first step. The plan below closes each gap explicitly before any flag is flipped.

---

## 1. Current Chat Grounding Pipeline

### Call sequence

```
ChatController → ChatService.ask
  └─ ZevraAgentRouter.route                      (miss ⇒ conversational path)
  └─ resolveAgent(...)                           → NexusAgent (domainKeys, connKeys)
  └─ BusinessLanguageResolver.resolve(raw, domainKeys)      → ResolvedQuestion
  └─ EnterpriseMapService.operationalContext(domainKeys, connKeys, raw)
                                                 → { entityContext, entityBlocks, … }
  └─ SemanticService.semanticContextWithBindings(domainKeys, raw)
                                                 → SemanticContext { contextText, bindings, termLinesByObjectKey }
  └─ KnowledgeGraphService.buildGraphContext(domainKeys)   → graph text
  └─ ChatService.buildContextSummary(...)        → schemaCtx  (assembled below)
        ├─ resolved.renderPromptBlock()                     RESOLUTIONS block
        ├─ resolved.renderLiteralCandidatesBlock()          LITERAL CANDIDATES block
        ├─ filterGraphContext(graph, question, expandedTokens)
        ├─ "=== TABLE SCHEMA ===" + assembleEntityContext(...)
        │       ├─ rankEntityBlockKeys(question, blocks, bindings, graph, expandedTokens)
        │       ├─ attach "Business terms: …" per block (termLinesByObjectKey)
        │       └─ truncateEntityContext(ordered, maxEntityContextChars = 1500)
        ├─ supporting context (memory count, semantic-layer flag, findings, anomaly, prior)
        └─ conversation history (last 4 turns)
  └─ (+ playbook text, + LearningContextBuilder.build(...) appended to schemaCtx)
  └─ ReasoningEngine.reason(question, enrichedQ, sessionKey, schemaCtx, runKey, userEmail,
                            forceAsync, ChatService.buildLiteralScope(resolved))
        └─ ReasoningPlanner.nextStep(enrichedQ, schemaCtx, evidence)
              └─ buildPrompt: "Question: …\n\nApproved schema:\n" + schemaCtx + "\n\nEvidence so far:\n…"
              └─ aiClient.chat(user=prompt, system=SYSTEM_PROMPT + SqlIdentifierGuidance)
        └─ GovernedSqlRuntime.execute(Request.forPlanner(…))     ← Phase 1, shared
```

### Components and responsibilities

| Component | Responsibility in grounding |
|---|---|
| `ChatService.ask` | Orchestrates; owns turn/history and which blocks appear |
| `BusinessLanguageResolver` | Produces `ResolvedQuestion` (resolutions, expansion tokens, literal candidates) |
| `EnterpriseMapService.operationalContext` | Renders **per-table blocks** (`entityBlocks`) + concatenated `entityContext` |
| `SemanticService.semanticContextWithBindings` | Entity/vocabulary→table bindings + per-object business-term lines |
| `KnowledgeGraphService` | Entity relationships / **join guidance** (the only source of joins today) |
| `ChatService.assembleEntityContext` | Ranks blocks by relevance, attaches business terms, applies the char budget |
| `ChatService.rankEntityBlockKeys` | Tier-1 bindings + join-neighbours, tier-2 keyword match, tier-3 renderer order |
| `ChatService.buildContextSummary` | Composes every block into the single `schemaCtx` string |
| `ChatService.buildLiteralScope` | Delegates to `AgentBrain.literalScopeOf` (Phase 2) |
| `ReasoningPlanner` | Wraps `schemaCtx` as "Approved schema" in the user message |

### The per-table block, exactly as rendered today

```
Table: <schema>.<table> (<businessName>)
Purpose: <purpose>
connection_key=<connKey> (use this exact value in your SQL plan)
Identifier columns: <csv>
Status columns: <csv>
Usage: <usageGuidance>
Filter guidance: <filterGuidance>
Avoid: <avoidGuidance>
Row limit: <n>
Columns:
  - <column> (<dataType | authoritative-domain values | dataType; observed: …>): <businessMeaning> [identifier, status, filterable]
Business terms: <term>; <term>
```

Note also the **fallback connection substitution**: if a `DataObject`'s stored connection no longer exists, the renderer substitutes the first active connection so the planner always receives a usable key.

---

## 2. Target Grounding Pipeline

```
AgentBrain                    → resolves business meaning for the chat scope
   ↓                             (connectionKeys + domainKeys + BLR resolutions, Phase 2)
EnterpriseSemanticAssembler   → Enterprise Map ⇒ canonical SemanticModel
   ↓
ExecutionContractBuilder      → immutable ExecutionContract (SemanticView + ExecutionBindings
   ↓                             + precomputed approvedAssets)
PromptContextBuilder          → model-agnostic PromptContext (meaning + physical bindings,
   ↓                             ranked, within a rendering budget)
PromptAssembler               → the TABLE SCHEMA grounding text (+ SqlIdentifierGuidance)
   ↓
ReasoningPlanner              → composes: question + [chat policy blocks] + grounding + evidence
```

**Stage responsibilities**

| Stage | Responsibility | Owns | Does not |
|---|---|---|---|
| `AgentBrain` | Business reasoning: resolve objects for the scope, apply resolutions, rank by relevance | reasoning, ranking | render, narrow the approved surface, execute |
| `EnterpriseSemanticAssembler` | Project Enterprise Map → `SemanticModel` | semantic assembly | reasoning |
| `ExecutionContractBuilder` | Compile the immutable contract + `approvedAssets` | bindings, identity | reasoning, rendering |
| `PromptContextBuilder` | Contract → `PromptContext`; apply the **rendering budget** in ranked order | prompt-context shape | change the contract or the approved surface |
| `PromptAssembler` | Render `PromptContext` → grounding text | rendering, identifier fidelity | decide meaning |
| `ReasoningPlanner` | Compose grounding + chat policy blocks + evidence; generate one SQL step | loop policy | schema rendering |

**Scope boundary (deliberate).** Only the **TABLE SCHEMA** portion converges. The RESOLUTIONS block, LITERAL CANDIDATES block, knowledge-graph/join guidance, memory/findings/anomaly, conversation history, playbook, and learning context **remain chat execution policy**, composed *around* the shared grounding. This matches the master plan's "Retired" list exactly and keeps `PromptContext` from becoming a dumping ground.

---

## 3. Component Mapping

| Current component | Future owner | Action |
|---|---|---|
| `ChatService.assembleEntityContext` | `PromptContextBuilder` (ranking + budget) | **Retire** |
| `ChatService.rankEntityBlockKeys` / `rankEntityBlocks` | `AgentBrain` (relevance ranking) | **Retire** |
| `ChatService.truncateEntityContext` | `PromptContextBuilder` (budget) | **Retire** |
| `EnterpriseMapService` per-table block rendering (`entityContext`/`entityBlocks`) | `PromptAssembler` | **Retire** (see §6 for other callers) |
| `EnterpriseMapService.operationalContext` (non-schema parts) | `EnterpriseMapService` | **Retain** |
| `ReasoningPlanner.SYSTEM_PROMPT` schema wording | `PromptAssembler` + `SqlIdentifierGuidance` | **Modify** (drop schema-describing lines) |
| `ReasoningPlanner.buildPrompt` | `ReasoningPlanner` | **Modify** (compose shared grounding) |
| `ChatService.buildContextSummary` | `ChatService` | **Modify** (schema section sourced from PromptAssembler) |
| `ChatService.buildLiteralScope` | `AgentBrain.literalScopeOf` | **Retire** (delegation shim from Phase 2) |
| RESOLUTIONS / LITERAL CANDIDATES blocks | `ChatService` (policy) | **Retain** |
| `KnowledgeGraphService` + `filterGraphContext` | `ChatService` (policy) | **Retain** |
| Memory / findings / anomaly / history / playbook / learning | `ChatService` (policy) | **Retain** |
| `SemanticService.semanticContextWithBindings` | `AgentBrain` inputs (bindings) / `ChatService` (terms) | **Modify** — bindings feed ranking; business terms move into `PromptContext.guidance` |
| `PromptContext` / `PromptContextBuilder` / `PromptAssembler` | unchanged owners | **Modify** (carry the load-bearing fields, §4 Step 3) |
| `GovernedSqlRuntime` | unchanged | **Retain** (gains chat's contract, §4 Step 2) |
| `ReasoningEngine` | unchanged | **Retain** |
| `SqlGovernancePipeline` | unchanged | **Retain — untouched (ADR-0003 A2)** |

### Grounding-content gap analysis (what must be closed before any swap)

| Fact in chat's block today | In `PromptContext` today? | Load-bearing? | Resolution |
|---|---|---|---|
| `connection_key=…` | ❌ | **Yes — planner must emit it** | Add to `PromptContext` (available on `ExecutionTarget.connectionKey`) |
| `schema.table` qualification | ❌ (bare table) | Yes for cross-schema SQL | Add schema (available on `ExecutionTarget.schema`) |
| Column **data type** | ❌ (renders role) | Yes — predicates/casts | Add type to the attribute projection |
| **Value-domain values** inline (PRO-10/24) | ❌ | **Yes — literal fidelity** | Render from the Phase-2 literal scope / attribute domain |
| Column flags `[identifier, status, filterable]` | partial (role only) | Medium | Derive from `AttributeRole` + flags |
| Column `businessMeaning` | ✅ (business name) | Yes | Already covered |
| `Purpose`, `Identifier/Status columns`, `Row limit` | ❌ | Medium | Fold into object `guidance` |
| Usage / Filter / Avoid guidance | ✅ (concatenated `guidance`) | Yes | Already covered |
| `Business terms:` companion line | ❌ | Medium (vocabulary recall) | Append into object `guidance` |
| Relevance ranking | ✅ (`AgentBrain`) | Yes | Already covered |
| Char budget (1500) | ❌ (unbounded) | **Yes — cost/latency** | Add budget to `PromptContextBuilder` |
| Deleted-connection fallback | ❌ | Medium | Preserve in the assembler/binding step |

---

## 4. Migration Steps

Each step is independently deployable and testable. **No step deletes legacy code**; deletion happens only in Step 6.

### Step 1 — Chat compiles an ExecutionContract (shadow, unused)
- **Objective:** `ChatService` calls `AgentBrain.resolve(agentKey, connKeys, domainKeys, raw)` → `ExecutionContractBuilder.compile(...)`, records `contractId` in the trace. Prompts and execution unchanged.
- **Files:** `ChatService`.
- **Risk:** Low (additive; output unused).
- **Validation:** contract non-empty for a tenant with mapped objects; `approvedAssets` ⊇ the tables the legacy context lists; `contractId` present in the run trace; full suite green; prompt bytes unchanged.
- **Rollback:** delete the call.

### Step 2 — Chat gains the business-object gate (shadow → enforce)
- **Objective:** pass the contract into `GovernedSqlRuntime.Request.forPlanner(...)` so chat gets the A14 gate.
- **Files:** `ReasoningEngine` (thread the contract), `GovernedSqlRuntime.Request.forPlanner` (accept it), `ChatService`.
- **Risk:** **High** — a contract surface narrower than governance's allow-list would reject queries that work today.
- **Validation:** **shadow mode first** — log `would-reject` without enforcing, over a real question corpus; require zero unexpected rejections before enforcing. Then enforce behind a flag.
- **Rollback:** stop passing the contract (gate becomes inert — it is already `null`-conditional).

### Step 3 — Enrich `PromptContext` with the load-bearing facts
- **Objective:** close every ❌ in the §3 gap table: connection key, schema, column data type, value-domain values, flags, object purpose/row-limit/terms folded into guidance, plus a **rendering budget** in `PromptContextBuilder` (ranked order, budget-bounded).
- **Files:** `PromptContext`, `PromptContextBuilder`, `PromptAssembler`, `ExecutionContractBuilder` (only if a field is not already on `ExecutionBindings`).
- **Critical constraint:** the **agent prompt must stay byte-identical**. New fields render only when populated, and `PromptContextBuilder` populates them only for the chat build. The budget must never shrink `approvedAssets` — the gate keeps the full surface; only the *rendering* is bounded.
- **Risk:** Medium.
- **Validation:** `AzurePayloadCaptureTest` still 4550 bytes (agent unchanged); new unit tests per field; budget test (N objects → bounded output, ranked order preserved).
- **Rollback:** fields unpopulated ⇒ renderer emits nothing new.

### Step 4 — Render chat's TABLE SCHEMA from `PromptAssembler` (behind a flag)
- **Objective:** `buildContextSummary` sources the `=== TABLE SCHEMA ===` section from the shared pipeline when `nexus.chat.grounding=spine`; legacy path remains the default.
- **Files:** `ChatService`.
- **Risk:** **Highest in Phase 3.**
- **Validation:** the full §5 parity battery, legacy vs spine, over a question corpus.
- **Rollback:** flip the flag to `legacy`.

### Step 5 — Trim `ReasoningPlanner` and flip the default
- **Objective:** remove schema-describing lines from `SYSTEM_PROMPT` now that `PromptAssembler` owns them; default the flag to `spine` after live validation.
- **Files:** `ReasoningPlanner`, config.
- **Risk:** Medium.
- **Validation:** live A/B on answer quality + identifier fidelity; SSE trace parity; token/latency comparison.
- **Rollback:** flag back to `legacy`.

### Step 6 — Retire legacy (only after parity is proven)
- **Objective:** delete the retired components in §6.
- **Files:** `ChatService`, `EnterpriseMapService`, tests.
- **Risk:** Low once unreferenced.
- **Validation:** compile + full suite; no references remain; flag removed.
- **Rollback:** revert the deletion commit.

---

## 5. Prompt Parity Strategy

**Byte-identity is impossible and is not the goal.** The legacy and target renderings are deliberately different formats. Parity here means **no loss of grounding information and no regression in grounding quality**, proven by a fixed battery run legacy-vs-spine over a stored **question corpus** (≥ 30 real questions across ≥ 3 tenants, including agentless and domain-scoped).

| Dimension | Method | Pass condition |
|---|---|---|
| **Payload comparison** | Capture both transmitted payloads via the existing `-Dnexus.capture.payload.dir` harness | Structural diff reviewed; no *missing* facts in spine output |
| **Prompt comparison** | Section-by-section diff of `schemaCtx` | Every non-schema block byte-identical; schema section differs only in format |
| **Token count** | Tokenise both groundings | Spine ≤ legacy × 1.10; never exceeds the configured budget |
| **Identifier fidelity** | Live probe (as used in prompt hardening), N samples/question | ≥ legacy rate; zero substituted/invented identifiers in generated SQL |
| **Retrieval fidelity** | Compare ranked object order: legacy `rankEntityBlockKeys` vs `AgentBrain` ranking | Top-K overlap ≥ 0.9; every legacy tier-1 (binding-matched) object present in spine |
| **Knowledge-graph context** | Diff the graph block | **Byte-identical** (unchanged, chat-composed) |
| **Business-object coverage** | Set of tables in legacy grounding vs spine (within budget) | No object present in legacy and absent in spine at equal budget |
| **Business-attribute coverage** | Set of `table.column` in legacy vs spine | 100% of legacy columns present, **and** every domain-bearing column still shows its legal/observed values |
| **connection_key presence** | Assert per rendered object | Present for every object; deleted-connection fallback preserved |
| **End-to-end quality** | Live A/B on the corpus; human review of answers | No regression in correctness/among reviewers; zero governance-block or gate-rejection increases |

**Harness:** a repeatable comparison test that, for each corpus question, builds both groundings and emits a per-dimension report. This is the artefact attached to the Step-4/5 sign-off. Structural metrics (coverage, ranking overlap, tokens, connection_key) are automated pass/fail; identifier fidelity and answer quality require live runs.

---

## 6. Legacy Retirement Plan

**Becomes unused after Step 5 (delete in Step 6):**
- `ChatService.assembleEntityContext` (all 3 overloads), `rankEntityBlocks`, `rankEntityBlockKeys`, `truncateEntityContext`, `TABLE_HEADER` / `JOIN_TABLE` patterns
- `ChatService.buildLiteralScope` (the Phase-2 delegation shim) — call `AgentBrain.literalScopeOf` directly
- `EnterpriseMapService` per-table block rendering (`entityContext` / `entityBlocks`) — **verified: `operationalContext` has exactly one production consumer, `ChatService` (line 294)**, so the schema-block rendering is chat-only and can be retired outright. `EntityBlocksRenderingTest` exercises the renderer directly and is retired with it. Retain the non-schema parts of `operationalContext` only if a caller emerges; otherwise the whole method retires with chat's last use.
- Schema-describing lines in `ReasoningPlanner.SYSTEM_PROMPT`

**Remains temporarily (removed with the flag, Step 6):**
- The legacy branch in `buildContextSummary` and the `nexus.chat.grounding` flag

**Remains permanently (chat policy):** RESOLUTIONS / LITERAL CANDIDATES blocks, graph context + `filterGraphContext`, memory/findings/anomaly, conversation history, playbook, learning context.

**Rule:** no deletion until the §5 battery has passed on the corpus *and* the spine default has run in production for one validation window.

---

## 7. Risk Assessment

| Class | Risk | Mitigation |
|---|---|---|
| **Architectural** | `PromptContext` bloats into a catch-all | Hard scope boundary (§2): only TABLE SCHEMA converges; policy blocks stay in chat |
| **Architectural** | Budget/ranking logic drifts back into `ChatService` | Budget lives in `PromptContextBuilder`; ranking in `AgentBrain` — asserted at step exit |
| **Grounding** | Silent loss of a fact (types, domains, purpose, terms) | §3 gap table is a checklist; coverage metrics are automated pass/fail |
| **Grounding** | Budget drops an object the question needed | Ranking overlap metric; budget ≥ legacy effective budget; contract still approves all |
| **Prompt** | Agent prompt accidentally changes | `AzurePayloadCaptureTest` byte assertion (4550) is a gate on every step |
| **Prompt** | `connection_key` missing ⇒ planner emits an invalid plan | Explicit per-object assertion; Step 3 closes it before Step 4 |
| **Semantic** | Value-domain values lost ⇒ literal fidelity regresses | Domain-coverage metric; `LiteralValidator` suite unchanged; literal scope already flows (Phase 2) |
| **Semantic** | Resolution runs twice (chat + AgentBrain) | Step 1 must **replace** `ChatService`'s direct `businessLanguageResolver.resolve` call, not add a second — explicit step check |
| **Retrieval** | Join guidance lost (relationships are empty in the semantic model) | Graph block stays chat-composed; do **not** rely on `BusinessObject.relationships` in Phase 3 |
| **Execution** | New gate rejects previously-working queries | Shadow mode before enforcement (Step 2) |
| **Testing** | Re-baselining format tests hides regressions | Byte-diff + human review gate for any re-baseline; live probes independent of unit tests |
| **Operational** | Change doesn't reach the running app | Rebuild/restart verification; capture harness confirms the live prompt |

---

## 8. Exit Criteria

**Architecture**
- Chat's TABLE SCHEMA grounding is produced solely by `AgentBrain → EnterpriseSemanticAssembler → ExecutionContractBuilder → PromptContextBuilder → PromptAssembler`.
- `AgentBrain` is the only reasoning/ranking owner; resolution happens **once** per chat request.
- `GovernedSqlRuntime` remains the only execution path; `SqlGovernancePipeline` unchanged.
- No new abstraction beyond the `PromptContext` field additions and the rendering budget.

**Behaviour parity**
- Full §5 battery passed on the corpus; graph and all policy blocks byte-identical; 100% object/attribute/domain coverage; tokens within budget; identifier fidelity ≥ legacy.
- Agent path byte-identical (payload capture unchanged).

**Testing**
- Full suite green; new tests for each added `PromptContext` field, the budget, and chat contract compilation; parity harness committed and repeatable.

**Production validation**
- Spine default live for one validation window with zero increase in gate rejections, governance blocks, or error rates; live identifier-fidelity probe passes on the chat path.

**Cleanup**
- §6 legacy deleted (subject to the `EnterpriseMapService` caller check), flag removed, no dead references.

---

## Final Review

**Missing steps — found and added.** Three gaps were caught while writing: (i) `connection_key` is a *hard* requirement of the chat planner's JSON contract and is absent from `PromptContext` — a naive swap breaks step one of every conversational query; (ii) chat's rendering is **budgeted** (1500 chars) while `PromptAssembler` is unbounded, so a large tenant would balloon the prompt; (iii) value-domain values are rendered inline in chat's column types (PRO-10/24) and would be silently lost, regressing literal fidelity. All three are now explicit Step-3 blockers with coverage metrics.

**Architectural drift.** The scope boundary in §2 is the main guard: only the schema section converges. Ranking stays in `AgentBrain`, budget in `PromptContextBuilder`, policy blocks in `ChatService`. Without that line, Phase 3 would quietly turn `PromptContext` into the whole chat prompt.

**Hidden dependencies.** (a) `EnterpriseMapService.operationalContext` was checked rather than assumed: it has exactly one production consumer (`ChatService`), so the block renderer is chat-only and retires outright — one less unknown at implementation time. (b) Join guidance comes from the knowledge graph, **not** from `BusinessObject.relationships` (still empty), so the graph block must not be retired. (c) The deleted-connection fallback is real behaviour that must survive. (d) `ChatService` currently resolves the question itself; Step 1 must move that call into `AgentBrain`, not duplicate it.

**Unnecessary abstractions.** None introduced. No new classes; `PromptContext` gains fields, `PromptContextBuilder` gains a budget parameter, and one temporary config flag. The one judgement call — keeping the agent rendering byte-identical by populating new fields only for the chat build — is a migration-safety measure; unifying the two renderings is a candidate follow-on once both are validated, not Phase 3 work.

**Rollback gaps.** Every step is flag- or call-reversible and no legacy code is deleted before Step 6. The riskiest change (the gate) is shadow-mode-first.

**Validation gaps.** Byte-identity being impossible, §5 replaces it with an information-coverage battery plus live quality checks — with the automated subset (coverage, ranking overlap, tokens, `connection_key`) as hard pass/fail so sign-off is not purely a judgement call.

**Simplification taken.** Converging only the schema section removes the largest source of risk and work; Reports and Brief inherit automatically through Chat/Agent and need no Phase 3 work at all.
