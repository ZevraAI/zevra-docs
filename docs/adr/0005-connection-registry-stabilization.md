---
description: ADR-0005 — Capability Stabilization Decision Record for the Connection Registry: KEEP/STABILIZE/DEFER decisions, the protected single-seam credential-custody model, the accepted production work (secret encryption, seam-level grant enforcement, pooling, lifecycle, rotation, audit), and the exit criteria for declaring the capability STABLE.
---

# ADR-0005: Connection Registry — Capability Stabilization Decision

| | |
|---|---|
| **Status** | Proposed (becomes Accepted on approval of the story set in §9) |
| **Date** | 2026-07-11 |
| **Deciders** | Zevra platform team |
| **Scope** | Connection Registry (governed data-source connections + the shared execution seam `DynamicSqlService`) only. Conversational Analytics, Autonomous Agents, Workflow Automation, the SQL Governance Chain, and Tenant Provisioning are referenced only where a decision requires it. |
| **Inputs** | The capability inventory (Connection Registry entry), ADR-0002/0003/0004, and code verification of the connection path (`NexusConnection`, `ConnectionController`, `ConnectionRepository`, `ConnectionTestService`, `DynamicSqlService`, `DemoConnectionSeeder`, `001_init.sql`). |
| **Relationship to prior ADRs** | ADR-0002, ADR-0003, and ADR-0004 each name **"connection secrets encrypted at rest — owned by Connection Registry stabilization"** as an external blocking dependency they cannot close themselves. This record owns that dependency and the rest of the credential-custody surface beneath all three. Where those records stabilized *how SQL is governed* on each path, this record stabilizes *how the connection and its credentials are custodied and enforced* underneath every path. |
| **Supersedes** | — |
| **Superseded by** | — |

Once accepted, this capability is not reopened for architecture review unless implementation changes significantly; changes to these decisions supersede this record, never edit it.

---

## 1. Executive Decision

## **STABILIZE BEFORE PRODUCTION.**

- **Not Stable:** the registry is the correct architecture operated without the enterprise controls a credential custodian requires. Code verification confirms the inventory's headline finding and sharpens it: **connection secrets are stored and used as plaintext** — `encrypted_secret` is `TEXT`, written verbatim by `ConnectionRepository.save`, read verbatim by `DynamicSqlService` and handed straight to `DriverManager.getConnection` (nine call sites, each annotated `PRODUCTION: decrypt via Vault here`). That single fact is the external blocker ADR-0002/0003/0004 each deferred to this record. Around it sit four more production blockers the code verification surfaced: (a) the registry's own access controls are **not enforced at the shared seam** — `tableIsOnAllowlist` is dead code and `read_only` is stored but read by nothing, so the allow-list is honored only on the conversational path and ignored by the agent and workflow paths; (b) **connection management is unauthenticated by role** — any authenticated tenant user can create, edit, or delete a connection and set an arbitrary `jdbc_url` plus credentials, and **no mutation is audited**; (c) **no connection pooling** — every query opens a fresh physical `DriverManager` connection, so concurrency multiplies TCP+auth handshakes against customer databases; (d) **broken lifecycle** — `archive()` performs a hard `DELETE` (`ConnectionRepository` line 85) instead of using the `ARCHIVED` soft-status the schema already supports (added by migration V011 precisely because `archive()` once set it), so the soft-archive machinery exists but the code abandoned it; and the delete-guard checks only data objects and agents, missing workflows, baselines, and reports.
- **Not Major Rework:** the foundational architecture is right and is the platform's most disciplined shared primitive. **Every external database access in the platform acquires its credentials and its physical connection through this one registry** — verified across all consumers (chat reasoning, agents, workflows, baselines, discovery, onboarding, integration templates); none holds its own credentials or opens its own driver. Connections are tenant-owned data in the tenant schema; secrets are redacted before ever returning to an API caller; the allow-list is the right coarse-grained access primitive. The blockers are custody (encryption), enforcement location (move the grant to the seam), lifecycle, authorization, pooling, and observability — engineering *around* a sound single-seam core, not a rebuild.

**Position verdict (the question this review was asked):** the Connection Registry is the platform's **data-access foundation** — the substrate beneath the AI Workforce triad, not a member of it. ADR-0002 made Conversational Analytics the governed intelligence engine, ADR-0003 made Autonomous Agents the analyst identity layer, ADR-0004 made Workflow Automation the execution surface; **all three stand on this registry for every byte they read from a customer system.** Its permanent job is narrow and total: be the single tenant-owned registry of approved connections and the sole custodian of their credentials, turning a connection grant into an authenticated, bounded, physical connection — and nothing more. It must never absorb query governance (the governance chain owns that), business meaning (the Foundation owns that), or user authorization (auth owns that). §11 fixes that boundary as a standing constraint.

## 2. Vision Alignment (ownership, not design)

Answers of record to the framing questions this review was asked:

1. **Permanent responsibility.** To be the platform's single registry of tenant-approved external data-source connections and the **sole custodian of their credentials** — the one seam that converts a connection grant into an authenticated physical connection, scoped by a tenant-owned allow-list and bounded by timeouts and (after C3) pooling. It is connectivity **and** credential custody **and** the connection-level access grant **and** connection lifecycle. It is a foundation capability, not an AI capability: it introduces no reasoning, no model call, no business logic.
2. **Connectivity only, or more?** More, but bounded. It owns **connectivity, credential custody and lifecycle (rotation), connection validation/health, the coarse connection-level allow-list grant, and connection pooling.** It does **not** own *query* authorization (row-level security, masking, contracts — the governance chain), *data discovery* (Enterprise Map consumes the registry), *user* authorization (Authentication/User Management), or fine-grained per-column grants (governance). It provides a read-only *transaction* as defense in depth (C2); it does not own read-only *statement* validation (the governance safety validator does).
3. **Should every capability access databases exclusively through this registry? Yes — and it already does.** This is the registry's defining and most valuable property, verified in code: there is exactly one place credentials live and one seam that opens connections. This is protected (§3.1) and any future capability that reaches a customer database by any other route is, by definition, mis-scoped.
4. **Are there code paths bypassing it?** For **credentials and connectivity — none.** No capability opens its own `DriverManager` with its own credentials; every path resolves through `ConnectionRepository` and `DynamicSqlService`. But the registry's **access controls are bypassed** because they are enforced in consumer code rather than at the seam: the allow-list check lives in `QueryGovernanceService` (conversational path only), so the agent runtime and the workflow engine execute against granted connections **without** an allow-list check, and the `read_only` flag is enforced nowhere. C2 closes this by moving enforcement to the seam every consumer already shares.
5. **Enterprise SaaS expectations — current posture:**

   | Expectation | Status | Evidence |
   |---|---|---|
   | Credential management | ◐ Partial | Central store, redacted on read, single seam — but plaintext at rest, no role gate on CRUD, no mutation audit. |
   | Secret protection | ✗ Fail | `encrypted_secret` is plaintext `TEXT`; passed verbatim to the driver; the platform's most-cited blocker (C1). |
   | Connection lifecycle | ✗ Fail | `archive()` hard-`DELETE`s; `ARCHIVED` not a legal status; incomplete dependency guard (C5). |
   | Connection validation | ◐ Partial | On-demand `POST /test` works and persists a result; no periodic revalidation, staleness invisible (C7). |
   | Pooling | ✗ Fail | Fresh physical connection per query, by design comment; no pool (C3). |
   | Tenant isolation | ✓ Sound | Registry lives in the per-tenant schema; `search_path` routing; no cross-tenant column to leak. Kept, verified at exit. |
   | Auditing | ✗ Fail | Query execution is audited by the governance chain; **connection create/edit/delete/test/secret-change are not** (C4). |
   | Health monitoring | ✗ Fail | `last_test_*` columns exist but update only on manual test; nothing surfaced to actuator (C7). |
   | Rotation | ✗ Fail | No rotation concept; secret change is an in-place overwrite with no audit or validation (C6). |
   | Least privilege | ◐ Partial | The allow-list exists; but no guidance/enforcement of least-privilege DB users, and the demo seeds the app's own datasource credentials (C8). |

6. **Permanently protected decisions:** §3.
7. **Implementation problems requiring stabilization:** §4/§5 (C1–C8).
8. **Capabilities that must never be added here:** §6 (deferred) and §11 (contract) — connector marketplace, query governance, business/semantic metadata, general-purpose (non-connection) secret storage, per-user data authorization, orchestration/execution logic.

**Responsibilities that permanently belong here:** the connection definition model and lifecycle; credential custody, encryption, and rotation; the single execution seam and its connection acquisition; connection pooling; the connection-level allow-list grant and its enforcement; connection validation and health; connection-mutation audit.

**Responsibilities that must never live here:** query governance — RLS, masking, contracts, cost routing (governance-owned, *invoked above* the seam); business meaning — vocabulary, domains, bindings (Foundation-owned); data discovery and metadata (Enterprise Map–owned); user identity and authorization (Auth-owned); reasoning or orchestration of any kind (engine/agent/workflow-owned); storage of secrets unrelated to connections (out of scope entirely).

## 3. Protected Architecture

Not to be redesigned without overwhelming evidence:

1. **Single-seam credential custody.** Every access to an approved external data source in the platform acquires its credentials and its connection through this registry and its one execution service; no capability holds its own credentials or opens its own driver or client. This is the property that makes encryption, rotation, pooling, and audit *installable in one place* — the entire value of the foundation. Verified true today (for the relational sources currently supported) and the reason stabilization is bounded; the property is what any future source type must also acquire connectivity through.
2. **Connection-as-tenant-data.** A connection is a tenant-owned row (`nexus_connection`) in the tenant schema; there is no connection logic in capability code. The mirror of ADR-0003's agent-as-data and ADR-0004's workflow-as-data properties, one layer down.
3. **The allow-list as the tenant-owned connection-level access grant.** Coarse schema/table scoping owned by the tenant is the correct permission primitive at this layer (fine-grained data governance belongs to the governance chain). The *primitive* is protected; C2 moves its *enforcement* to the shared seam so every consumer inherits it — a relocation, not a redesign.
4. **Secret never returned to callers.** Every API read redacts the secret (`***REDACTED***`); the secret exists only between the store and the driver. Protected and strengthened by C1 (encrypted between store and use) and C4 (never logged).
5. **Schema-per-tenant placement of the registry.** The connection table lives in the per-tenant schema, isolated by `search_path`; there is no shared connection pool or cross-tenant credential surface. Protected; verified at exit.
6. **Shared-primitive discipline.** One connection registry, one execution seam, one place credentials live — no capability-private connection stores, drivers, or pools. This is what ADR-0002/0003/0004 each relied on when they deferred secret encryption here instead of solving it five times.

## 4. Stabilization Decisions by Area

| Area | Decision | Reason · Business impact · Risk if ignored |
|---|---|---|
| **Connection definition model** (`NexusConnection`, tenant-owned, redacted reads) | **KEEP (protected)** | §3.2/§3.4. Correct shape, right placement, secret redacted on read. *Risk if reopened:* destroying the single-custodian property every other capability's stabilization depends on. |
| **Single execution seam** (`DynamicSqlService` credential acquisition) | **KEEP (protected)** | §3.1. Verified: every consumer resolves credentials and opens connections here. This is the foundation's whole value. *Risk if reopened:* re-scattering credentials across capabilities, which is exactly what makes encryption/rotation/pooling tractable. |
| **Secret storage** (`encrypted_secret` plaintext) | **STABILIZE — the record's critical work** | Plaintext at rest, handed verbatim to the driver at nine sites; the external blocker ADR-0002/0003/0004 each named. Decision: envelope-encrypt at rest via a managed backend (Vault/KMS/Secrets Manager), decrypt only at acquisition, never log or return (C1). *Risk if ignored:* one database read or backup of the tenant schema discloses every customer database credential — disqualifying in any enterprise security review. |
| **Connection-level enforcement** (allow-list + `read_only`) | **STABILIZE** | `tableIsOnAllowlist` is dead code; `read_only` is read by nothing; enforcement lives only in the conversational governance path. Decision: enforce the allow-list and open a read-only transaction **at the shared seam** so the agent, workflow, and baseline paths inherit them by construction (C2). Complements ADR-0003 A1/A2 and ADR-0004 W1/W2 — those add the governance chain per path; C2 adds the connection-level backstop no caller can skip. *Risk if ignored:* the tenant's own access grant is honored on one of three execution paths. |
| **Connection pooling** (none; per-query `DriverManager`) | **STABILIZE** | A fresh physical connection per query — every chat step, agent tool call, baseline, and discovery scan pays a full connect+auth. Decision: bounded per-connection pools keyed by `connection_key`, with caps and eviction; the seam's API is unchanged (C3). The design comment's stated reason (multi-tenant credential complexity) is answered by *per-connection* pools, not a shared one. *Risk if ignored:* connect-storm latency and customer-database connection exhaustion under modest concurrency. |
| **Connection management authorization** (CRUD open to any user) | **STABILIZE** | `ConnectionController` has no role gate: any authenticated tenant user can register a connection to an arbitrary host with arbitrary credentials, or overwrite an existing one. Decision: gate connection mutations to a steward/admin role (C4). *Risk if ignored:* a low-privilege user points the platform's query engine at an attacker-controlled or unauthorized host (SSRF/exfiltration), or silently swaps a connection's credentials. |
| **Connection-mutation audit** (none) | **STABILIZE** | The governance chain audits *queries*; credential lifecycle events — create, edit, delete, test, secret change, rotation — are unaudited. Decision: audit every connection mutation with actor, tenant, and before/after non-secret diff (C4). *Risk if ignored:* no forensic record of who granted or altered access to a customer database. |
| **Lifecycle** (`archive()` = hard `DELETE`; unused soft-status) | **STABILIZE** | Destructive delete: `archive()` hard-`DELETE`s instead of setting the `ARCHIVED` status the schema already permits (V010/V011); the dependency guard misses workflows, baselines, reports. Decision: soft-archive using the existing `ARCHIVED` status, complete the dependency check across all referencing capabilities, block destructive delete on referenced connections (C5). *Risk if ignored:* an operator deletes a connection still driving live workflows/baselines, breaking them with no recovery. |
| **Credential rotation** (none) | **STABILIZE** | No rotation concept; secret change is an unvalidated in-place overwrite. ADR-0002 §10.1 makes "rotation executed once successfully" an exit criterion. Decision: a rotation operation that validates the new secret before cutover and audits the event (C6). *Risk if ignored:* no supported path to rotate a leaked credential without risking every dependent capability. |
| **Connection health** (manual test only) | **STABILIZE** | `last_test_*` updates only on manual `POST /test`; staleness and silent credential rot are invisible. Decision: periodic lightweight revalidation with staleness surfaced, plus metrics (C7). *Risk if ignored:* a connection dies quietly and every dependent capability fails at use time instead of surfacing on a dashboard. |
| **Demo connection seeding** (`DemoConnectionSeeder`) | **STABILIZE** | Seeds `local-db` at the platform's **own** control-plane database using the app's `spring.datasource` credentials, plaintext, `read_only=true` (unenforced), unconditionally in every environment. Decision: environment-gate the seeder and use a least-privilege demo credential distinct from the platform datasource user (C8). *Risk if ignored:* every tenant can aim the query engine at Zevra's own metadata database with the application's credentials. |
| **Connection validation service** (`ConnectionTestService`) | **KEEP + reuse** | On-demand test with dialect-aware probes and sanitized errors is correct; C6/C7 reuse it for pre-cutover validation and periodic health. *Risk if reopened:* none — it is sound. |
| **REST_API connection type** | **KEEP (minimal) — deeper auth DEFERRED** | The type exists and tests reachability; real authenticated REST integration (bearer/OAuth) is not built and is a future integration surface (§6), not stabilization work. Secret custody (C1) still applies to whatever REST credential is stored. |
| **Tenancy & data placement** | **KEEP with verification** | Registry in the per-tenant schema, `search_path`-isolated, no cross-tenant column. Sound; isolation becomes an exit criterion (§10). |
| **Fine-grained data governance in the registry** | **KEEP (absent) — deliberately** | RLS, masking, contracts, and cost routing belong to the governance chain invoked *above* the seam; the registry provides connectivity and the coarse grant only. Deliberate absence, not a gap. |
| **Configuration** | **KEEP compiled bounds, document them** | Query timeouts are constants (`DEFAULT_QUERY_TIMEOUT_SECONDS`, etc.); acceptable at this maturity. C3 externalizes only the pool bounds it introduces; a configuration reference entry is story DoD. |

## 5. Production Stabilization Work (accepted)

Eight stories owned by this record, full definitions in §9:

| # | Work | Area | Severity |
|---|---|---|---|
| C1 | Encrypt connection secrets at rest (managed backend; decrypt only at acquisition; never log/return) | Security | Critical |
| C2 | Enforce the connection grant at the shared seam (read-only transaction + allow-list for every consumer) | Security/Governance | Critical |
| C3 | Per-connection pooling with bounds | Resilience/Performance | High |
| C4 | Connection-management authorization + mutation audit | Security/Governance | High |
| C5 | Lifecycle correctness (soft-archive, complete dependency guard, non-destructive delete) | Operational maturity | Medium |
| C6 | Credential rotation (validate-before-cutover, audited) | Security | Medium |
| C7 | Connection health + registry observability | Ops/SaaS readiness | Medium |
| C8 | Demo-connection hardening (env-gate + least-privilege credential) | Security | Medium |

### External blocking dependencies

| Dependency | Owning record | Why it blocks this capability |
|---|---|---|
| A managed secret backend provisioned (Vault / cloud KMS / Secrets Manager) | **Platform Security / Infrastructure** | C1 encrypts against it; the encryption story cannot land without a key-management substrate to hold the data-encryption key. |
| A steward/admin role distinct enough to gate on | **User Management stabilization** | C4 gates connection mutations on it; the inventory notes the role model is coarse. A minimal admin/steward distinction must exist. |
| Actuator authentication for metrics exposure | **Authentication / Platform Security stabilization** | C7's registry metrics ship through the secured actuator and must not be public. |

### Downstream records this unblocks

C1 is the exact external dependency **ADR-0002 §5, ADR-0003 §5, and ADR-0004 §5** each record against this capability. On C1's completion, all three can close their secret-at-rest exit criterion. C2 is the connection-level backstop that complements (does not replace) ADR-0003 A1/A2 and ADR-0004 W1/W2.

## 6. Deferred Work (belongs to platform evolution)

| Item | Why deferred |
|---|---|
| **Connector marketplace / non-JDBC connector zoo** | The registry supports POSTGRES/ORACLE/REST_API deliberately; a catalog of SaaS connectors is a product-expansion decision, not stabilization, and any growth beyond the foundation's identity requires a superseding ADR (§11). |
| **Authenticated REST/OAuth connections** | `REST_API` tests reachability only; real authenticated third-party integration is future integration work, gated on the same secret custody (C1) landing first. |
| **Dynamic / short-lived credentials** (Vault dynamic secrets, per-session DB users) | The correct long-term posture, but it presupposes static encryption + rotation (C1/C6) as the floor; sequencing it before them inverts the dependency. |
| **Automatic scheduled rotation** | C6 delivers the rotation *operation*; policy-driven automatic rotation is an enhancement on top of it, not a production blocker. |
| **Network egress policy / hostname allow-listing for `jdbc_url`** (full SSRF control) | C4's authorization gate bounds *who* can register a connection; a full egress-allowlist of reachable hosts is defense-in-depth deferred to a network-security decision, noted as a residual risk until then. |
| **Read-replica / multi-region connection routing** | No current consumer needs it; it is a scaling feature, not a stabilization gap. |
| **Fine-grained per-column/per-row connection grants** | The governance chain already owns fine-grained data authorization; duplicating it on the allow-list would build the same thing twice. |
| **Cross-tenant shared/global connections** | The schema-per-tenant placement is protected (§3.5); a shared-connection concept would breach isolation and requires a superseding ADR if ever justified. |

## 7. Rejected Recommendations (not to be implemented in stabilization)

| Rejected | Why |
|---|---|
| **A bespoke in-app encryption scheme for secrets** (hand-rolled crypto, key in `application.yml`) | Re-creates the plaintext problem one indirection deeper and invites key-management mistakes; C1 mandates a managed backend with the DEK held outside the tenant database. |
| **A single shared connection pool across tenants/connections** | Breaks credential isolation — the exact reason the current code avoids pooling. C3 uses *per-connection* pools keyed by `connection_key`, preserving §3.5. |
| **Moving RLS/masking/contracts into the registry** | Query governance belongs to the governance chain invoked above the seam; absorbing it would make the foundation a second governance engine and re-couple what ADR-0002 separated. |
| **Turning the registry into a general-purpose secrets manager** | Its charter is *connection* credentials; storing unrelated application secrets expands identity beyond the foundation and requires a superseding ADR at minimum. |
| **Migrating the registry out of tenant schemas into a shared table** | Churn against a protected isolation property (§3.5) with no defect demonstrated; the per-tenant placement is correct. |
| **Continuous aggressive health-polling of every tenant database** | Cost and noise against customer systems; C7 uses lightweight periodic revalidation with staleness surfacing plus the existing on-demand test, not a constant probe. |
| **Deleting `tableIsOnAllowlist` as dead code and relying on the governance chain everywhere** | The governance chain is bypassed by two of three execution paths (ADR-0003/0004); the correct fix is to *wire the seam method in* (C2) so enforcement cannot be skipped, not to remove the one seam-level control. |

## 8. SaaS Product Assessment

- **Customer value:** foundational and mostly invisible — customers do not buy a connection registry, but they reject a platform whose credential handling fails review. Its value is *trust surface*: "your database passwords are encrypted, access is scoped and audited, and one place controls it all."
- **Differentiated:** the registry itself is table stakes; the **single-seam custody** (one place to encrypt, rotate, pool, and audit every data-access credential) is a real operational advantage that compounds with the platform's governance story once C1/C2 make it true.
- **Understandable by business users:** the Connections page is legible (name, type, test result); the missing pieces are operator-facing (rotation, health, audit), delivered by C4/C6/C7.
- **Would customers trust it today?** No, knowingly — plaintext credentials and unauthenticated, unaudited connection management fail any security review at the first question. After C1/C2/C4: yes, with evidence.
- **Strengthens the platform:** structurally — it is the one dependency every other capability's stabilization named and could not close itself; stabilizing it retires that blocker across ADR-0002/0003/0004 at once.
- **Enterprise-ready:** no (§1); yes upon §10.

## 9. Linear Stories (accepted work only)

**C1 — Encrypt connection secrets at rest**
*Business Objective:* eliminate plaintext customer-database credentials — the single external blocker ADR-0002/0003/0004 each record against this capability.
*Technical Scope:* connection secrets are envelope-encrypted at rest via a managed backend (Vault / cloud KMS / Secrets Manager) with the data-encryption key held outside the tenant database; `DynamicSqlService` and `ConnectionTestService` decrypt only at connection acquisition, never earlier; the plaintext secret is never logged, never placed in a trace, and never returned by any API (existing redaction preserved); a one-time migration encrypts existing rows; the nine `PRODUCTION: decrypt via Vault here` sites are reduced to a single decryption seam.
*Acceptance Criteria:* direct inspection of the tenant schema shows zero recoverable plaintext secrets; a query executes end-to-end against a live connection using a decrypted secret; logs and traces under a forced error contain no secret material; the migration encrypts a seeded plaintext row idempotently.
*Dependencies:* managed secret backend (external). *Priority:* Critical. *Estimate:* Medium.

**C2 — Enforce the connection grant at the shared seam**
*Business Objective:* make the tenant's connection-level access grant true on **every** execution path, not just the conversational one.
*Technical Scope:* `DynamicSqlService` opens external connections in a **read-only transaction** as connection-level defense in depth, and enforces the connection allow-list (wiring in the existing `tableIsOnAllowlist`, extended to parse the target schema/table from the approved SQL or accept it from the governed caller) before execution; the agent runtime, workflow engine, and baseline paths inherit both by construction; violations fail the call with the reason surfaced to the caller's trace; the conversational governance path's existing allow-list check is reconciled to the seam (single source of enforcement). This is the connection-level backstop beneath ADR-0003 A1/A2 and ADR-0004 W1/W2, not a replacement for them.
**Ownership boundary (explicit, so no future reader conflates the two layers):** at the seam, Connection Registry owns **only** connection acquisition, credential custody, connection-level allow-list enforcement, and read-only *transaction* enforcement. It does **not** own SQL *statement* validation, row-level security, masking, governance contracts, governance audit, or cost routing — those remain the SQL Governance Chain's responsibility, invoked *above* the seam on each path. C2 is a coarse, unskippable floor beneath governance, never a substitute for it: a query can pass the connection grant and still be rejected by governance, and every executed query still traverses its path's governance chain.
*Acceptance Criteria:* an agent query and a workflow query against a non-allow-listed table are both rejected at the seam (tests); a write statement fails against the read-only transaction on all paths; `read_only` is demonstrably enforced (a connection marked read-only rejects a write even if a caller's own validation is absent); conversational path behavior unchanged.
*Dependencies:* complements ADR-0003 A1, ADR-0004 W1/W2 (independent; lands regardless). *Priority:* Critical. *Estimate:* Medium.

**C3 — Per-connection pooling with bounds**
*Business Objective:* bounded, reused physical connections instead of a fresh connect+auth per query.
*Technical Scope:* a bounded connection pool per `connection_key` (not a shared pool — preserving credential isolation, §3.5), created lazily on first use, with configurable max size, idle eviction, and validation-on-borrow; pool lifecycle tied to connection edit/rotation/archive (invalidate on secret change); the seam's public methods are unchanged; metrics feed C7.
*Acceptance Criteria:* repeated queries on one connection reuse pooled physical connections (test/metric); a configured concurrency test stays within the per-connection cap without exhausting the source database; a rotation or edit invalidates the pool; pool bounds documented and configurable.
*Dependencies:* coordinates with C1 (decryption at acquisition) and C6 (invalidate on rotation). *Priority:* High. *Estimate:* Medium.

**C4 — Connection-management authorization + mutation audit**
*Business Objective:* only privileged stewards may grant or alter database access, and every such event is recorded.
*Technical Scope:* gate all `ConnectionController` mutations (upsert, delete, test, rotate) to a steward/admin role; audit every connection mutation to the governance audit surface with actor, tenant, connection key, action, and a before/after diff of non-secret fields (secret values never audited, only "secret changed: true"); reads remain redacted.
*Acceptance Criteria:* a non-privileged user is denied connection create/edit/delete (test); every mutation appears in governance audit with actor and diff; secret material never appears in an audit record; existing connections remain readable per current rules.
*Dependencies:* steward/admin role (external, User Management). *Priority:* High. *Estimate:* Small.

**C5 — Lifecycle correctness**
*Business Objective:* archiving a connection is safe, reversible where possible, and never silently breaks a dependent capability.
*Technical Scope:* replace the hard `DELETE` with a soft-archive using the `ARCHIVED` status the schema already supports (V010/V011) — the status set is already correct; only `archive()` ignores it by deleting; complete the dependency guard to cover data objects, agents, **workflows, baselines, and scheduled reports** before any destructive action; a referenced connection cannot be hard-deleted; archived connections are excluded from active listings but retained for audit correlation.
*Acceptance Criteria:* archiving a referenced connection is blocked with the full dependent list; `archive()` sets `ARCHIVED` rather than deleting, and archive is non-destructive and reversible per spec; a query for dependents returns workflows and baselines, not only objects and agents.
*Dependencies:* none. *Priority:* Medium. *Estimate:* Small.

**C6 — Credential rotation**
*Business Objective:* a supported, audited path to rotate a connection's credential without breaking dependents — and the exit-criterion rotation ADR-0002 requires.
*Technical Scope:* a rotation operation that accepts a new secret, validates it against the live source (reusing `ConnectionTestService`) **before** cutover, encrypts and stores it (C1), invalidates the connection's pool (C3), and audits the event (C4); on validation failure the existing secret is retained unchanged.
*Acceptance Criteria:* a successful rotation swaps the secret with no failed dependent query across the cutover (test); a rotation with an invalid new secret is rejected and leaves the old secret working; the event is audited without exposing secret material; the rotation procedure is documented and executed once end-to-end.
*Dependencies:* C1 (encryption), C3 (pool invalidation). *Priority:* Medium. *Estimate:* Small.

**C7 — Connection health + registry observability**
*Business Objective:* connection failures and registry pressure are visible on a dashboard, not discovered at query time.
*Technical Scope:* periodic lightweight revalidation of active connections updating `last_test_*` with staleness surfaced in the API and UI; metrics — per-connection pool utilization and wait time, connection-acquisition failures, per-connection query volume, decrypt failures, revalidation outcomes — exposed via the secured actuator; a health indicator reflecting connection reachability.
*Acceptance Criteria:* a forced connection failure, a pool-exhaustion event, and a stale connection are each observable in metrics/health without log access; revalidation updates test status on a schedule; metric names documented.
*Dependencies:* actuator authentication (external). *Priority:* Medium. *Estimate:* Small.

**C8 — Demo-connection hardening**
*Business Objective:* the out-of-the-box demo does not hand every tenant the platform's own database credentials.
*Technical Scope:* environment-gate `DemoConnectionSeeder` so it runs only in demo/dev environments; when seeded, use a least-privilege, read-only credential scoped to the demo schema — distinct from the `spring.datasource` platform user; the seeded secret is stored encrypted (C1) like any other.
*Acceptance Criteria:* production startup seeds no demo connection (test/flag); the demo credential cannot read outside the demo schema; the seeded secret is encrypted at rest; existing demo workflows still run in demo environments.
*Dependencies:* C1 (encryption of the seeded secret). *Priority:* Medium. *Estimate:* Small.

## 10. Exit Criteria — declaring Connection Registry **STABLE**

1. **Secrets encrypted:** direct inspection of the tenant schema finds zero recoverable plaintext connection secrets; a live query executes with a decrypted secret; no secret appears in any log, trace, or API response under forced-error drills.
2. **Rotation proven:** a credential rotation is executed once end-to-end with no failed dependent query across the cutover, validated-before-cutover, and audited — satisfying ADR-0002/0003/0004's shared rotation exit criterion.
3. **Seam enforcement:** with C2 in place, an allow-list violation and a write attempt are both rejected at the seam on the **agent**, **workflow**, and **baseline** paths (not only conversational), proven by a fixture suite; `read_only` is demonstrably enforced.
4. **Single-seam integrity:** a code audit confirms no capability opens an external database connection outside `DynamicSqlService`, and no capability holds its own credentials.
5. **Pooling under load:** a recorded concurrency test reuses pooled physical connections within the per-connection cap and does not exhaust the source database; pool invalidation on rotation/edit verified.
6. **Authorization:** connection mutations are denied to non-privileged users and permitted to stewards (tests); the role gate is active in a deployed environment.
7. **Audit:** every connection create/edit/delete/test/rotation in a sampled window is findable in governance audit with actor, tenant, and non-secret diff; zero secret material present in audit.
8. **Lifecycle:** archiving a referenced connection is blocked with the complete dependent list (objects, agents, workflows, baselines, reports); `archive()` sets the existing `ARCHIVED` status instead of deleting; archive is non-destructive.
9. **Health & observability:** forced drills (connection failure, pool exhaustion, stale connection, decrypt failure) are each visible in metrics/health without log access.
10. **Demo hardening:** production startup seeds no demo connection; the demo credential is least-privilege and cannot read outside the demo schema.
11. **Isolation:** cross-tenant tests confirm no connection or credential is reachable from another tenant, including identical connection keys/names across tenants and `search_path` edge cases.
12. **Checklist closure:** the capability page's stabilization-checklist items covered by C1–C8 are checked with links to tests/evidence; the full backend suite is green with zero removed tests; the inventory's "connection secrets stored unencrypted" status is updated truthfully; ADR-0002/0003/0004's secret-at-rest external dependency is marked closed.

## 11. Future Evolution Contract

The Connection Registry is the platform's **data-access foundation** — the single tenant-owned registry of approved connections and the sole custodian of their credentials, beneath the AI Workforce triad (engine, identity, execution) and every other capability that reads customer data.

Two standing constraints, violable only by a superseding ADR:

1. **It must remain the single credential custodian and the single execution seam.** No capability may hold its own external-connection credentials, open its own driver or client, or maintain its own connection pool — this holds for any external data source the registry comes to support (relational databases today; REST/OAuth APIs, object storage, SaaS and document sources tomorrow), not JDBC alone. New data-access capabilities acquire connectivity here and inherit encryption, pooling, the allow-list grant, and audit by construction. A feature that requires bypassing the seam is mis-scoped.
2. **It must remain a foundation, not a governance or product surface.** It must never absorb query governance (RLS, masking, contracts, cost routing — the governance chain), business meaning (the Foundation), user authorization (Auth), or become a general-purpose secrets manager or a connector marketplace. Growth beyond connectivity + credential custody + the connection-level grant + pooling + lifecycle is an identity change requiring a superseding ADR, not incremental accretion.

Future evolution — dynamic/short-lived credentials, authenticated REST/OAuth connections, automatic rotation policy, network egress control — builds **on** this foundation through future ADRs against the deferred items (§6), never by widening the foundation's charter.

A future platform ADR may formally define the **end-to-end data-access pipeline** — connection acquisition → governance chain → execution → evidence — as a single named contract across capabilities. This record owns only the **connection and credential segment** of that pipeline; it does not define the pipeline as a whole, and such a definition, when written, sits above this ADR and refers down to it rather than superseding it.

---

## Consequences

**Positive:** the platform's single most-cited external blocker (plaintext credentials) is closed at its root and simultaneously retires the secret-at-rest dependency in ADR-0002, ADR-0003, and ADR-0004; the tenant's connection grant becomes enforced on every execution path via the one seam no caller can bypass, completing the "one posture for all data access" the governance ADRs each approached from their own side; credential custody gains encryption, rotation, pooling, authorization, audit, and health in the one place the architecture already concentrated them.

**Negative:** C1 introduces a hard dependency on a managed secret backend that must be provisioned before the story can land; C2's read-only transaction will surface any latent write that a consumer's SQL performed against a "read-only" connection (deliberate, but visible); per-connection pooling (C3) adds memory and lifecycle complexity that C7's metrics must make observable; the demo experience (C8) requires a provisioned least-privilege credential rather than reusing the app's datasource user.

## Alternatives considered

- **Declare Stable now** — rejected: plaintext customer-database credentials with unauthenticated, unaudited connection management is disqualifying, and it is the exact dependency three prior records are already blocked on.
- **Fix only encryption (C1) and defer the rest** — rejected: encryption without seam-level enforcement (C2) still leaves the allow-list and read-only grant bypassed on two of three paths, and without authorization/audit (C4) the credential store remains unaccountable; the blocker set in §1 is jointly disqualifying.
- **Solve secret encryption inside each consuming capability** — rejected: it multiplies the hardest security work across ADR-0002/0003/0004 and abandons the single-custodian property that makes the foundation valuable; the whole point of the seam is to solve it once here.
- **Adopt dynamic/short-lived credentials directly, skipping static encryption** — rejected: dynamic secrets presuppose a rotation and custody floor (C1/C6); sequencing the advanced posture before the foundation inverts the dependency and delays closing the blocker every other record is waiting on.
