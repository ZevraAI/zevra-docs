# Cross-Document Issues

Meta-document for the Zevra Documentation Verification Project. Not published (repo root, outside `docs/`, per `STYLE_GUIDE.md` §10). Logs conflicts discovered *between* documents (or between a document and inconsistent implementation) found while auditing sections independently. No issue here is resolved during the verification phase — resolution happens in a dedicated reconciliation phase after 100% of sections have been audited.

Companion to `DOCUMENTATION_VERIFICATION_TRACKER.md`.

---

## Issue 1 — Multi-tenancy model: schema-per-tenant vs. shared-schema-with-tenant-column

**Documents involved:**
- `docs/capabilities/executive-brief.md` — states "Both tables live in the platform (`public`) schema, keyed by tenant"
- `docs/architecture-review/15-database-architecture.md` — states "Zevra runs on Postgres with **one schema per tenant**... Everything else... is replicated per tenant"
- `docs/architecture-review/04-authentication-layer.md` — states "**Schema-per-tenant isolation.** Isolation is structural... not a discipline every query must remember to apply"

**Conflicting statements:** The Executive Brief doc describes `nexus_morning_brief` / `nexus_morning_brief_config` as shared-schema tables filtered by an explicit `tenant_schema` column. The two architecture-review documents assert, as a platform-wide architectural guarantee, that tenant isolation is structural via one Postgres schema per tenant, with only a small registry schema shared — implying no tenant-scoped application table should exist in the shared schema with a filter column.

**Implementation evidence:**
- `sei-nexus-ai/src/main/resources/db/migration/V031__morning_brief.sql:5` — `SET search_path = public;` before both `CREATE TABLE` statements.
- `V031__morning_brief.sql:9,22` — both tables carry an explicit `tenant_schema TEXT` column (one `NOT NULL`, one `PRIMARY KEY`), not schema-qualified table names.
- `MorningBriefRepository.java:60,95` — queries use plain unqualified table names filtered by `WHERE tenant_schema = ?` (e.g. `"SELECT * FROM nexus_morning_brief WHERE tenant_schema = ? AND brief_date = ?"`).
- Countering evidence (from the research pass behind `04`/`15`): `TenantContext.java` (ThreadLocal schema holder), `TenantAwareDataSource` (switches `search_path` per connection), `TenantSchemaMigrator.java` (`@Order(0)` `ApplicationRunner`, runs Flyway catch-up per tenant schema) — real code supporting a genuine schema-per-tenant pattern for *most* of the platform's tables.

**Documentation-only or implementation-level:** Cannot yet classify with confidence. Two possibilities, not yet distinguished:
1. **Implementation-level inconsistency** — the platform is mostly schema-per-tenant, but the brief/config tables (and possibly others) were deliberately or inadvertently built as shared-schema-with-filter-column, making the codebase itself architecturally inconsistent.
2. **Documentation-level inconsistency only** — the "one schema per tenant, nothing else" framing in `04`/`15` is an overgeneralization from partial evidence; the actual, intended architecture may always have been a mixed model (schema-per-tenant for most subsystems, shared-schema-with-filter for scheduler-driven/cross-tenant-iterated tables like brief and report), and the architecture-review docs are the ones that need correcting, not the code.

**Recommended future audit:** Enumerate every `CREATE TABLE` across all Flyway migrations and classify each as (a) created under a tenant-specific schema via dynamic DDL/migration-per-schema, or (b) created under `public` with a `tenant_schema`/`tenant_id`-style filter column. Cross-check against every repository class's query pattern (schema-qualified vs. filtered). Produce a definitive statement of the actual multi-tenancy model — likely "mixed: schema-per-tenant for most subsystems, shared-schema-with-filter for scheduler-driven tables" — before touching either the capability docs or the architecture-review docs.

**Status:** Open. Not resolved. Both documents left as-is.

**Supporting evidence added (Autonomous Agents audit, 2026-08-01):** `nexus_zevra_agent` and `nexus_zevra_session` (`V028__zevra_agents.sql`) follow the identical pattern — no `SET search_path`, plain `tenant_schema TEXT NOT NULL` column, `ZevraAgentRepository.java:46,54,87,97` filters `WHERE tenant_schema=?`. Two independent subsystems (brief, agents) now confirmed on the shared-schema-with-filter-column pattern, strengthening the "mixed model, not pure schema-per-tenant" hypothesis over "isolated inconsistency."

**Mechanism found (Conversational Analytics audit, 2026-08-01) — likely resolves the "documentation-only vs. implementation-level" question toward "mixed model by design."** `TenantAwareDataSource.java` (Javadoc, `:1-27`) documents a real, deliberate mechanism: it sets Postgres `search_path` to the current tenant's schema on every connection checkout, transparently, for all `JdbcTemplate`/Flyway calls — described in its own Javadoc as "the central mechanism for schema-per-tenant isolation," falling back to `public` only "when `TenantContext` is not set... making the shared registry tables accessible." `TenantSchemaMigrator.java` (Javadoc `:12-34`) documents a **real historical incident (PRO-10)**: tables that exist only in `public` and were never replicated into a tenant's own schema get silently served from `public` via the `search_path = tenant, public` fallback — invisible until an `ALTER` on such a table broke tenants whose schema had drifted. This strongly suggests: (a) the platform's *default*, intended mechanism genuinely is schema-per-tenant (matching `04`/`15`'s claim), via automatic `search_path` switching that requires no per-query tenant filtering in application code; (b) `nexus_morning_brief`/`nexus_zevra_agent`/`nexus_zevra_session` are *deliberate exceptions* — forced into `public` (via explicit `SET search_path = public;` in their own migrations) specifically because scheduler processes need to iterate/query across all tenants in one pass, which the per-connection schema-switching mechanism doesn't support; (c) `nexus_run` (`V001__init.sql`, no `tenant_schema` column, no forced-public override) is the ambiguous case — it has neither an explicit tenant column NOR proof it was actually replicated into every tenant's own schema by `TenantProvisioningService`/`TenantSchemaMigrator`. Code alone cannot distinguish "correctly schema-per-tenant, hence no column needed" from "a PRO-10-shaped bug where it silently shares one `public.nexus_run` across all tenants" — this requires querying live `information_schema` against a real tenant schema, which was not possible in this environment (no `psql` or local Postgres client available). `docs/capabilities/conversational-analytics.md`'s "all stores are schema-resident per tenant" claim for `nexus_run` is classified **⚠ Partially Verified**, not confirmed or contradicted, pending this direct check.

**Corroborating evidence (Scheduled Reports audit, 2026-08-01) — strengthens the "mixed model by design" reading.** `nexus_scheduled_report` (`V017__scheduled_reports.sql`) has **no `tenant_schema` column at all**, and `ScheduledReportService.java:74-104` explicitly loops every tenant schema with `TenantContext.set(schema)` / `.clear()` per iteration before querying — the genuine per-tenant-schema pattern, architecturally the *opposite* of `nexus_morning_brief`/`nexus_zevra_agent`/`nexus_zevra_session`'s forced-`public`-with-filter-column pattern. This is the first table pair confirmed to actually match the "schema-per-tenant" claim in `04`/`15` at the repository/scheduler level. Three capability docs now each accurately describe a *different* actual tenancy mechanism for their own tables (Executive Brief: forced-public+filter; Autonomous Agents: forced-public+filter; Scheduled Reports: genuine per-tenant-schema iteration) — all three checked out exactly as written, which is strong evidence the real architecture is a deliberate mixed model, not a documentation error in any one of them. `nexus_run` (Conversational Analytics) remains the only unresolved case, since it has neither a filter column nor confirmed proof of per-tenant replication.

**Recommended future audit (superseded by the correction below — kept for the record):** (1) Get direct database access (psql or equivalent) and run `SELECT table_schema, table_name FROM information_schema.tables WHERE table_name = 'nexus_run'` against the live Supabase instance. (2) Enumerate every `CREATE TABLE` across all Flyway migrations and classify each as replicated-per-tenant vs. forced-public-with-filter-column.

---

### CORRECTION (Workflow Automation audit, 2026-08-01) — the mechanism is now understood definitively; this also corrects an error in the Autonomous Agents audit entry above.

**What was found:** `TenantProvisioningService.buildFlyway(...)` (`sei-nexus-ai/src/main/java/com/sei/nexus/tenant/TenantProvisioningService.java:327-329`) configures Flyway with **`.schemas(schemaName)`** — meaning `TenantSchemaMigrator`'s per-tenant catch-up (and the original at-creation provisioning) runs **every migration file, V001 through latest, literally inside each tenant's own Postgres schema**, unless that migration explicitly opts itself out with `SET search_path = public;`.

Grepping all 7 migrations relevant to tables audited so far for the literal string `search_path` found it in **exactly one file**:

```
V031__morning_brief.sql:5:SET search_path = public;
```

`V028__zevra_agents.sql` (agents/sessions), `V024__automations.sql` (workflows/executions), `V017__scheduled_reports.sql`, `V016__proactive_alerts.sql`, `V013__rebuild_document_tables.sql` (memory), and `V001__init.sql` (`nexus_run`) all have **no such override** — meaning every one of these tables genuinely gets replicated into each tenant's own physical schema by Flyway, exactly like the architecture-review docs (`04`, `15`) describe as the platform's default mechanism.

**This means the Autonomous Agents audit entry above was wrong to group `nexus_zevra_agent`/`nexus_zevra_session` with `nexus_morning_brief` as "the identical forced-public pattern."** They share only the *cosmetic* similarity of carrying a `tenant_schema` column and being queried with a `WHERE tenant_schema = ?` filter — but for agents/sessions (and workflows/reports/alerts/memory/run), that filter is **redundant defense-in-depth on top of genuine schema-per-tenant isolation**, not the sole isolation mechanism. `nexus_morning_brief`/`nexus_morning_brief_config` are the *only* tables found across this entire verification project that are actually forced into the single shared `public` schema via an explicit migration override — a deliberate, narrow exception, not a widespread pattern.

**Resolved classification:**
- **The real architecture is schema-per-tenant by default** (Flyway replicates almost every table into every tenant's own schema; `TenantAwareDataSource` transparently routes connections there), **exactly as `docs/architecture-review/04-authentication-layer.md` and `15-database-architecture.md` describe.**
- **`nexus_morning_brief`/`nexus_morning_brief_config` are a single, deliberate, well-motivated exception** (the brief scheduler needs one query to see all due tenant configs at once, which per-tenant schema isolation doesn't support without N connections). `docs/capabilities/executive-brief.md`'s description of these two tables is accurate and was never wrong.
- **The `tenant_schema` column present on several genuinely-per-tenant tables (agents, sessions, automation, and likely others) is redundant** — real but harmless double-scoping, not evidence of shared-schema architecture. The Workflow Automation doc's "doubly scoped" claim (line 94) is accurate as a mechanism description, but its implicit framing as something unusual to that one capability is not — the same double-scoping exists on `ZevraAgentRepository`, `UsageRepository`, `TenantDomainRepository`, `UserProfileRepository`, and `TenantRepository` too (found via repo-wide grep during this pass).
- **`nexus_run` (Conversational Analytics) is very likely also genuinely per-tenant-schema-resident**, matching that document's claim — no forced-public override exists in `V001__init.sql`, and the same Flyway `.schemas(schemaName)` mechanism applies to it as to every other table now resolved. The earlier "Partially Verified, pending live-DB check" classification can be upgraded, though a live `information_schema` check (not possible in this environment) would still be the fully rigorous confirmation.

**Status: RESOLVED at the documentation-conflict level.** This was a **documentation-only inconsistency in my own audit reasoning**, not an implementation-level inconsistency in the codebase, and not an error in any of the six capability/architecture-review documents themselves — every one of `executive-brief.md`, `autonomous-agents.md` (re: its own tenancy claim, which it never actually got wrong — I mischaracterized it while cross-referencing), `conversational-analytics.md`, `scheduled-reports.md`, `alerts.md`, `ai-memory.md`, `workflow-automation.md`, `04-authentication-layer.md`, and `15-database-architecture.md` turns out to describe its own subject matter correctly. No document requires correction from this issue. Recorded here in full rather than deleted, so the reasoning trail — including my own intermediate error — stays auditable per the project's "never silently resolve, always show the evidence" rule.

---

## Issue 2 — Knowledge graph context: "table names stripped, advisory only" vs. actual rendering

**Documents involved:**
- `docs/architecture/semantic-foundation.md` (already audited, marked mostly accurate — this is a miss from that audit, found later via ADR-0008) — states: "Entity *descriptions* contribute business meaning to prompts; table-name references are deliberately stripped from graph context, because the schema section is the single authority for physical names" and describes the graph as "a context supplier, not a decider."
- `docs/adr/0008-knowledge-graph-stabilization.md` — states this is "Verified false" and that the graph is "a **JOIN authority consumed by SQL generation**, not table-stripped advisory prose" (its finding KG5).

**Conflicting statements:** `semantic-foundation.md` says table names are stripped from graph context and the graph is advisory-only. ADR-0008 says the opposite and cites code.

**Implementation evidence (independently confirmed in this pass, not just taken from the ADR):** `sei-nexus-ai/src/main/java/com/sei/nexus/graph/KnowledgeGraphService.java:105-128` — `buildGraphContext` emits `[table: <name>]` for every node (via `tableName(n.primaryObjectKey())`) and `"JOIN: " + e.joinGuidance()` for every edge, under the heading `"--- Relationships (use these exact JOINs) ---"`. `sei-nexus-ai/src/main/java/com/sei/nexus/chat/ChatService.java:1191` (`joinTableNames`) parses those same table names back out of the rendered graph text to rank schema blocks — i.e., the platform's own downstream code treats the graph's table names as real, structured data, not stripped advisory prose.

**Documentation-only or implementation-level:** Documentation-only. The code is internally consistent (`KnowledgeGraphService` renders table names and joins; `ChatService` consumes them as such) — there is no code-level inconsistency. `semantic-foundation.md`'s Knowledge Graph section is simply factually wrong about this one point.

**Recommended future audit:** None needed beyond confirming this in the update phase — `semantic-foundation.md`'s Knowledge Graph section should be corrected to match ADR-0008's finding once the reconciliation phase begins. Also worth noting: this was missed in this project's own earlier Semantic Foundation audit (which focused on BLR/DLR/learning/value-domains and did not deeply check the Knowledge Graph subsection) — a reminder that even a thorough per-document pass can miss a claim outside its main focus area, which is exactly why cross-referencing ADRs against already-audited docs is valuable.

**Status:** Open. Not resolved. Neither document modified.

---

*(Further issues will be appended here as they're found during ongoing verification.)*
