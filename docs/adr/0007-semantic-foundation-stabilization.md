---
description: ADR-0007 — Capability Stabilization Decision Record for the Semantic Foundation, the platform's canonical business-meaning layer: KEEP/STABILIZE/DEFER decisions, the protected deterministic-before-AI / referents-only / annotate-never-substitute / fail-open model, the accepted production work (promotion review gate, vocabulary tier provenance, agentless resolution, semantic observability), and the exit criteria for declaring the capability STABLE.
---

# ADR-0007: Semantic Foundation — Capability Stabilization Decision

| | |
|---|---|
| **Status** | Proposed (becomes Accepted on approval of the story set in §9) |
| **Date** | 2026-07-11 |
| **Deciders** | Zevra platform team |
| **Scope** | The Semantic Foundation only — business meaning: entities and physical bindings, operational vocabulary, value domains, semantic roles, learned mappings, Business Language Resolution, Deterministic Literal Resolution, the learning lifecycle, and the metadata that turns business language into governed execution. The Knowledge Graph, AI Memory, Connection Registry, Conversation History, Workflow Automation, Analysts, and SQL Governance are referenced only to establish ownership boundaries. |
| **Inputs** | `architecture/semantic-foundation.md` (documentation + ratified constitution v1.0 it cites), ADR-0002–0006, and code verification of the resolution/learning path (`BusinessLanguageResolver`, `SemanticLearningService`, `OperationalVocabulary`, `SemanticService`, `reasoning/LiteralValidator`). **Where documentation and implementation differ, the implementation is treated as source of truth** and divergences are recorded for the page to correct. |
| **Relationship to prior ADRs** | ADR-0002 named the Foundation the intelligence engine's first pipeline stage and **explicitly deferred the learning-promotion review gate and vocabulary tier provenance to this record** (its §4 Learning-integration row). ADR-0003 recorded the agent runtime's bypass of the Foundation as owned by the Agents record (its A5/A9), not here. ADR-0005/0006 established the data-access and knowledge foundations; the Semantic Foundation is the third foundation — the **meaning** foundation. |
| **Supersedes** | — |
| **Superseded by** | — |

Once accepted, this capability is not reopened for architecture review unless implementation changes significantly; changes to these decisions supersede this record, never edit it.

---

## 1. Executive Decision

## **STABILIZE BEFORE PRODUCTION.**

This is **architecturally the most mature capability reviewed to date, and the only one with no Critical defect.** It carries a ratified constitution, a disciplined deterministic-before-AI contract, referents-only resolution, annotate-never-substitute, and a fail-open/zero-cost guarantee — and code verification confirms the design is faithfully implemented, not aspirational. There is no security exposure here: the Foundation produces *annotations and candidate values*, holds no credentials, opens no connection, executes no SQL, and — verified — **the AI never writes to a semantic store**. Everything it influences still passes the governance chain downstream. Its gaps are therefore **governance-trust and coverage**, not safety:

- **Learning auto-promotes to curated vocabulary with no review gate (the constitution's own "most significant contract gap").** Verified: the nightly job (`SemanticLearningService.runMaintenanceForCurrentSchema`, lines 284–299) calls `semanticService.createTerm(... status ACTIVE ...)` directly for every mapping crossing use-count ≥ 10 / confidence ≥ 0.8 — machine-learned language becomes authoritative **company-tier** vocabulary with no human approval. The constitution requires "thresholds *plus human review*; nothing promotes itself." Stewardship is post-hoc (promoted rows are editable) but the gate does not exist. For a capability whose entire value proposition is *auditable, steward-owned, deterministic meaning*, silent self-promotion into the authoritative tier is the one gap a careful enterprise review will flag. This is the exact item ADR-0002 deferred here. (SF1)
- **Vocabulary carries no tier provenance.** Verified: `OperationalVocabulary` has no `createdBy`/tier field, and `BusinessLanguageResolver` (lines 186–189) hardcodes every vocabulary row to the `company` tier. Pack-provenance flows through entities (`tierOf`) but not vocabulary — so vendor-pack language is indistinguishable from tenant-curated language in prompts and traces, and the trust ladder the architecture advertises is partially unlabelled. The code seam exists; the column does not. (SF2)
- **Resolution silently does nothing without a domain scope.** Verified: `BusinessLanguageResolver.resolve` returns the empty result when `domainKeys` is null/empty (line 117). Questions handled without an agent get *zero* semantic help — the least-configured tenants get the least benefit, invisibly. (SF3)
- **No observability on the meaning layer.** No metrics for resolution hit/miss, literal-validation rejects/hard-blocks, promotion/purge counts, or fail-open activations — a fail-open architecture with no telemetry on how often it fails open, and an auto-promotion loop with no dashboard for what it admitted. (SF4)

**Not Major Rework — emphatically.** The contracts are correct and worth protecting verbatim; the work is to *finish the constitution* (the review gate), *complete the provenance seam* (the column), *widen coverage* (agentless resolution), and *instrument* (observability) — around an untouched core. Nothing about resolution, validation, or the store model needs redesign.

**Position verdict (the question this review was asked):** the Semantic Foundation is the platform's **canonical business-meaning layer** — the single, tenant-owned source of what data *means*, and the deterministic machinery that translates business language into normalized, governed platform meaning before any model reasons. It is a **foundational platform pillar**, the meaning counterpart to the Connection Registry's data-access foundation (ADR-0005) and AI Memory's knowledge foundation (ADR-0006), and like both it is advisory and fail-open — its absence costs nothing (byte-identical baseline), its presence adds only correctness and trust. §11 fixes its identity as a standing constraint.

## 2. Vision Alignment (ownership, not design)

Permanent architectural positions, answering the fourteen questions of record:

1. **Permanent responsibility.** To be the platform's canonical business-meaning layer: hold, as tenant-owned data, what physical data *means* (entities and bindings, vocabulary, value domains, semantic roles, learned language), and deterministically resolve a user's words to **existing referents** — with trust-tier provenance — before any model reasons, then validate the model's literal choices against persisted domains before execution.
2. **Is it the canonical source of business meaning? Yes.** It is *the* source of truth for what data means; meaning is tenant data written only through owned lifecycles, and nothing else in the platform is authorized to assert business meaning.
3. **What belongs here permanently.** Business entities and their physical bindings; operational vocabulary; value domains and their authority levels; column semantic roles and provenance; learned mappings and the learning lifecycle; the deterministic resolution and literal-validation machinery; trust-tier provenance; the discovery gates that govern what may *become* a value.
4. **What must never belong here.** Physical data (the customer database is the truth of *data*, never of *meaning*); query execution and execution governance (the SQL chain); interpretation and reasoning (the model's, which the Foundation constrains and verifies but never performs); universal language (common abbreviations/general English live in the model and are never shipped as dictionaries); unstructured document knowledge (AI Memory); a relationship/navigation engine (the Knowledge Graph's concern); user identity/authorization (Auth); connection credentials (the registry).
5. **Does it duplicate AI Memory? No.** AI Memory is unstructured document text retrieved by vector similarity; the Foundation is structured business meaning resolved by deterministic exact-match to referents. Complementary substrates — one grounds answers in written knowledge, the other translates language to governed data meaning. Neither can do the other's job.
6. **Does it duplicate the Knowledge Graph? No — and the boundary needs stating.** The Foundation owns **entities and their physical bindings** (meaning: *this concept lives in that table*). The Knowledge Graph owns **relationships between entities** (navigation: *how concepts connect*). Today the graph is hosted adjacent to the Foundation and consumed as *advisory context only* — it participates in neither resolution nor validation. The Foundation must never become the relationship/navigation engine; the graph's own record (out of scope here) governs it. No duplication: meaning vs. relationships.
7. **Does it duplicate SQL Governance? No.** The Foundation validates that a literal *exists* (semantic correctness — is `'Texas'` a legal value of this column?); SQL Governance validates that a query is *safe and permitted* (execution safety — RLS, masking, contracts). Two different questions asked at two different points. The literal validator is the Foundation's contract; it is *invoked* inside the reasoning loop and *enforced* before the governance chain runs — distinct from, and upstream of, governance.
8. **Should every AI capability obtain business meaning through this layer? Yes.** That is the canonical-source claim: all business meaning flows through the Foundation, so it is resolved deterministically, provenance-tagged, and auditable. Any capability asserting meaning another way is, by definition, mis-scoped.
9. **Can anything bypass it? Yes today — one bypass, dispositioned elsewhere.** The Autonomous Agent runtime reads operational vocabulary directly and live-samples status values, engaging neither resolution, bindings, nor the value-domain gates (ADR-0003 §4, its A5). That bypass is owned by the Agents record; this record names it and holds the contract the agent runtime must eventually conform to. No *other* consumer bypasses the Foundation.
10. **Foundational platform capability or supporting service? Foundational.** It is a core pillar — the meaning foundation beneath the AI Workforce triad, consumed first by the intelligence engine and (through it) by reports. Its foundational role is advisory and fail-open: load-bearing for *correctness and trust*, never for *availability* (absence yields the byte-identical baseline).
11. **Does it introduce business reasoning? No.** The runtime is deterministic and meaning-blind — it knows *that* "SBD" resolved to an entity, never *why* that entity matters. The model interprets, chooses, and composes; the Foundation offers referents and verifies choices. Reasoning is categorically the engine's, not the Foundation's.
12. **Does it own ontology, vocabulary, canonical naming, bindings, and semantic normalization? Yes — with one boundary.** It owns operational vocabulary, canonical concepts (entities), physical bindings, and semantic normalization (resolution). It owns the *entity* half of ontology; the *relationship* half (a full navigable ontology) is the Knowledge Graph's boundary (§2.6).
13. **What must be protected forever.** §3 — deterministic-before-AI, referents-only, annotate-never-substitute, fail-open/zero-cost, trust-tier provenance, tenant-schema stores, one-store-per-fact/one-writer-per-lifecycle, the AI-never-writes-stored-truth rule, and one-way discovery gates.
14. **What future requests must always be rejected.** §7 — fuzzy/embedding resolution; shipping universal-language dictionaries; letting the model write metadata directly; substituting resolutions into the question string; cross-tenant shared meaning; making the Foundation execute or govern queries; auto-promotion without review (the gap SF1 closes); and turning the Foundation into a document store or a reasoning engine.

**Responsibilities that permanently belong here:** the semantic stores and their lifecycles; deterministic resolution and literal validation; the learning lifecycle and its thresholds; trust-tier provenance; the discovery gates governing domain values.

**Responsibilities that must never live here:** query execution and execution governance; interpretation/reasoning; physical data as a truth of meaning; universal language as shipped data; unstructured documents; the relationship/navigation engine; user authorization; credentials.

## 3. Protected Architecture

Not to be redesigned without overwhelming evidence — these are the constitution made concrete, and the reason this capability is trusted:

1. **Deterministic before probabilistic.** Resolution, candidate retrieval, and value validation are exact-match machinery that runs *before* and *constrains* AI reasoning — so identical questions resolve identically and every resolution is explainable. Verified: `resolveAgainstIndex` uses only the normalization family (lowercase, separator-insensitive, singular/plural, n-grams ≤ 3); no fuzzy, no embeddings.
2. **Referents only — the AI never invents meaning.** Every resolution target is composed from something that already exists (a vocabulary row's SQL equivalent, a persisted domain value, an entity's bound table, a real column). Verified in `buildIndexEntries`. Every path from model output to a store passes a deterministic gate that admits only references to existing things.
3. **Annotate, never substitute.** The user's question string is immutable through the pipeline; resolutions sit *beside* it with provenance. The planner always reads the user's verbatim words. Protected.
4. **Fail-open / zero-cost guarantee.** Any resolver error, zero matches, or absence of semantic metadata yields the empty result and a **byte-identical** baseline pipeline. Verified: `resolve` catches all and returns `ResolvedQuestion.empty`. Semantics may only add signal, never reduce availability — the property that makes this a safe foundation.
5. **Trust-tier provenance on every claim.** `company` > `pack:<id>` > universal (the model, never emitted). Tiers propagate into prompts, traces, and audit. Protected — and SF2 completes it for vocabulary.
6. **Tenant-owned, schema-resident stores; one store per fact, one writer per lifecycle.** No semantic signal crosses tenants; derived structures (the Resolution Index, candidate sets) are recomputed, never persisted as parallel truth. Protected.
7. **The model never writes stored truth.** A model contribution becomes tenant metadata only by surviving validation, accumulating evidence, and crossing learning thresholds — never directly. Protected. (SF1 restores the human gate the constitution requires on the last step.)
8. **One-way discovery gates.** Cardinality- and content-refused values are invisible to every consumer and cannot be resurrected at question time. Protected.
9. **Enforcement strength follows metadata honesty.** Authoritative (`ENUM`) domains gate hard; observed domains advise; the validator fails open on its own errors while its verdicts fail closed on authoritative domains. Protected.

## 4. Stabilization Decisions by Area

| Area | Decision | Reason · Business impact · Risk if ignored |
|---|---|---|
| **Business Language Resolution** (index derivation, matching, precedence, caps) | **KEEP (protected)** | §3.1–3.4. Verified faithful to contract; cost bounded by the `max-resolutions` cap, not tenant size. *Risk if reopened:* losing determinism/explainability — the capability's core value. |
| **Deterministic Literal Resolution + validator** | **KEEP (protected); boundary noted** | The reject-once / hard-block-on-authoritative / advisory-on-observed ladder is correct. The validator is implemented in the `reasoning` package (`LiteralValidator`) as the in-loop enforcement point; **the Foundation owns the rules, the reasoning engine hosts the invocation** — correct placement, recorded so no future reader mistakes it for governance. *Risk if reopened:* re-coupling semantic validation with execution governance. |
| **Learning promotion (auto-promote at thresholds)** | **STABILIZE — the record's headline fix** | Verified: the nightly job promotes directly to `company` vocabulary with no review — the constitution's "most significant contract gap." Decision: a promotion **review gate** — threshold-crossing mappings enter a pending-review state surfaced to stewards; approve → vocabulary, reject → purge (SF1). *Risk if ignored:* machine-learned language silently becomes authoritative company meaning, undermining the steward-owned-meaning claim; workforce volume amplifies it. |
| **Vocabulary tier provenance** | **STABILIZE** | Verified: `OperationalVocabulary` has no creator/tier column; every row resolves `company` (resolver line 186–189). Decision: add the provenance column, stamp it on all writers (steward, pack instantiation, promotion), resolver reads the real tier (SF2). *Risk if ignored:* pack and tenant language are indistinguishable in prompts/traces — the trust ladder is partly unlabelled. |
| **Resolution without domain scope** | **STABILIZE** | Verified: agentless questions (no domain keys) skip resolution entirely (resolve, line 117). Decision: run resolution across all ACTIVE entities/vocabulary when no scope is supplied, bounded by the same caps, mirroring AI Memory's unscoped-retrieval posture (SF3). *Risk if ignored:* the least-configured tenants get the least semantic help, silently — a compounding onboarding cliff. |
| **Semantic observability** | **STABILIZE** | No metrics on resolution hit/miss, literal reject/hard-block, promotion/purge, or fail-open activations. Decision: semantic metrics via the secured actuator (SF4). *Risk if ignored:* a fail-open, self-promoting layer runs blind — degradation and questionable promotions are invisible. |
| **Value domains + discovery gates** | **KEEP (protected)** | Cardinality/content gates and authoritative-vs-observed enforcement are correct and one-way. *Risk if reopened:* re-admitting refused values or weakening the gate. |
| **Semantic roles (confirmed > declared > inferred)** | **KEEP (protected)** | Provenance-ranked roles are the single source of truth for discovery decisions; consumed, never re-decided (verified in `rankLiteralCandidates`). |
| **Learning signal capture (heuristic extraction)** | **KEEP** | Term extraction and correction detection are pattern-based; noise washes out through the confidence lifecycle by design. Extraction-quality tuning is deferred (§6), not a blocker. |
| **Nightly maintenance job (tenant iteration)** | **KEEP; instrument** | Verified: iterates active tenants (incl. public) with per-schema tenant context; correct. SF4 adds outcome metrics; SF1 changes what it *does* on promotion (propose, not promote). |
| **Knowledge-graph hosting (advisory context)** | **KEEP (bounded); relationship engine DEFERRED to its own record** | The graph is advisory context only, participating in neither resolution nor validation — correct and bounded. The Foundation owns entities/bindings; the relationship/navigation engine is a separate concern (§2.6), not this record's to expand. |
| **Value-domain source completeness** (`ENUM` live; `CHECK`/`DOMAIN`/`OBSERVED`/`MANUAL` in model) & **`metric` resolution kind** (reserved) | **KEEP reserved; reconcile docs** | Model-ahead-of-pipeline reserved fields are latent capability, not dead code and not a production blocker. Decision: reconcile the documentation with code (the page states only `ENUM` is live; the metadata arc shipped `OBSERVED` value domains — verify and correct), and mark the rest reserved explicitly. Activation is deferred (§6). |
| **Universal-language absence** | **KEEP (protected)** | Common knowledge deliberately lives in the model, never shipped as dictionaries — the property that makes the architecture model-independent. Not a gap; a contract. |
| **Configuration** | **KEEP** | `nexus.blr.*` / `nexus.dlr.*` caps externalized and sensible; SF-work adds only what it introduces; a configuration-reference entry is story DoD. |

## 5. Production Stabilization Work (accepted)

Four stories owned by this record, full definitions in §9:

| # | Work | Area | Severity |
|---|---|---|---|
| SF1 | Learning promotion **review gate** (propose, don't promote; steward approves) | Governance/Trust | High |
| SF2 | Vocabulary tier provenance (creator/tier column, stamped by all writers) | Provenance/Trust | Medium |
| SF3 | Resolution without a domain scope (agentless coverage) | Coverage | Medium |
| SF4 | Semantic observability (resolution, validation, learning, fail-open metrics) | Observability | Medium |

No Critical work: the Foundation has no security exposure, no write path, and no ungoverned execution — a first among the capabilities reviewed.

### External blocking dependencies

| Dependency | Owning record | Why it relates |
|---|---|---|
| Agent-runtime conformance to the Foundation (resolution, value domains) | **ADR-0003 (Autonomous Agents) A5/A9** | The one consumer that bypasses the Foundation; SF work makes bypass unnecessary but closing it is owned there. |
| Actuator authentication for metrics exposure | **Authentication / Platform Security stabilization** | SF4's metrics ship through the secured actuator. |
| Steward role for the promotion-review surface | **User Management stabilization** | SF1's approval action is gated on a steward/curator role (same dependency as ADR-0005 C4, ADR-0006 M5). |

## 6. Deferred Work (belongs to platform / capability evolution)

| Item | Why deferred |
|---|---|
| **Additional value-domain sources** (`CHECK`, `DOMAIN`, `OBSERVED` pipeline breadth, `MANUAL`) | Latent model capacity; activate when a discovery need arises. `OBSERVED` from the metadata arc should be confirmed live and the docs reconciled (§4), but broadening the rest is evolution, not stabilization. |
| **The `metric` resolution kind** | Reserved in the grammar; emit it when a metrics-semantics need is designed, via a future ADR — not a production gap. |
| **Fuzzy / embedding-assisted resolution** | Explicitly **rejected as a default** (§7); if ever justified it is a *separate, opt-in* advisory tier that never weakens the deterministic core, and needs its own ADR — deferred, not adopted. |
| **Learning-extraction quality** (better term extraction, correction detection) | The confidence lifecycle washes out noise by design; precision tuning is enhancement, gated behind SF1/SF4 evidence. |
| **Relationship/navigation engine (Knowledge Graph promotion from advisory to participant)** | A distinct capability with its own record; the Foundation must not absorb it (§2.6). |
| **Cross-domain / global vocabulary reuse within a tenant** | Domain scoping is deliberate; broader reuse is a product decision after SF3 establishes agentless coverage. |
| **Synonym/alias stores as first-class curated data** | The learning loop is the sanctioned path from usage to curated language; a hand-curated synonym store is an enhancement to evaluate after SF1's gate exists, not a parallel writer. |

## 7. Rejected Recommendations (not to be implemented in stabilization)

| Rejected | Why |
|---|---|
| **Fuzzy/embedding resolution as the matching engine** | Destroys determinism, explainability, and identical-question-identical-answer — the core value. The compensation for exact-match limits is the learning loop, not probabilistic matching. |
| **Shipping universal-language dictionaries** (abbreviation packs, general-English synonyms) | Breaks model-independence and forks meaning into the product; universal knowledge stays in the model, entering tenant metadata only through evidenced, validated use. |
| **Letting the model write semantic stores directly** (auto-create entities/vocabulary from a single answer) | Violates the AI-never-writes-stored-truth protected property; every path to a store passes a deterministic gate and, for curated tiers, human review (SF1). |
| **Substituting resolutions into the question string** | Annotate-never-substitute is protected; the planner must always read the user's verbatim words. |
| **Moving RLS/masking/contracts (execution governance) into the Foundation** | Semantic correctness ≠ execution safety (§2.7); merging them would make the meaning layer a second governance engine and re-couple what the architecture separates. |
| **Absorbing the Knowledge Graph's relationship engine into the Foundation** | Meaning and relationships are distinct concerns (§2.6); the graph stays advisory here and evolves under its own record. |
| **Removing the reserved value-domain sources / `metric` kind as "dead code"** | They are latent capability with live seams, not dead code; the correct disposition is *reserved and documented* (§4), activated by future need, not deleted. |
| **Keeping auto-promotion and amending the constitution instead of adding the gate** | Considered and rejected: the steward-owned-meaning guarantee is the capability's differentiator; the evidence favors implementing the gate the constitution already ratified, not lowering the bar to match the code. |

## 8. SaaS Product Assessment

- **Customer value:** foundational and differentiating — "Zevra understands *your* language, resolves it the same way every time, shows its provenance, and can be corrected by your people" is a concrete enterprise-trust story that generic NL-to-SQL cannot make.
- **Differentiated:** strongly. Deterministic, provenance-tagged, steward-owned meaning that *constrains* the model (rather than hoping the model guesses) is a genuine architectural moat; competitors annotate, few govern meaning deterministically, and none pair it with a fail-open zero-cost guarantee.
- **Understandable by business users:** the steward surfaces (entities, vocabulary, roles) are legible; SF1's review queue makes the learning loop legible too — "here is the language Zevra learned from your team; approve what's right."
- **Would customers trust it today?** Largely yes — there is no safety defect — but a rigorous review flags the auto-promotion gap ("machine-learned terms become authoritative company vocabulary with no approval") and the unlabelled vocabulary provenance. After SF1/SF2: yes, with the trust ladder fully honest.
- **Strengthens the platform:** it is the reason every governed answer is *right in the tenant's terms*, and the reason answers are model-independent — swap the model and the contracts hold. It is the meaning pillar the workforce reasons on top of.
- **Enterprise-ready:** not quite (§1 governance-trust gaps); yes upon §10. Architecturally the closest to ready of the capabilities reviewed — no Critical work, four bounded stories.

## 9. Linear Stories (accepted work only)

**SF1 — Learning promotion review gate**
*Business Objective:* honor the constitution's "thresholds plus human review" — machine-learned language never becomes authoritative company vocabulary without steward approval.
*Technical Scope:* the nightly maintenance stops calling `createTerm` directly; threshold-crossing mappings (use ≥ 10, confidence ≥ 0.8) transition to a **pending-review** state and surface on the Semantic page as a review queue; a steward action approves (→ formal `nexus_operational_vocabulary`, provenance stamped per SF2) or rejects (→ purge); purge thresholds (use ≥ 5, confidence < 0.2) are unchanged; a one-time audit reports what auto-promotion has *already* admitted for retroactive steward review.
*Acceptance Criteria:* a mapping crossing the thresholds appears in the review queue and is **not** in vocabulary until approved (test); approve creates a resolvable vocabulary row; reject purges it; the nightly job promotes nothing without approval; the historical auto-promoted set is listed for review; the capability page's "promotion lacks its review gate" limitation is deleted truthfully.
*Dependencies:* steward role (external). *Priority:* High. *Estimate:* Medium.

**SF2 — Vocabulary tier provenance**
*Business Objective:* make the trust ladder fully labelled — pack, company, and learned-then-approved vocabulary are distinguishable in prompts and traces.
*Technical Scope:* add a creator/tier column to `nexus_operational_vocabulary` (migration); stamp it on every writer — steward creation (`company`), pack instantiation (`pack:<id>`), and SF1 approval (recording the learned origin); `BusinessLanguageResolver` reads the real tier instead of the hardcoded `company` (lines 186–189); tiers flow into the RESOLUTIONS block and trace as they already do for entities.
*Acceptance Criteria:* a pack-instantiated vocabulary term resolves with `[pack:<id>]` in prompt and trace (test); a steward term resolves `[company]`; existing rows migrate to `company`; the resolver no longer hardcodes the tier; the "vocabulary tier provenance is unfinished" limitation is deleted truthfully.
*Dependencies:* coordinates with pack instantiation. *Priority:* Medium. *Estimate:* Small.

**SF3 — Resolution without a domain scope**
*Business Objective:* the least-configured tenants get semantic help too — agentless questions resolve against the tenant's whole active semantic store.
*Technical Scope:* when `domainKeys` is null/empty, `resolve` builds the index from **all** ACTIVE entities and vocabulary (and their bound-table columns/domains) in the tenant schema rather than returning empty, bounded by the same `max-resolutions` cap and cost discipline (mirroring AI Memory's unscoped-retrieval posture); the trace records that unscoped resolution ran; the fail-open guarantee is preserved.
*Acceptance Criteria:* an agentless question resolves known entity/vocabulary terms (test); prompt cost stays within the cap on a vocabulary-rich tenant; a no-metadata tenant still yields the byte-identical baseline; the "resolution requires a domain scope" limitation is updated.
*Dependencies:* none. *Priority:* Medium. *Estimate:* Small.

**SF4 — Semantic observability**
*Business Objective:* make the meaning layer's behavior — including how often it fails open and what it promotes — visible on a dashboard.
*Technical Scope:* metrics via the secured actuator — resolution hit/miss and resolutions-per-question distribution, literal-validation reject/hard-block/advisory counts, learning promotion/purge/pending-review counts, resolver and validator fail-open activations, nightly-job outcomes; a health signal for the nightly maintenance job.
*Acceptance Criteria:* a forced resolver failure (fail-open), a literal hard-block, and a promotion-to-pending are each observable in metrics without log access; nightly-job outcomes are visible; metric names documented.
*Dependencies:* actuator authentication (external). *Priority:* Medium. *Estimate:* Small.

## 10. Exit Criteria — declaring the Semantic Foundation **STABLE**

1. **Promotion gate proven:** a threshold-crossing mapping reaches vocabulary only after steward approval (test); the nightly job promotes nothing autonomously; the historical auto-promoted set has been surfaced for review; the constitution divergence is closed and the page corrected.
2. **Provenance complete:** pack, company, and approved-learned vocabulary resolve with correct tiers in prompt and trace (test); no code path hardcodes the vocabulary tier; existing rows migrated.
3. **Agentless coverage:** an unscoped question resolves against the tenant's active store within the cap; a no-metadata tenant still yields the byte-identical baseline; the trace shows whether resolution was scoped or unscoped.
4. **Determinism preserved:** the matching-family suite proves exact/singular-plural/separator-insensitive/n-gram-≤3 behavior and that nothing fuzzier ever matches; identical questions resolve identically across runs.
5. **Fail-open / zero-cost proven:** a no-metadata tenant, a forced resolver exception, and a zero-match question each produce a **prompt-level** byte-identical baseline (not just an equal answer).
6. **Referents-only proven:** every emitted target exists in its store at resolution time; stale bindings (archived entity, dropped column) resolve to nothing, never to ghosts.
7. **Literal-validation ladder proven:** reject-once with the exact legal list, planner replan, hard-block only on authoritative repeat, advisory-only on observed, validator-error fail-open; user-typed literals never vetoed.
8. **Discovery gates one-way:** cardinality/content-refused values are unreachable from resolution, candidates, and validation.
9. **Observability drills:** resolver fail-open, literal hard-block, and promotion-to-pending are each visible in metrics without log access; nightly-job outcomes visible.
10. **Isolation:** no store, derived index, or learning signal crosses tenants, including under the nightly job's tenant iteration.
11. **Consumer conformance recorded:** Conversational Analytics consumes every contract correctly (reference path); the agent-runtime bypass is dispositioned to ADR-0003 with its consumers named.
12. **Documentation reconciled:** the value-domain source status (ENUM/OBSERVED live vs. reserved), the promotion-gate closure, and the provenance completion are corrected on the capability page; stabilization-checklist items covered by SF1–SF4 are checked with links to tests/evidence; the full backend suite is green with zero removed tests.

## 11. Future Evolution Contract

The Semantic Foundation is the platform's **canonical business-meaning layer** — the single, tenant-owned source of what data means, and the deterministic machinery that translates business language into normalized, governed platform meaning. It is a **foundational pillar** beneath the AI Workforce triad, the meaning counterpart to the Connection Registry's data-access foundation and AI Memory's knowledge foundation; like both, it is advisory and fail-open — load-bearing for correctness and trust, never for availability.

Three standing constraints, violable only by a superseding ADR:

1. **Deterministic-before-AI is inviolable.** Resolution and validation remain exact-match, referents-only, annotate-never-substitute, and fail-open. Probabilistic assistance, if ever added, is a separate opt-in advisory tier that never weakens the deterministic core, never substitutes into the question, and never emits an invented referent. The AI never writes stored truth; curated tiers always pass human review.
2. **It owns meaning, not execution, not relationships, not reasoning.** It must never absorb execution governance (the SQL chain), the relationship/navigation engine (the Knowledge Graph), interpretation/reasoning (the engine), unstructured document knowledge (AI Memory), or become a shipped universal-language dictionary. It translates language to governed meaning and stops there.
3. **Meaning stays tenant-owned and provenance-labelled.** Every semantic influence on an answer is traceable to a store row and a trust tier; no anonymous authority, no cross-tenant meaning, no product-forked vocabulary.

A future **Context Assembly ADR** (also referenced by ADR-0006) will define how the intelligence engine composes context from its governed sources — Conversation History, AI Memory, the Semantic Foundation, the Knowledge Graph, Live Operational Data, Analyst Findings. This record owns the **business-meaning segment** — resolution, validation, and the meaning stores; that future ADR sits above this one and refers down to it rather than superseding it. Evolution — additional value-domain sources, the `metric` kind, richer learning signals, relationship participation — builds **on** these contracts through future ADRs against the deferred items (§6), never by weakening determinism, provenance, or the ownership boundary.

---

## Consequences

**Positive:** the constitution's most significant gap (self-promotion into authoritative vocabulary) is closed with the review gate it already ratified; the trust ladder becomes fully labelled; the least-configured tenants gain semantic coverage; and the meaning layer becomes observable — all around a core that verification confirmed is correct and needs no redesign. The capability's identity as the canonical, deterministic, fail-open meaning foundation is recorded so it cannot erode into a probabilistic matcher, a second governance engine, a relationship engine, or a document store.

**Negative:** SF1 introduces a steward review step in the learning loop — deliberate friction that slows the path from usage to curated vocabulary (the point), and depends on a steward role maturing (shared with ADR-0005/0006); the historical auto-promoted vocabulary must be retroactively reviewed, a one-time steward cost; the agent-runtime bypass remains open until ADR-0003's conformance work lands, so the Foundation's guarantees still do not extend to that runtime in the interim.

## Alternatives considered

- **Declare Stable now** — rejected: auto-promotion into authoritative company vocabulary with no review contradicts the steward-owned-meaning guarantee that is the capability's core claim, and the vocabulary provenance gap leaves the advertised trust ladder partly unlabelled — both are production-relevant for a governed-meaning layer even absent any security defect.
- **Amend the constitution to bless auto-promotion instead of building the gate** — rejected (§7): the differentiator is steward-owned meaning; the evidence favors implementing the ratified gate, not lowering the bar to match the code.
- **Adopt fuzzy/embedding resolution to fix the exact-match coverage limit** — rejected: it would trade the capability's defining determinism for recall; the sanctioned compensation is the learning loop (and SF3's agentless coverage), not probabilistic matching.
- **Treat the agent-runtime bypass as this record's blocker** — rejected: scope discipline; the bypass is the Agents runtime's defect (ADR-0003), and this record holds the contract it must conform to rather than owning the fix.
- **Delete the reserved value-domain sources and `metric` kind as dead code** — rejected: latent capability with live seams; disposition is reserved-and-documented, activated by future need.
