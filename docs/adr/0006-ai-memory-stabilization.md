---
description: ADR-0006 — Capability Stabilization Decision Record for AI Memory (Knowledge Memory), the platform's governed enterprise document knowledge repository for AI: KEEP/STABILIZE/DEFER decisions, the protected advisory-knowledge-substrate model (foundational, advisory, AI never writes), the accepted production work (fail-open retrieval, real async indexing, storage-path safety, relevance floor, lifecycle, observability), and the exit criteria for declaring the capability STABLE.
---

# ADR-0006: AI Memory — Capability Stabilization Decision

| | |
|---|---|
| **Status** | Proposed (becomes Accepted on approval of the story set in §9) |
| **Date** | 2026-07-11 |
| **Deciders** | Zevra platform team |
| **Scope** | AI Memory (Knowledge Memory — tenant document knowledge base + retrieval) only. Conversational Analytics, the Semantic Foundation, the Knowledge Graph, conversation history, and chat attachments are referenced only where a decision requires it. |
| **Inputs** | `capabilities/ai-memory.md` (documentation), ADR-0002–0005, and code verification of the memory path (`MemoryController`, `DocumentMemoryService`, `MemoryRepository`). **Where documentation and implementation differ, the implementation is treated as source of truth** and the divergences are recorded below for the capability page to correct. |
| **Relationship to prior ADRs** | ADR-0002 established Conversational Analytics as the governed intelligence engine, whose single context assembler consumes memory as one advisory input; **ADR-0002 S8 already accepts the fail-open wrap for memory retrieval** at the consumer seam. This record owns the memory capability itself and coordinates its retrieval-side change with ADR-0002 S8. ADR-0003/0004/0005 established the analyst, execution, and data-access layers; AI Memory is none of them — it is a shared foundational knowledge substrate beneath the intelligence engine, the document-knowledge counterpart to the Connection Registry's data-access foundation. |
| **Supersedes** | — |
| **Superseded by** | — |

Once accepted, this capability is not reopened for architecture review unless implementation changes significantly; changes to these decisions supersede this record, never edit it.

---

## 1. Executive Decision

## **STABILIZE BEFORE PRODUCTION.**

This is the **least severe** capability reviewed to date: no unauthenticated surface, no plaintext credentials, no write path to any business system, no ungoverned SQL, and — verified — **the AI never writes to memory**. The architecture is sound and its ownership is narrow and correct. But code verification found one documentation-contradicting defect and three real production risks:

- **Indexing is synchronous, not asynchronous (implementation contradicts documentation).** `DocumentMemoryService.uploadDocument` calls `indexDocument(documentKey)` — a `@Async` method — by **self-invocation within the same bean** (line 125). Spring's async proxy cannot intercept an internal call, so `@Async` is a no-op here: extraction, chunking, and the **sequential per-chunk embedding loop** all run in the upload HTTP request thread. The documentation's "uploads return immediately while indexing proceeds asynchronously" and the method's own "the HTTP response is not blocked" comment are false. A large document holds the request open across hundreds of embedding calls; a client timeout mid-index abandons the run and, because nothing reconciles it, leaves the document stuck `INDEXING` forever. (M2)
- **Retrieval is not fail-open — and it runs on every chat question.** `retrieveContext` embeds the question with no guard (line 229); an embedding-service blip throws and fails the **entire chat run**, not just the memory contribution. This violates the advisory-fail-open posture ADR-0002 protects (§3.8 there) for a layer that is purely advisory. Because retrieval is a fixed step of every question, a transient embedding outage takes down all of chat. (M1, coordinating with ADR-0002 S8.)
- **Upload writes user-controlled strings into filesystem paths.** The on-disk path is `{storagePath}/{domainKey}/{documentKey}/{originalFilename}` where `domainKey` and `originalFilename` are caller-supplied and used verbatim (lines 99, 105); only `documentKey` is generated. A path-hostile filename or domain key can escape the storage root (traversal/overwrite). (M3)
- **No relevance floor.** Retrieval is pure top-K by cosine distance with no similarity threshold (`MemoryRepository.retrieveChunks`/`retrieveAllChunks`): the similarity score is computed but never used as a filter, so the K nearest chunks reach the prompt however weak the match — irrelevant context on out-of-scope questions, and an untrusted-content surface. (M4)

**Not Major Rework:** every defect is a bounded implementation fix around a correct design — the AI role is narrow (embedding only), content is tenant-owned and schema-resident, retrieval only ever sees `INDEXED` documents, and there is exactly one document-memory system with no competing memory engine. The fixes are execution model, failure posture, input hygiene, a retrieval threshold, and lifecycle — not a redesign.

**Position verdict (the question this review was asked):** AI Memory is the platform's **governed enterprise document knowledge repository for AI** — a shared **foundational** knowledge substrate that stores, indexes, manages, and retrieves official organizational documentation so the intelligence engine can ground answers in it. It is the second of Zevra's two truth sources (governed written knowledge; live operational data is the governed-SQL side), exposed to the pipeline as retrieved passages the model is instructed to answer from. Like the Connection Registry (ADR-0005) it is a **shared foundational platform capability the AI Workforce stands on**, differing in kind — the registry is the data-access foundation, AI Memory the document-knowledge foundation — and it is consumed by the Conversational Analytics engine today and potentially by future Analysts. It is emphatically **not** a member of the AI Workforce triad, **not** a reasoning engine, and owns no orchestration or business decision: it supplies context and never reasons, remaining one advisory input among several to ADR-0002's single context assembler. §11 fixes that boundary as a standing constraint.

## 2. Vision Alignment (ownership, not design)

Answers of record to the framing questions this review was asked:

1. **Permanent responsibility.** To be the platform's **governed enterprise document knowledge repository for AI** — the tenant-owned substrate that stores, indexes, manages, and retrieves official organizational documentation so that, on request, it supplies the passages most relevant to a question as **advisory context**, scoped by domain, never as an answer of its own and never as a reasoning step. Its content is enterprise knowledge assets — HR policies, the employee handbook, leave and expense policies, procurement SOPs, vendor operating guides, security policies, compliance documentation, architecture documentation, operational runbooks, training manuals, internal standards — not merely uploaded files.
2. **What belongs in memory.** Official organizational documentation as **enterprise knowledge assets** (the classes above), and their derived chunks, embeddings, and metadata. Unstructured written knowledge that exists only in documents and is meant to ground AI answers. Enterprise customers care not only that knowledge is retrievable but that it is **authoritative** — so AI Memory's charter includes, as a long-term concern, **Knowledge Governance**: the metadata and lifecycle that let the platform ground answers in the *correct version* of enterprise knowledge (versioning, publication state, effective dates, supersession, steward ownership — §11). This is knowledge governance *for AI grounding*, not document collaboration; the distinction is that **AI Memory governs enterprise knowledge for AI retrieval, not enterprise document collaboration** (the excluded set — editing, approvals, comments, workflow — stays out, §11).
3. **What must never belong in memory.** Live operational data (governed SQL owns that); structured business meaning — vocabulary, value domains, entity bindings (the Semantic Foundation owns that); entity relationships (the Knowledge Graph owns that); conversation history (the conversation store owns that); per-conversation chat attachments (ephemeral per-conversation context, explicitly *not* memory); credentials or secrets (the Connection Registry owns those). Memory must never become a write path or a reasoning participant.
4. **Foundation or supporting service?** A **shared foundational platform capability** — specifically a *knowledge* foundation, the document-knowledge counterpart to the Connection Registry's *data-access* foundation (ADR-0005). It sits beneath the AI Workforce triad and is consumed by the intelligence engine today and potentially by future Analysts, but it is emphatically **not** a member of the triad, **not** a reasoning engine, and owns no orchestration or business decision. Its foundational role is advisory: it supplies grounded document knowledge, and nothing else in the platform depends on it to *function* (chat degrades to a memory-less answer when it is absent — or *should*, once M1 lands), which is exactly why it must fail open rather than closed.
5. **Does it duplicate the Knowledge Graph, Conversation History, or the Semantic Foundation?** **No.** Four distinct substrates with distinct shapes: Memory = unstructured document text retrieved by vector similarity; Knowledge Graph = typed entities and relationships; Semantic Foundation = curated business meaning (vocabulary/domains/bindings); Conversation History = dialog turns. They are complementary inputs to context assembly, not competing stores. The boundary is protected (§3) precisely so memory does not drift into absorbing structured meaning.
6. **Do multiple memory systems exist?** **No — there is exactly one document-memory system** (`DocumentMemoryService`), verified in code. "Memory" as a loose word also covers conversation history and chat attachments, but those are separate substrates by deliberate design (attachments are per-conversation; history is dialog state) and are correctly *not* merged into this store. This capability does not repeat the two-reasoning-engines problem ADR-0003 had to disposition.
7. **Does memory participate in reasoning, or only supply context?** **Only supplies context.** Verified: the AI never creates, modifies, promotes, or deletes documents or chunks; content changes only through user upload, metadata edit, and archival. Retrieval feeds the orchestrator's routing decision and answer composition as one advisory signal. Protected (§3.1).
8. **Are its ownership boundaries correct?** **Yes.** The narrow AI role, tenant-owned schema-resident content, no-write-path stance, and advisory-only consumption are all correct and are the protected core. The problems are implementation defects, not ownership errors.

**Responsibilities that permanently belong here:** the knowledge-document model and lifecycle; text extraction, chunking, and embedding at index time; question-time semantic retrieval and its domain scoping; the document/chunk/embedding stores; upload validation and curation surfaces.

**Responsibilities that must never live here:** reasoning or answer authorship (the engine's); structured business meaning (the Foundation's); entity relationships (the Knowledge Graph's); conversation state and attachments (the conversation layer's); any write path to a business system; credential storage (the registry's).

## 3. Protected Architecture

Not to be redesigned without overwhelming evidence:

1. **Memory is an advisory context supplier, never a reasoning participant.** The AI's only role is embedding (chunks at index time, questions at retrieval time); it never writes memory. Retrieval informs routing and composition but decides nothing on its own. Verified true in code and the property that keeps this a knowledge substrate rather than a shadow reasoning engine.
2. **Content is tenant-owned and schema-resident.** Documents, chunks, and `vector(1536)` embeddings live in the tenant's own schema; retrieval runs on the tenant-scoped connection. No cross-tenant memory signal is possible. (File bytes on local disk are the exception the stabilization work bounds — M3/§6.)
3. **Retrieval only ever sees `INDEXED` documents.** Both retrieval queries filter `d.status = 'INDEXED'`, and archival (`status = 'ARCHIVED'`) removes a document from all future retrieval by status filter — nothing downstream can resurrect it. This decoupling of retrieval from ingestion state is sound and protected. *(The asynchronicity of ingestion is broken and fixed by M2; the retrieval-only-sees-INDEXED invariant is separate and correct.)*
4. **A single document-memory system.** One store, one service, one retrieval path — no competing memory engines; attachments and conversation history remain separate substrates by design. Protected against future fragmentation.
5. **Read-only stance.** Memory adds knowledge to answers and grants no write path to any business system. Protected.
6. **Narrow, tenant-driven curation.** Content enters and changes only through authenticated upload, metadata edit, and archival, each attributed to a user. No AI-driven or automatic mutation. Protected.

## 4. Stabilization Decisions by Area

| Area | Decision | Reason · Business impact · Risk if ignored |
|---|---|---|
| **Advisory-context role & narrow AI involvement** | **KEEP (protected)** | §3.1/§3.6. The AI embeds and nothing more; memory reasons about nothing. *Risk if reopened:* turning a clean context supplier into a second reasoning surface. |
| **Tenant-schema content placement** | **KEEP (protected)** | §3.2. Documents/chunks/embeddings isolated by schema and tenant connection. *Risk if reopened:* a cross-tenant knowledge-leak surface where none exists today. |
| **Retrieval-only-sees-INDEXED invariant** | **KEEP (protected)** | §3.3. Correct decoupling of retrieval from ingestion state and archival. *Risk if reopened:* archived or half-indexed content reaching prompts. |
| **Question-time retrieval robustness** | **STABILIZE — the record's most impactful fix** | `retrieveContext` throws on embedding failure and fails the whole chat run; memory is advisory and must fail open (ADR-0002 §3.8). Decision: retrieval returns empty on embedding/search failure, logged and counted; coordinates with ADR-0002 S8's consumer-side wrap (M1). *Risk if ignored:* a transient embedding blip takes down every chat question, not just memory. |
| **Indexing execution model** | **STABILIZE** | `@Async` is a no-op via self-invocation; indexing runs in the upload request thread; crash/timeout mid-index orphans the document at `INDEXING`. Decision: make indexing genuinely asynchronous (separate bean / self-injected proxy) so uploads return immediately, plus startup reconciliation of stuck `INDEXING` documents (M2), and correct the capability page. *Risk if ignored:* upload latency proportional to document size, request-thread exhaustion under concurrent uploads, permanently wedged documents. |
| **Upload path & content handling** | **STABILIZE** | Caller-supplied `domainKey` and original filename are used verbatim as filesystem path segments; validation is by file extension only, not content. Decision (storage-technology-independent principle): **user-controlled values must never influence storage identifiers or storage paths** — storage identity derives only from platform-generated keys; today's filesystem layout is one implementation of that rule, and it holds equally for a future object-storage backend (Blob/S3/GCS). Metadata is validated (`domainKey` against the tenant's real domains; declared type reconciled with detected content) (M3). *Risk if ignored:* path traversal / content overwrite via a hostile filename or domain key — on any backend that lets caller input shape the storage location. |
| **Retrieval relevance** | **STABILIZE** | Pure top-K with no similarity floor; the computed similarity is never used to filter. Decision: a configurable minimum-similarity threshold; below-floor chunks are excluded, and an all-below-floor result yields zero chunks (the orchestrator then routes to a knowledge gap) (M4). *Risk if ignored:* irrelevant passages reach the prompt on out-of-scope questions — quality erosion and an untrusted-content surface. |
| **Indexing lifecycle surface** | **STABILIZE (light)** | `FAILED` is terminal through the API; `indexDocument` already supports re-index (it deletes prior chunks first) but no endpoint invokes it except upload. Decision: a re-index/retry endpoint for `FAILED` and stale documents (M5). Re-embedding after an embedding-model change is deferred (§6). *Risk if ignored:* a transiently failed document requires a full re-upload to recover. |
| **Observability & audit** | **STABILIZE** | No metrics on index outcomes, retrieval latency, embedding failures, or chunk volume; uploads/edits/archivals are attributed on the row but not surfaced to a reviewable audit. Decision: memory metrics via the secured actuator and mutation audit to the governance audit surface (M6). *Risk if ignored:* silent indexing failures and no operator view of the store's health or curation history. |
| **Domain/agent retrieval scoping** | **KEEP — but documented as scope, not permission** | Scoping follows the handling agent's domains; unscoped questions search all tenant documents. This is a *routing scope*, not an access-control boundary, consistent with the platform's coarse-grant posture (ADR-0005 §3.3). Fine-grained per-document access control is deferred (§6), recorded as a known posture, not a silent gap. |
| **Chunking strategy** (word windows) | **KEEP** | Structure-blind word-window chunking with overlap is adequate at this maturity; structure-aware chunking is a quality enhancement (§6), not a stabilization blocker. |
| **Local-disk file storage** | **KEEP with bound; object storage DEFERRED** | Original files on the app host are a known single-node limitation; M3 removes the *path-safety* risk regardless of backend. Object storage is a platform-infrastructure decision (§6), shared with the storage questions ADR-0004/0005 also touch. |
| **Archival data retention** | **KEEP posture; align retention** | Archived documents' files and chunks are retained and excluded by status filter — correct for auditability; unbounded retention aligns to the platform retention split (ADR-0002 S6), not a memory-specific story. |
| **Embedding-model dimension binding** (`vector(1536)`) | **KEEP (accepted constraint)** | The fixed vector dimension binds the tenant to 1536-dim models; migration/re-embed tooling is deferred evolution (§6), not a production blocker at current model choice. |
| **Configuration** | **KEEP** | `nexus.memory.*` keys are externalized and sensible; M4 adds the relevance-floor property; a configuration-reference entry is story DoD. |

## 5. Production Stabilization Work (accepted)

Six stories owned by this record, full definitions in §9:

| # | Work | Area | Severity |
|---|---|---|---|
| M1 | Fail-open retrieval (never throw; empty on failure; coordinates with ADR-0002 S8) | Resilience | High |
| M2 | Genuinely asynchronous indexing + orphan reconciliation | Resilience/Performance | High |
| M3 | Storage-path safety + content validation (user input never shapes storage identity) | Security | High |
| M4 | Retrieval relevance floor | Quality/Safety | Medium |
| M5 | Re-index / retry surface for failed and stale documents | Operational maturity | Medium |
| M6 | Memory observability + mutation audit | Ops/Auditability | Medium |

### External blocking dependencies

| Dependency | Owning record | Why it relates |
|---|---|---|
| Consumer-side fail-open wrap of memory retrieval in the chat pipeline | **ADR-0002 S8 (Conversational Analytics)** | M1 hardens the memory service; S8 wraps the consumption seam. They must land together to guarantee end-to-end fail-open; either alone is insufficient. |
| Actuator authentication for metrics exposure | **Authentication / Platform Security stabilization** | M6's metrics ship through the secured actuator and must not be public. |
| Platform retention split (archived-document cleanup) | **ADR-0002 S6** | Archived files/chunks age out under the platform retention policy, not a memory-specific clock. |

## 6. Deferred Work (belongs to platform / capability evolution)

| Item | Why deferred |
|---|---|
| **Object-storage backend for files** (replace local disk) | A platform-infrastructure decision shared with ADR-0004/0005's storage questions; M3 removes the security risk of local disk regardless, so the backend swap is not a production blocker. |
| **Re-embedding / embedding-model migration tooling** | The `vector(1536)` binding is an accepted constraint at the current model; changing models is an evolution project (re-embed every tenant's chunks), not stabilization. |
| **Structure-aware chunking** (headings, tables, sentence boundaries) | A retrieval-quality enhancement; word-window chunking is adequate now and M4's relevance floor bounds the worst case. |
| **Fine-grained per-document / per-user access control** | Domain scoping is routing scope, not permissions, consistent with the platform's coarse-grant model; document-level ACLs are a product decision requiring the user/role model to mature first (as ADR-0005 C4 also depends on). |
| **Hybrid / keyword-plus-vector retrieval, re-ranking** | Retrieval-quality evolution; the top-K-plus-floor path is sufficient for stabilization. |
| **Memory as a source for non-chat consumers** (agents, workflows reading memory) | No current integration exists; if a future analyst needs document knowledge it consumes memory through the engine's context assembler, never by a private path — a future contract, not stabilization. |
| **Cross-domain relevance / automatic domain inference on upload** | The tenant assigns the domain today; automatic classification is an enhancement, not a gap. |

## 7. Rejected Recommendations (not to be implemented in stabilization)

| Rejected | Why |
|---|---|
| **Merging memory with the Knowledge Graph or Semantic Foundation** | They are distinct substrates (§2.5); merging unstructured document retrieval with structured meaning would build one blurred store and violate the ownership boundary this record protects. |
| **Letting the AI curate memory** (auto-promote chunks to "facts," auto-summarize into the store) | Breaks the no-AI-write-path protected property (§3.1/§3.6); a learned-fact store, if ever justified, is a separate substrate requiring a superseding ADR, not a mutation bolted onto document memory. |
| **Making retrieval a hard dependency of chat** (fail-closed when memory is down) | The opposite of the correct fix; memory is advisory and must fail open (M1). Fail-closed would let a memory outage take down all of chat — the very defect being removed. |
| **Rewriting chunking/retrieval for quality now** (structure-aware chunking, re-ranking) | Quality evolution, not a production blocker; the relevance floor (M4) addresses the concrete safety/quality risk without a retrieval rewrite. |
| **Moving to object storage as part of stabilization** | Infrastructure change; M3 closes the security risk of local disk without it, so the backend swap is deferred, not entangled with the blockers. |
| **Adding a second, per-conversation persistent memory** | Attachments already own per-conversation context; a parallel persistent store would fragment the single-memory property (§3.4). |

## 8. SaaS Product Assessment

- **Customer value:** real and legible — "Zevra answers from your own policies and procedures, and tells you when your knowledge base is missing something" (the knowledge-gap signal) is a concrete, demonstrable value loop that differentiates from a model answering out of general knowledge.
- **Differentiated:** document Q&A is common; **document grounding fused with governed live-data answers in one conversational surface, with a knowledge-gap signal when neither can answer**, is the differentiator — and it depends on memory failing open (M1) so the fused pipeline stays up.
- **Understandable by business users:** the Knowledge Memory page (upload, domain, tags, status) is among the platform's most approachable surfaces.
- **Would customers trust it today?** Mostly — there is no credential or injection catastrophe here — but the path-traversal on upload (M3) and the not-fail-open behavior (M1) would surface in a security and reliability review respectively. After M1/M2/M3: yes.
- **Strengthens the platform:** it is one of the two truth sources the intelligence engine composes; a healthy memory makes every document-grounded answer better, and the knowledge-gap signal is a genuine curation product. The long-term differentiator enterprises will ask for is **Knowledge Governance** — the assurance that answers are grounded in the *authoritative, current* version of a policy or SOP, not a stale or draft one (versioning, publication state, effective dates — §11); that is the trust dimension beyond mere retrieval, and it is knowledge governance for AI grounding, not document collaboration.
- **Enterprise-ready:** not quite (§1); yes upon §10. The mildest gap-to-ready of the capabilities reviewed so far.

## 9. Linear Stories (accepted work only)

**M1 — Fail-open retrieval**
*Business Objective:* a memory or embedding outage degrades to a memory-less answer instead of failing every chat question.
*Technical Scope:* `DocumentMemoryService.retrieveContext` never propagates an exception — an embedding-service or search failure is caught, logged, counted (M6 metric), and returned as an empty result; the chat pipeline treats empty/failed retrieval as the fail-open baseline (coordinating with ADR-0002 S8's consumer-side wrap, which owns the assembler seam); no behavior change when retrieval succeeds.
*Acceptance Criteria:* a forced embedding failure during a question yields a completed answer with zero memory chunks and a recorded fail-open activation (test); healthy retrieval is unchanged; end-to-end fail-open verified with ADR-0002 S8 in place.
*Dependencies:* coordinates with ADR-0002 S8. *Priority:* High. *Estimate:* Small.

**M2 — Genuinely asynchronous indexing + orphan reconciliation**
*Business Objective:* uploads return immediately (as documented), and no document is left permanently `INDEXING`.
*Technical Scope:* fix the `@Async` self-invocation so indexing runs off the request thread (move `indexDocument` to a separate bean or invoke through a self-injected proxy so the async proxy is honored); the upload response returns after persistence, before indexing completes; startup reconciliation marks documents stuck `INDEXING` beyond a threshold as `FAILED` (recoverable via M5); correct the capability page's async claims to match the fixed behavior.
*Acceptance Criteria:* an upload of a large document returns promptly while status progresses `UPLOADED → INDEXING → INDEXED` observed asynchronously (test/metric); a simulated crash mid-index leaves no document stuck `INDEXING` after restart; concurrent uploads do not exhaust request threads; documentation updated.
*Dependencies:* none. *Priority:* High. *Estimate:* Medium.

**M3 — Storage-path safety + content validation**
*Business Objective:* enforce the governing principle that **user-controlled values never influence storage identifiers or storage paths**, so no hostile filename or domain key can determine where content is written, escape the storage root, or overwrite existing content. (Storage-backend-independent; the filesystem is today's implementation.)
*Technical Scope:* storage location and identity derive only from platform-generated keys, never from caller input, regardless of backend (filesystem today; object storage — Blob/S3/GCS — under the deferred backend swap, §6). In the current filesystem implementation: store files under generated keys only (`{storagePath}/{documentKey}/…` with a sanitized or generated leaf name); retain the original filename as metadata, never as a filesystem path segment; validate `domainKey` against the tenant's real domains before it touches a path; reconcile the declared/allowed extension with the detected content type and reject on disagreement; reject path-hostile inputs explicitly.
*Acceptance Criteria:* uploads with traversal-hostile filenames and domain keys are stored safely within the root or rejected (fixture suite); a file whose content contradicts its extension is rejected; existing documents remain retrievable; the original filename still displays in listings.
*Dependencies:* none. *Priority:* High. *Estimate:* Small.

**M4 — Retrieval relevance floor**
*Business Objective:* weakly-relevant chunks stop reaching the prompt; out-of-scope questions surface as knowledge gaps rather than being answered from noise.
*Technical Scope:* a configurable minimum cosine-similarity threshold (`nexus.memory.retrieval-min-similarity`) applied in both retrieval queries (the similarity is already computed — filter on it); below-floor chunks excluded; an all-below-floor result returns zero chunks so the orchestrator routes to a knowledge gap; default tuned against representative content.
*Acceptance Criteria:* a question with no relevant document returns zero chunks and routes to a knowledge gap (test); a strongly-matching question is unaffected; the threshold is configurable and documented.
*Dependencies:* none. *Priority:* Medium. *Estimate:* Small.

**M5 — Re-index / retry surface**
*Business Objective:* a failed or stale document can be recovered without re-uploading.
*Technical Scope:* an authenticated endpoint to re-trigger indexing for a `FAILED` or reconciled document, reusing the existing chunk-replacing `indexDocument` path; guarded to a curation role consistent with the platform role model; status transitions back through `INDEXING`.
*Acceptance Criteria:* re-indexing a `FAILED` document produces an `INDEXED` document with fresh chunks (test); re-indexing replaces prior chunks wholesale; the action is attributable in audit (M6).
*Dependencies:* M2 (async path), M6 (audit). *Priority:* Medium. *Estimate:* Small.

**M6 — Memory observability + mutation audit**
*Business Objective:* indexing failures and curation actions are visible on a dashboard and in the audit trail, not discovered at question time.
*Technical Scope:* metrics — index outcomes by terminal state, indexing latency and chunk counts, embedding-failure counts (index and retrieval), retrieval latency and empty-result rate, per-tenant chunk volume — exposed via the secured actuator; audit every document mutation (upload, metadata edit, archive, re-index) to the governance audit surface with actor and non-content diff (document content never audited).
*Acceptance Criteria:* a `FAILED` index, a retrieval fail-open activation, and an archive are each observable in metrics/audit without log access; audit records carry actor and document key, never document content; metric names documented.
*Dependencies:* actuator authentication (external). *Priority:* Medium. *Estimate:* Small.

## 10. Exit Criteria — declaring AI Memory **STABLE**

1. **Fail-open proven:** a forced embedding/retrieval failure yields a completed, memory-less chat answer with a recorded fail-open activation; verified end-to-end with ADR-0002 S8 in place; no failure path takes down a chat run.
2. **Async proven:** uploads return before indexing completes; status progresses asynchronously to `INDEXED`; a crash-mid-index drill leaves zero documents stuck `INDEXING` after restart; concurrent-upload load does not exhaust request threads.
3. **Path-safety proven:** the traversal fixture suite (hostile filenames, hostile domain keys, extension/content mismatch) is stored-within-root or rejected; no user string reaches a filesystem path unsanitized.
4. **Relevance floor active:** an out-of-scope question returns zero chunks and routes to a knowledge gap; a strong match is unaffected; the threshold is configurable and documented.
5. **Recovery surface:** a `FAILED` document is recoverable via re-index to `INDEXED` without re-upload; chunks are replaced wholesale.
6. **Observability & audit:** forced drills (failed index, retrieval fail-open, archive) are each visible in metrics/audit without log access; no document content appears in metrics, logs, or audit.
7. **Isolation:** cross-tenant tests confirm no document, chunk, or embedding is retrievable from another tenant, including identical titles/domains across tenants and the unscoped all-documents retrieval path.
8. **AI-write-path absent:** a code audit confirms no path lets the AI create, modify, promote, or delete documents or chunks.
9. **Single-memory integrity:** a code audit confirms one document-memory system with no competing persistent memory store; attachments and conversation history remain separate.
10. **Documentation reconciled:** the capability page's async-indexing claims, fail-open limitation, and relevance behavior are corrected to match the stabilized implementation; stabilization-checklist items covered by M1–M6 are checked with links to tests/evidence.
11. **Regression:** the full backend suite is green with zero removed tests; upload → index → retrieve round-trips for every allowed file type.

## 11. Future Evolution Contract

AI Memory is the platform's **governed enterprise document knowledge repository for AI** — the shared, tenant-owned foundational substrate that stores, indexes, manages, and retrieves official organizational documentation and supplies it as grounded context to the intelligence engine, reasoning about nothing. It is a **foundation beneath** the AI Workforce triad (Engine / Analysts / Workflow), consumed by the Conversational Analytics engine today and potentially by future Analysts — never a member of the triad itself, and the document-knowledge counterpart to the Connection Registry's data-access foundation.

**A knowledge foundation, not an operational one.** This distinction is load-bearing and makes AI Memory's role precise without diminishing it. The Connection Registry (ADR-0005) is an *operational* foundation — governed operational data access is impossible without it, so it fails closed. AI Memory is the platform's *knowledge* foundation — it makes answers more grounded and more trustworthy, but the platform keeps operating without it: when it is unavailable, answers degrade gracefully through fail-open behavior (M1) rather than the platform failing. Both are shared foundations the AI Workforce stands on; they differ in **criticality posture** — operational-critical and fail-closed versus grounding-advisory and fail-open — which is exactly why M1 (fail-open retrieval) is this record's most impactful fix.

Two standing constraints, violable only by a superseding ADR:

1. **It must remain an advisory context supplier that the AI never writes.** No auto-curation, no AI-promoted "facts," no reasoning inside the memory service. If a *learned-knowledge* or *structured-memory* substrate is ever justified, it is a new capability with its own ADR — never a mutation bolted onto document memory.
2. **It must remain one document-knowledge repository with a bounded charter.** It must never absorb structured business meaning (the Semantic Foundation), entity relationships (the Knowledge Graph), conversation state or attachments (the conversation layer), or become a second persistent memory store. New consumers (e.g. future Analysts) reach it through the engine's context assembler, never by a private path. Absorbing another substrate, or becoming a second competing memory, is an identity change requiring a superseding ADR.

**Permitted evolution — including Knowledge Governance.** AI Memory may grow into a fuller **governed enterprise document knowledge repository for AI**. Permitted directions include **knowledge governance** and an **authoritative document lifecycle** — document versioning, publication state (Draft / Published / Retired), effective dates, superseded-document tracking, steward ownership — together with **version-aware retrieval** (grounding answers in the currently-authoritative version), **steward-driven curation**, richer metadata, and AI-retrieval optimization (hybrid retrieval, re-ranking, structure-aware indexing). These belong because they make AI answers come from the *correct, authoritative* version of enterprise knowledge — and every one of them must continue to serve a single purpose: **improving AI grounding and retrieval**. Growth that deepens grounding and retrieval belongs here.

**Excluded evolution (a hard boundary).** AI Memory must **never** become a general-purpose document-management or collaboration platform. It must not acquire document editing, approvals, collaboration, comments, workflow management, check-in/check-out, or generic content management — those belong to dedicated document-management products, not Zevra's AI Memory. The line is precise: **AI Memory governs enterprise knowledge for AI retrieval, not enterprise document collaboration** — knowledge-lifecycle metadata that improves grounding is in; human document workflow is out. A requested feature that serves human document workflow rather than AI grounding is out of charter and requires a superseding ADR even to be considered.

A future **Context Assembly ADR** will formally define how the intelligence engine composes context from its governed sources — **Conversation History, AI Memory, the Semantic Foundation, the Knowledge Graph, Live Operational Data, Analyst Findings, and other governed context sources** — into what the model sees on each question. This record owns only the **document-memory portion** of that pipeline; the Context Assembly ADR, when written, sits above this record and refers down to it rather than superseding it. Other future evolution — object storage, structure-aware chunking, embedding-model migration, per-document access control — builds **on** this substrate through future ADRs against the deferred items (§6), never by widening the charter beyond AI grounding.

---

## Consequences

**Positive:** the one behavior that could take down all of chat (not-fail-open retrieval) is corrected and memory becomes a true advisory layer; the documentation's central async claim is made true rather than aspirational, and orphaned indexing is reconciled; the upload path-traversal risk is closed; irrelevant context stops reaching prompts; and the capability's clean ownership (advisory-only, no AI write path, one memory system) is recorded as protected so it cannot erode into a shadow reasoning surface.

**Negative:** M1 and ADR-0002 S8 must land together to guarantee end-to-end fail-open — neither alone suffices, a cross-record coordination cost; making indexing genuinely asynchronous (M2) surfaces the real need for indexing observability that M6 must satisfy; local-disk storage remains (its security risk removed by M3, its single-node limitation deferred); the embedding-model dimension binding remains an accepted constraint until the deferred migration tooling exists.

## Alternatives considered

- **Declare Stable now** — rejected: not-fail-open retrieval on an every-question path and a filesystem path-traversal on upload are production-blocking despite the absence of any credential or injection catastrophe.
- **Treat the async defect as documentation-only (relax the docs to say "synchronous")** — rejected: synchronous indexing blocks the upload request across the sequential embedding loop and orphans documents on timeout; the correct fix is real asynchronicity (M2), not documenting the defect as intended.
- **Own the fail-open fix entirely here (ignore ADR-0002 S8)** — rejected: the assembler seam is ADR-0002's; the robust fix needs both the memory service not throwing (M1) and the consumer treating empty as baseline (S8); splitting by seam and coordinating is consistent with how ADR-0005 C2 related to the per-path governance stories.
- **Redesign retrieval for quality (hybrid search, re-ranking, structure-aware chunking) as part of stabilization** — rejected: quality evolution, not a production blocker; the relevance floor (M4) removes the concrete safety risk without a retrieval rewrite, and the rest is deferred (§6).
