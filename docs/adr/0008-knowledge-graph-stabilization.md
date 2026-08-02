---
description: ADR-0008 — Capability Stabilization Decision Record for the Knowledge Graph, the platform's entity-relationship and JOIN-navigation overlay: KEEP/STABILIZE/DEFER/REJECT decisions, the protected relationships-not-entities / deterministic-discovery / JOIN-guidance / fail-open model, the accepted production work (relationship provenance, edge lifecycle, discovery-SQL hardening, observability, documentation), and the exit criteria for declaring the capability STABLE.
---

# ADR-0008: Knowledge Graph — Capability Stabilization Decision

| | |
|---|---|
| **Status** | Proposed (becomes Accepted on approval of the story set in §9) |
| **Date** | 2026-07-11 |
| **Deciders** | Zevra platform team |
| **Scope** | The Knowledge Graph only — entity relationships (`nexus_entity_relationship`), relationship discovery, graph traversal, JOIN-guidance rendering, and the graph visualization/curation surface. The Semantic Foundation (which owns the *entities* the graph's nodes read), AI Memory, Connection Registry, Conversational Analytics, Autonomous Agents, Workflow Automation, and SQL Governance are referenced only to establish ownership boundaries. |
| **Inputs** | Code verification of the graph path (`graph/KnowledgeGraphService`, `graph/KnowledgeGraphRepository`, `graph/KnowledgeGraphController`, `graph/GraphNode`/`GraphEdge`/`GraphData`, `semantic/RelationshipDiscoveryService`), the consumer path (`chat/ChatService`, `agentrunner/AgentRunner`), and the Semantic Foundation documentation (there is **no dedicated Knowledge Graph capability page** — the first finding). **Implementation is the source of truth; documentation divergences are recorded below.** |
| **Relationship to prior ADRs** | ADR-0007 drew the boundary this record confirms in code: the Semantic Foundation owns entities and bindings (meaning); the Knowledge Graph owns relationships (navigation), consumed as context. ADR-0002 established the intelligence engine that consumes the graph's JOIN guidance; ADR-0003 recorded the agent runtime's non-use of it. This record establishes the graph's permanent constitutional identity. |
| **Supersedes** | — |
| **Superseded by** | — |

Once accepted, this capability is not reopened for architecture review unless implementation changes significantly; changes to these decisions supersede this record, never edit it.

---

## 1. Executive Decision

## **STABILIZE BEFORE PRODUCTION.** *(Architecturally complete; not production-ready; no redesign required.)*

The Knowledge Graph is a **small, coherent, correctly-scoped capability** whose core design is sound and — verified — faithfully implemented: it owns entity **relationships**, discovers them **deterministically**, and renders **JOIN guidance** the intelligence engine uses to compose correct multi-table SQL. There is no security catastrophe, no AI write path (verified), and its context builder fails open. But code verification found that the capability is **materially mis-documented**, carries **no provenance on its most consequential output**, and lacks lifecycle and observability. Because its output directly shapes generated SQL, these are production-relevant:

- **The most consequential finding — heuristic guesses are rendered as authoritative JOINs.** `RelationshipDiscoveryService` discovers relationships in two phases: FK constraints from `information_schema` (definitive) and, when a database has no FKs, a **column-name heuristic** fallback (deterministic pattern rules, e.g. an `X_id` column matching a table named `X`). Both are written to `nexus_entity_relationship` with **identical shape and no source/confidence marker**, and `KnowledgeGraphService.buildGraphContext` renders every edge to the model as `"use these exact JOINs"`. A heuristic mis-join therefore reaches SQL generation labelled as certain as a real foreign key, with nothing letting a steward catch it. This is the Foundation's vocabulary-provenance gap (ADR-0007 SF2), one layer over. (KG1)
- **Documentation contradicts implementation on a load-bearing behavior.** The Semantic Foundation page states the graph context has "table-name references deliberately stripped" and is "advisory context only." Verified false: `buildGraphContext` (lines 105–128) emits `[table: <name>]` for every node and `JOIN: <exact join>` for every edge, and `ChatService.joinTableNames` (line 1191) parses those table names back out to rank schema blocks. The graph is a **JOIN authority consumed by SQL generation**, not table-stripped advisory prose. Recording what the capability *actually is* is a stabilization deliverable. (KG5)
- **No edge lifecycle.** Discovery only *inserts* (idempotent via existence check); nothing removes an edge when its FK is dropped, its heuristic basis changes, or its entity is archived. Stale JOIN guidance can outlive its schema and reach the planner. (KG2)
- **Discovery SQL string-interpolates the schema name** (`.formatted(schemaName)`, lines 100/157) into `information_schema` queries, and **no observability** exists on discovery outcomes or context-build fail-opens. Low-risk (schema names are not end-user input) and consistent-with-platform, but both are hardening gaps. (KG3, KG4)

**Not Major Rework — and not a redesign candidate.** The store model (edges over Foundation entities), the deterministic discovery, the recursive-CTE traversal, and the fail-open rendering are all correct and worth protecting. The work is provenance, lifecycle, hardening, observability, and honest documentation — around an untouched core.

**Position verdict (the question this review was asked):** the Knowledge Graph is the platform's **entity-relationship and JOIN-navigation overlay** — it owns the *relationships between* business entities and the *exact joins* that let the engine navigate them, plus the steward-facing graph visualization. It is **not a standalone foundation**: its nodes are the Semantic Foundation's entities (read, never owned), so it is a **supporting overlay on the meaning foundation**, load-bearing for multi-table correctness but fail-open and advisory in posture. §11 fixes that identity permanently.

## 2. Vision Alignment (ownership, not design)

Permanent architectural positions, answering the twenty questions of record:

1. **Permanent responsibility.** Own entity **relationships** (`nexus_entity_relationship`) and supply **navigation and JOIN guidance** — how business entities connect, and the exact joins the engine needs to compose correct multi-table SQL — plus the graph visualization stewards curate.
2. **Why an independent capability?** Relationships are a distinct concern from meaning: their own store, their own discovery mechanism (FK + heuristic), their own traversal algorithms (neighbors, shortest-path), and their own consumer contract (JOIN guidance in prompts). Folding them into the Foundation would blur *what a concept means* with *how concepts connect*.
3. **Foundational or supporting?** A **supporting overlay**, not a standalone foundation. It has no independent node store — nodes are the Foundation's entities — so it cannot stand alone; it is the navigation layer *over* the meaning foundation. Distinct from the three foundations (data-access ADR-0005, knowledge ADR-0006, meaning ADR-0007).
4. **What does it solve that AI Memory cannot?** Memory retrieves unstructured document prose; it cannot tell the engine *how two business tables join*. The graph supplies structured, deterministic JOIN paths — navigation, not passages.
5. **What does it solve that the Semantic Foundation cannot?** The Foundation resolves a term to an entity/value/column (meaning); it deliberately does **not** model how entities connect or how to join their tables (ADR-0007 §2.6). The graph owns exactly that: relationships and joins.
6. **What belongs permanently.** The entity-relationship store; deterministic relationship discovery (FK constraints + column-name heuristics); graph traversal (neighbors, shortest-path); JOIN-guidance rendering; the steward graph-curation/visualization surface.
7. **What must NEVER belong.** The entity/node store (the Foundation's `nexus_business_entity` — the graph reads, never owns it); business meaning/vocabulary/values (Foundation); query execution and governance (the SQL chain); reasoning (the engine); unstructured documents (Memory); an AI write path; probabilistic relationship *inference presented as fact*.
8. **Relationships, navigation, ontology, lineage, reasoning, analytics, or something else?** **Relationships and navigation** (JOIN guidance + traversal). It owns neither the entity half of ontology (Foundation), nor data lineage (not built; a separate future concern), nor reasoning (engine), nor graph analytics beyond neighbor/shortest-path traversal.
9. **Should it become a graph database? No.** It is a relational overlay (two tables + recursive CTEs) over *entity-scale* data — tens of entities per tenant, not millions of rows. A graph database is unwarranted infrastructure; the recursive-CTE approach is adequate and simpler. Reject migration absent proven entity-graph scale (§7).
10. **Should AI ever write into it? No — and verified it does not.** Discovery is deterministic (FK + heuristics); stewards curate; no model output writes an edge. AI may one day *propose* candidate relationships, but writes must remain deterministic-discovery plus steward confirmation — never an AI write path.
11. **Should workflows consume it?** Not today (they don't). A future workflow needing join guidance consumes it through the engine's context assembler, never by a private path. Deferred (§6).
12. **Should Conversational Analytics consume it? Yes — it does, correctly.** The reference consumer: JOIN guidance in prompts, join-table extraction for schema-block ranking.
13. **Should Autonomous Agents consume it?** In principle yes, through the governed context path — but today `AgentRunner` injects `KnowledgeGraphService` and **never calls it** (dead injection). Disposition: remove the dead injection until real, governed consumption is wired (owned with the agent conformance work, ADR-0003).
14. **Should SQL generation consume it? Yes — that is its primary purpose in practice.** The JOIN guidance directly shapes the multi-table SQL the model writes. Verified, and the reframe at the heart of this record.
15. **Should it participate in Context Assembly? Yes** — as one governed input (the relationship/JOIN segment) to the future Context Assembly ADR's pipeline.
16. **Can anything bypass it?** The agent runtime effectively bypasses it (dead injection). Its guidance is fail-open (absent → empty context), so nothing is *forced* through it — but on the governed path it is load-bearing for multi-table correctness.
17. **Does it introduce business reasoning? No.** Deterministic discovery + deterministic traversal + template rendering. The model consumes its guidance; the graph reasons about nothing.
18. **Deterministic or probabilistic? Deterministic.** FK constraints (definitive) and column-name heuristics (rule-based with confidence scoring, not ML inference); recursive-CTE traversal. No embeddings, no model, no probability.
19. **What future requests must always be rejected.** §7 — an AI write path; graph-database migration without scale evidence; owning entities (duplicating the Foundation); executing or governing SQL; presenting heuristic relationships as authoritative without provenance/steward review; becoming a reasoning or analytics engine.
20. **Boundaries protected forever.** §3 — nodes are the Foundation's entities (read, never owned); deterministic discovery only (no AI writes); fail-open advisory posture; tenant-schema isolation; relationships carry provenance and are steward-overridable; JOIN guidance is *guidance*, and everything still executes through governance.

**Responsibilities that permanently belong here:** the relationship store; deterministic discovery; traversal; JOIN-guidance rendering; the graph-curation surface.

**Responsibilities that must never live here:** the entity store; business meaning; query execution and governance; reasoning; documents; an AI write path; a graph-database engine.

## 3. Protected Architecture

Not to be redesigned without overwhelming evidence:

1. **Relationships, not entities — the graph reads the Foundation's entities and owns only edges.** Verified: `findNodesByDomain` selects from `nexus_business_entity`; the graph's own store is `nexus_entity_relationship`. This is the ADR-0007 boundary made concrete and the reason the graph is an overlay, not a foundation. Protected.
2. **Deterministic discovery.** FK constraints first (definitive), column-name heuristics only as fallback when no FKs exist; idempotent (existence-checked before insert). No model, no probability. Protected — KG1 adds provenance so the two sources are distinguishable, not fuzzier.
3. **JOIN guidance is guidance, not execution.** The graph renders exact joins into the prompt; the model composes SQL; the SQL still passes the governance chain downstream. The graph never executes anything. Protected.
4. **Fail-open rendering.** `buildGraphContext` catches all failures and returns the empty string; a graph failure degrades to a graph-less prompt, never a failed answer. Verified (lines 135–138). Protected.
5. **Tenant-schema isolation.** Nodes and edges are schema-resident; every read runs on the tenant-scoped connection; no relationship crosses tenants. Protected.
6. **Steward authority.** Relationships are steward-curatable and overridable; discovery proposes, humans correct. Protected — KG1 makes heuristic edges reviewable.
7. **No AI write path.** Verified: nothing from a model writes an edge. Protected as a permanent constitutional boundary.

## 4. Stabilization Decisions by Area

| Area | Decision | Reason · Business impact · Risk if ignored |
|---|---|---|
| **Relationship store** (`nexus_entity_relationship`, edges over Foundation entities) | **KEEP (protected)** | §3.1. Correct, minimal, the right ownership boundary. *Risk if reopened:* duplicating the Foundation's entity store or inventing a parallel node store. |
| **Deterministic discovery** (FK + heuristic) | **KEEP (protected); STABILIZE provenance** | §3.2. FK phase is definitive; the heuristic fallback is sound *but indistinguishable from FK edges once stored*. Decision: record relationship provenance (definitive-FK vs heuristic-inferred, with confidence) so JOIN guidance can be weighted and heuristic edges reviewed (KG1). *Risk if ignored:* heuristic mis-joins reach SQL generation as authoritative — silently wrong multi-table answers. |
| **JOIN-guidance rendering** (`buildGraphContext`) | **KEEP; correct the record** | The behavior (emit tables + exact joins, consumed by SQL planning) is correct and load-bearing; the *documentation* describing it as table-stripped/advisory is wrong. Decision: keep the behavior, fix the docs (KG5). *Risk if ignored:* the platform's architecture record misrepresents what reaches the model. |
| **Edge lifecycle** (insert-only; no removal/archival) | **STABILIZE** | No path removes an edge when a FK is dropped, a heuristic basis changes, or an entity is archived. Decision: re-discovery reconciliation that archives/removes edges whose basis no longer holds, and excludes archived-entity edges (KG2). *Risk if ignored:* stale JOIN guidance outlives its schema and misdirects SQL. |
| **Discovery SQL construction** | **STABILIZE (hardening)** | `schemaName` is string-interpolated into `information_schema` queries (lines 100/157). Not end-user input, so low risk, but inconsistent with the platform's parameterization principle (ADR-0004 W1). Decision: parameterize/whitelist the schema identifier (KG3). *Risk if ignored:* a latent injection surface if schema names ever become caller-influenced. |
| **Graph traversal** (recursive-CTE neighbors, shortest-path to depth 8) | **KEEP; bound documented** | Correct for entity-scale graphs (tens of nodes). `getShortestPath` loads all nodes then filters in-memory; `findShortestPath` traverses to depth 8. Acceptable at entity scale; the small-graph assumption is recorded, not a blocker. *Risk if reopened:* premature graph-DB infrastructure (rejected, §7). |
| **`bidirectional` flag vs traversal** | **KEEP; note** | Discovery always writes `bidirectional=false`, yet `findShortestPath` traverses both directions regardless and `findNeighbors` honors the flag for reverse edges — a minor inconsistency with no correctness impact (path-finding is intentionally bidirectional). Documented, not a story. |
| **Observability** | **STABILIZE** | No metrics on discovery outcomes (edges created/skipped, FK vs heuristic), context-build fail-open activations, or traversal latency. Decision: graph metrics via the secured actuator (KG4). *Risk if ignored:* silent discovery drift and invisible fail-opens. |
| **Agent consumption (dead injection)** | **KEEP behavior; remove dead injection (hygiene)** | `AgentRunner` injects `KnowledgeGraphService` but never calls it. Disposition: remove the unused dependency; real governed consumption is owned by the agent conformance work (ADR-0003), not invented here. Hygiene note, not a story. |
| **Graph visualization / steward curation** (`KnowledgeGraphController`) | **KEEP** | The domain/full/neighbor/path endpoints and the context endpoint are a legible steward surface; correct. |
| **Documentation** (no capability page; Foundation page inaccurate) | **STABILIZE** | The capability has a controller, service, repository, its own store, and live prompt influence, but **no capability page**, and the Foundation page misdescribes it. Decision: author the Knowledge Graph capability page and correct the Foundation page (KG5). *Risk if ignored:* a load-bearing capability governed by no accurate record. |
| **Configuration** | **KEEP** | Traversal depth and discovery bounds are code constants; acceptable at this maturity; a configuration-reference entry is story DoD where KG-work introduces a knob. |

## 5. Production Stabilization Work (accepted)

Five stories owned by this record, full definitions in §9:

| # | Work | Area | Severity |
|---|---|---|---|
| KG1 | Relationship provenance (definitive-FK vs heuristic-inferred, confidence, steward review) | Correctness/Trust | Medium |
| KG2 | Edge lifecycle + stale-edge reconciliation | Correctness/Lifecycle | Medium |
| KG3 | Discovery-SQL hardening (parameterize/whitelist schema identifier) | Security | Low |
| KG4 | Knowledge Graph observability | Observability | Medium |
| KG5 | Documentation: capability page + correct the Foundation page | Governance-of-record | Medium |

No Critical or High work: the capability has no security catastrophe, no AI write path, no ungoverned execution, and fails open — the second capability (after the Semantic Foundation) with no Critical/High blocker.

### External blocking dependencies

| Dependency | Owning record | Why it relates |
|---|---|---|
| Governed agent consumption of graph JOIN guidance (and removal of the dead injection) | **ADR-0003 (Autonomous Agents)** | The agent runtime injects but does not use the graph; real consumption is owned there, not invented here. |
| Actuator authentication for metrics exposure | **Authentication / Platform Security stabilization** | KG4's metrics ship through the secured actuator. |
| Steward role for the heuristic-edge review surface | **User Management stabilization** | KG1's review action is gated on a steward/curator role (same dependency as ADR-0005 C4, ADR-0006 M5, ADR-0007 SF1). |

## 6. Deferred Work (belongs to platform / capability evolution)

| Item | Why deferred |
|---|---|
| **Graph-database backend** | Rejected as default (§7); if entity-graph scale is ever proven to exceed relational recursive-CTE traversal, it is a future ADR with evidence — not stabilization. |
| **AI-proposed candidate relationships** (model suggests edges for steward confirmation) | A future enhancement that must preserve the no-AI-write boundary (§3.7): the model may *propose*, deterministic discovery + steward confirmation still *writes*. Requires its own ADR. |
| **Data lineage / provenance-of-data graphs** | A distinct concern (how data *flows*, not how entities *relate*); if built, a separate capability, not an extension of the relationship overlay. |
| **Cross-system relationships** (`cross_system` column exists, always false today) | The store reserves cross-connection relationships; activation is evolution, gated on a real multi-connection join need. |
| **Non-chat consumers** (workflows, agents reading the graph directly) | Consumers arrive through the engine's context assembler, never a private path; a future contract, not stabilization. |
| **Graph analytics** (centrality, clustering, community detection) | Beyond navigation; the graph is a JOIN/navigation overlay, not an analytics engine — a separate capability if ever justified. |
| **Richer traversal** (weighted paths, relationship-type-aware routing) | Enhancement over neighbor/shortest-path; entity-scale graphs do not need it yet. |

## 7. Rejected Recommendations (not to be implemented in stabilization)

| Rejected | Why |
|---|---|
| **Migrate to a graph database (Neo4j, pgRouting, etc.)** | Entity graphs are tens of nodes per tenant; recursive CTEs are adequate and simpler, and a graph DB adds operational surface for no proven benefit. Reject absent scale evidence. |
| **Let the model write relationships into the graph** | Violates the no-AI-write protected boundary (§3.7); relationships are deterministic-discovered and steward-confirmed. A model may propose (deferred, §6), never write. |
| **Move entities into the Knowledge Graph** (own nodes as well as edges) | Duplicates the Semantic Foundation's entity store and dissolves the meaning/navigation boundary ADR-0007 drew; nodes stay the Foundation's, read-only here. |
| **Make the graph execute or validate SQL** (own the joins it renders) | JOIN guidance is guidance; execution and governance belong to the SQL chain. Owning execution would make the overlay a second governance engine. |
| **Present heuristic relationships as authoritative** (the status quo) | Rejected by KG1: heuristic and FK edges must be distinguishable so the model and stewards can weight them; equal-authority rendering is the defect being fixed. |
| **Promote the graph from advisory to a hard dependency of chat** | Fail-open is protected; a graph outage must degrade to a graph-less prompt, never fail the answer. |
| **Build graph analytics into the capability** | Out of charter (§2.8); the graph navigates, it does not analyze. |

## 8. SaaS Product Assessment

- **Customer value:** real but mostly invisible — the graph is why Zevra writes *correct multi-table joins* against the customer's real schema instead of guessing, and the visualization is a legible "here is how your business entities connect" artifact for stewards.
- **Differentiated:** modestly. Auto-discovered, steward-curated, deterministic join guidance that constrains generated SQL is a quiet correctness advantage; it compounds with the governance and meaning stories rather than selling on its own.
- **Understandable by business users:** the graph visualization is among the more intuitive surfaces — entities as nodes, relationships as labelled edges.
- **Would customers trust it today?** Largely — there is no safety defect — but a rigorous review flags that heuristic-guessed joins are presented as certain (KG1) and that stale joins can linger (KG2), both of which can silently produce wrong multi-table answers. After KG1/KG2: yes.
- **Strengthens the platform:** it is the navigation half of "understand the business and query it correctly" — the Foundation says what things mean, the graph says how they connect. Together they make multi-table answers trustworthy.
- **Enterprise-ready:** not quite (§1 correctness/record gaps); yes upon §10. Alongside the Semantic Foundation, the closest-to-ready of the capabilities reviewed — no Critical/High work.

## 9. Linear Stories (accepted work only)

**KG1 — Relationship provenance and heuristic review**
*Business Objective:* the engine and stewards can distinguish a definitive foreign-key join from a heuristic guess, so guessed joins never masquerade as certain.
*Technical Scope:* add a source column (e.g. `discovery_source`: `FK` | `HEURISTIC` | `STEWARD`) and a confidence field to `nexus_entity_relationship` (migration); `RelationshipDiscoveryService` stamps FK vs heuristic; heuristic edges are surfaced for steward confirmation (a review queue, parallel to ADR-0007 SF1) and are marked in `buildGraphContext` JOIN guidance so the model can weight them (definitive vs suggested); steward-created/confirmed edges carry `STEWARD`.
*Acceptance Criteria:* an FK-derived edge and a heuristic edge are distinguishable in the store, the graph context, and the steward UI (test); heuristic edges await confirmation before being rendered as authoritative; existing edges migrate with a best-effort source classification; a mis-join can be traced to its discovery source.
*Dependencies:* steward role (external). *Priority:* Medium. *Estimate:* Medium.

**KG2 — Edge lifecycle and stale-edge reconciliation**
*Business Objective:* JOIN guidance never outlives the schema or entities it describes.
*Technical Scope:* re-discovery reconciles rather than only inserts — edges whose FK no longer exists (or whose heuristic basis no longer holds) are archived/removed; edges referencing an archived entity are excluded from rendering and traversal (the render path already joins to entities, but the rows persist — clean them); a reconciliation pass runs with discovery and is idempotent.
*Acceptance Criteria:* dropping a FK and re-running discovery removes/archives the corresponding edge (test); archiving an entity removes its edges from graph context and traversal; re-discovery is idempotent and reversible where safe; no stale JOIN reaches `buildGraphContext`.
*Dependencies:* KG1 (source marks what may be safely reconciled). *Priority:* Medium. *Estimate:* Small.

**KG3 — Discovery-SQL hardening**
*Business Objective:* the graph's only SQL-constructing path follows the platform's parameterization principle.
*Technical Scope:* replace `String.format`/`.formatted(schemaName)` interpolation in `RelationshipDiscoveryService`'s `information_schema` queries with a parameterized or strictly-whitelisted schema identifier (identifiers cannot bind as parameters, so validate against the connection's known schemas); the existing `quotedList` value-escaping is retained; consistent with ADR-0004 W1 / ADR-0005 C2.
*Acceptance Criteria:* a hostile schema-name fixture cannot alter the introspection query (test); discovery behavior is unchanged for legitimate schemas.
*Dependencies:* none. *Priority:* Low. *Estimate:* Small.

**KG4 — Knowledge Graph observability**
*Business Objective:* discovery drift and fail-opens are visible on a dashboard, not inferred from wrong answers.
*Technical Scope:* metrics via the secured actuator — discovery outcomes (edges created/skipped, FK vs heuristic counts), `buildGraphContext` fail-open activations, traversal (neighbor/shortest-path) latency and depth-cap hits, per-tenant edge volume.
*Acceptance Criteria:* a discovery run, a forced context-build failure (fail-open), and a shortest-path query are each observable in metrics without log access; metric names documented.
*Dependencies:* actuator authentication (external). *Priority:* Medium. *Estimate:* Small.

**KG5 — Documentation: capability page + Foundation correction**
*Business Objective:* the platform's architecture record accurately describes what the graph is and what reaches the model.
*Technical Scope:* author a dedicated Knowledge Graph capability page (responsibility, store, discovery, traversal, JOIN-guidance consumption, fail-open posture, ownership boundary vs the Foundation); correct the Semantic Foundation page's claims that graph context is "table-name-stripped" and "advisory context only" to reflect that it emits table names and exact JOIN guidance consumed by SQL generation; cross-link both.
*Acceptance Criteria:* the capability page exists and matches the implementation (verified against `buildGraphContext`); the Foundation page's inaccurate claims are corrected; the pages cross-reference.
*Dependencies:* KG1 (documents provenance once it exists). *Priority:* Medium. *Estimate:* Small.

## 10. Exit Criteria — declaring the Knowledge Graph **STABLE**

1. **Provenance proven:** FK, heuristic, and steward edges are distinguishable in store, graph context, and UI; heuristic edges are not rendered as authoritative before confirmation (test); existing edges classified.
2. **Lifecycle proven:** a dropped FK and an archived entity each remove the corresponding JOIN guidance from context and traversal after re-discovery (test); reconciliation is idempotent.
3. **Discovery hardened:** a hostile schema-name fixture cannot alter the introspection query; legitimate discovery unchanged.
4. **Fail-open proven:** a forced `buildGraphContext` failure yields a graph-less prompt and a completed answer, with the fail-open recorded in metrics — never a failed run.
5. **Determinism confirmed:** a code audit confirms discovery is FK + rule-based heuristic only, with no model in the write path; traversal is deterministic.
6. **No AI write path:** a code audit confirms nothing from a model writes an edge.
7. **Boundary intact:** a code audit confirms nodes are read from `nexus_business_entity` and the graph owns only `nexus_entity_relationship`; it owns no entity, meaning, execution, or reasoning.
8. **Isolation:** cross-tenant tests confirm no node, edge, or traversal result crosses schemas, including the full-graph and shortest-path paths that read all nodes.
9. **Observability drills:** discovery outcome, context-build fail-open, and a shortest-path query are each visible in metrics without log access.
10. **Consumer conformance:** Conversational Analytics consumes JOIN guidance correctly (reference path); the agent runtime's dead injection is removed or its governed consumption wired (dispositioned to ADR-0003).
11. **Documentation reconciled:** the Knowledge Graph capability page exists and matches code; the Foundation page's table-stripped/advisory claims are corrected; stabilization-checklist items covered by KG1–KG5 are checked with links to tests/evidence.
12. **Regression:** the full backend suite is green with zero removed tests; a discover → render → JOIN-in-generated-SQL round-trip is verified end-to-end.

## 11. Future Evolution Contract

The Knowledge Graph is the platform's **entity-relationship and JOIN-navigation overlay** — it owns the relationships between the Semantic Foundation's entities and the join guidance that lets the intelligence engine navigate them, plus the steward graph visualization. It is a **supporting overlay on the meaning foundation**, not a standalone foundation and not a member of the AI Workforce triad; consumed first by SQL generation in the intelligence engine, fail-open and advisory in posture.

Three standing constraints, violable only by a superseding ADR:

1. **Relationships, never entities; navigation, never meaning.** Nodes are the Foundation's entities, read-only here; the graph owns edges and traversal and nothing else. It must never absorb the entity store, business meaning, query execution/governance, reasoning, documents, or data lineage.
2. **Deterministic discovery, no AI writes.** Relationships come from foreign keys, deterministic heuristics, and steward curation — never from a model. AI may propose (via a future ADR); deterministic discovery and human confirmation always write. Heuristic and definitive relationships remain distinguishable by provenance.
3. **Guidance, not execution; fail-open, not blocking.** JOIN guidance is advice the engine may use; everything still executes through the governance chain, and a graph failure degrades to a graph-less prompt. It never becomes a graph-database engine, an analytics engine, or a hard dependency of chat.

A future **Context Assembly ADR** (referenced by ADR-0006 and ADR-0007) will define how the intelligence engine composes context from its governed sources — Conversation History, AI Memory, the Semantic Foundation, the Knowledge Graph, Live Operational Data, Analyst Findings. This record owns the **relationship/JOIN-navigation segment**; that future ADR sits above this one and refers down to it rather than superseding it. Evolution — AI-proposed edges, cross-system relationships, richer traversal — builds **on** these contracts through future ADRs against the deferred items (§6), never by weakening determinism, the no-AI-write rule, or the relationships-not-entities boundary.

---

## Consequences

**Positive:** the one behavior that can silently produce wrong multi-table answers (heuristic joins rendered as authoritative) gains provenance and steward review; stale joins are reconciled; the capability's only SQL-constructing path is hardened; the graph becomes observable; and — for the first time — the platform's record accurately describes what the graph is and what it puts in front of the model. The capability's identity as a deterministic, fail-open, relationships-not-entities navigation overlay is fixed so it cannot drift into a second entity store, a reasoning engine, a graph database, or an AI-written store.

**Negative:** KG1 introduces a steward review step for heuristic edges — deliberate friction on FK-less databases, and another dependency on the maturing steward role (shared with ADR-0005/0006/0007); classifying existing edges is a one-time best-effort migration that may mislabel some legacy rows until reviewed; the documentation debt (an entire missing capability page plus an inaccurate Foundation section) is real work that must land for the record to be trustworthy.

## Alternatives considered

- **Declare Stable now** — rejected: heuristic joins presented as authoritative and stale joins lingering can both silently produce wrong multi-table answers, and the platform's own architecture record misdescribes what reaches the model — production-relevant despite the absence of any security defect.
- **Fold the Knowledge Graph into the Semantic Foundation** — rejected: relationships are a distinct concern with their own store, discovery, traversal, and consumer contract; ADR-0007 drew this boundary deliberately and code confirms it (nodes read the Foundation's entities, edges are the graph's own). Merging would blur meaning with navigation.
- **Migrate to a graph database** — rejected (§7): entity-scale graphs do not justify the infrastructure; recursive CTEs are adequate.
- **Treat the documentation divergence as a doc-only footnote** — rejected: the divergence is about a load-bearing behavior (table names and exact joins reaching the model and shaping SQL); the record must describe the capability as it is, so the correction is stabilization work, not a footnote.
- **Make the agent runtime's graph consumption this record's blocker** — rejected: the dead injection is the agent runtime's concern (ADR-0003); this record removes the dead dependency and holds the contract, rather than owning the agent's consumption.
