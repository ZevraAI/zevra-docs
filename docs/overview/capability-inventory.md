---
description: The master inventory of every capability, platform service, administrative feature, runtime component, and background process implemented in Zevra — the entry point for architecture review and stabilization.
---

# Zevra Capability Inventory

The definitive inventory of everything implemented in Zevra today, discovered from the codebase (30 backend packages, 26 REST controllers, 21 UI routes, 9 background jobs) rather than from prior documentation. Each entry appears once, in its logical domain, with its components, surfaces, dependencies, and documentation status. Findings — incomplete features, dead surfaces, duplications, hidden services — are collected at the end, followed by the [coverage table](#capability-coverage) that drives review and stabilization.

Discovery only: this page records what exists. It recommends nothing.

---

## AI Platform

### Conversational Analytics

**Purpose:** the primary engine — the governed pipeline turning a business question into a validated, explained, auditable answer.
**Components:** `ChatService` (16-stage pipeline), `ChatController`, `ReasoningEngine`/`ReasoningPlanner`/`ReasoningEvaluator`, `LiteralValidator`, `ReasoningEventBus` (SSE), `CrossSourceMerger`, `EvidenceStore`.
**UI:** Chat page (`/chat`), `ReasoningTrace` live renderer, conversation history/pins.
**APIs:** `/chat/ask`, `/chat/conversations*`, `/chat/runs/{key}/feedback`, `/chat/runs` (SSE), `/chat/async/{key}`.
**Tables:** `nexus_run`, run evidence, reasoning sessions/steps, `nexus_conversation_pin`.
**Dependencies:** Semantic Foundation, AI Memory, Enterprise Map, Knowledge Graph, temporal anomaly context, SQL governance chain, connection registry, AI client, usage tracking.
**Consumers:** Scheduled Reports (per question), semantic learning, the chat UI.
**Status:** implemented end to end; the platform's reference governance posture.
**Documentation:** ✅ Complete — [capabilities/conversational-analytics.md](../capabilities/conversational-analytics.md)

### Autonomous Agents (Zevra Agents)

**Purpose:** tenant-defined AI investigators running a bounded ReAct loop over granted connections; the platform's second reasoning engine.
**Components:** `AgentRunner`, `AgentToolRegistry` (4 compiled-in tools), `ZevraAgentService`/`Controller`/`Repository`, `ZevraAgentRouter` (LLM chat dispatcher).
**UI:** Agents page (`/agents`), `AgentChat`, `AgentStepTrace`, `AgentFormModal`.
**APIs:** `/zevra-agents` CRUD, `/zevra-agents/{id}/chat`, `/zevra-agents/{id}/sessions`, `/zevra-agents/sessions/{id}`.
**Tables:** `nexus_zevra_agent`, `nexus_zevra_session`.
**Dependencies:** connection registry, shared SQL execution (ungoverned path), operational vocabulary (direct read), AI client, usage tracking.
**Consumers:** Executive Brief (scheduled runs), chat pipeline (step-1b routing).
**Status:** implemented; outside the SQL governance chain (documented gap).
**Documentation:** ✅ Complete — [capabilities/autonomous-agents.md](../capabilities/autonomous-agents.md)

### AI Memory (Knowledge Memory)

**Purpose:** tenant document knowledge base — upload, chunk, embed, and retrieve business documents to ground conversational answers.
**Components:** `DocumentMemoryService`, `MemoryRepository` (pgvector), `MemoryController`, Tika extraction.
**UI:** Memory page (`/memory`).
**APIs:** `/memory/documents` (list/upload/patch/archive).
**Tables:** `nexus_document`, `nexus_document_chunk` (`vector(1536)`, IVFFlat).
**Dependencies:** Azure OpenAI embeddings, local-disk file storage, business domains.
**Consumers:** conversational pipeline (every question), memory-grounded answers.
**Status:** implemented; local-disk storage; retrieval not fail-open in chat.
**Documentation:** ✅ Complete — [capabilities/ai-memory.md](../capabilities/ai-memory.md)

### Executive Brief (Morning Brief)

**Purpose:** scheduled daily operational briefing synthesized from autonomous-agent investigations.
**Components:** `MorningBriefService`/`Scheduler`/`Controller`/`Repository`.
**UI:** Brief page (`/brief`); schedule configuration via Settings.
**APIs:** `/brief`, `/brief/generate`, `/brief/config`.
**Tables:** `nexus_morning_brief`, `nexus_morning_brief_config`.
**Dependencies:** Zevra Agent runtime, AI client (synthesis), tenant registry.
**Consumers:** none (UI only).
**Status:** implemented; `email_to` stored but never delivered; failed days don't retry.
**Documentation:** ✅ Complete — [capabilities/executive-brief.md](../capabilities/executive-brief.md)

### Semantic Foundation (BLR, DLR, Semantic Learning)

**Purpose:** the business-understanding layer — tenant-owned meaning stores plus deterministic resolution, literal validation, and governed language learning.
**Components:** `BusinessLanguageResolver`, `ResolvedQuestion`, `LiteralValidator`, `SemanticService`, `SemanticRepository`, `SemanticLearningService`, `LearningContextBuilder`, `TermExtractor`, `CorrectionDetector`, `EntityCandidateService`, `RelationshipDiscoveryService`.
**UI:** Semantic page (`/semantic`) — entities, vocabulary, candidates stewardship.
**APIs:** `/semantic/*`.
**Tables:** `nexus_business_entity`, `nexus_operational_vocabulary`, learned-mapping and correction stores, `nexus_value_domain` (shared with Enterprise Map).
**Dependencies:** Enterprise Map (bindings), discovery pipeline (producers).
**Consumers:** conversational pipeline (reference consumer), agents (partial, divergent), reports (inherited).
**Status:** implemented; auto-promotion lacks its review gate; vocabulary tier provenance unfinished.
**Documentation:** ✅ Complete — [architecture/semantic-foundation.md](../architecture/semantic-foundation.md)

### Knowledge Graph

**Purpose:** business entities and relationships as navigable structure; keyword-filtered context supplier to planner prompts.
**Components:** `KnowledgeGraphService`/`Repository`/`Controller`, `GraphNode`/`GraphEdge`, `RelationshipDiscoveryService` (producer).
**UI:** Knowledge Graph page (`/graph`) — visual graph.
**APIs:** `/knowledge-graph`.
**Tables:** graph node/edge stores (tenant schema).
**Dependencies:** semantic entities, relationship discovery.
**Consumers:** conversational context assembly (advisory only).
**Status:** implemented; advisory context only — participates in neither resolution nor validation.
**Documentation:** ◐ Partial — covered inside [semantic-foundation.md](../architecture/semantic-foundation.md); no dedicated page. Suggested: `capabilities/knowledge-graph.md` or fold permanently.

---

## Automation & Proactive Surfaces

### Workflow Automation

**Purpose:** visual DAG workflow builder and execution engine with webhook/manual triggers and AI generation.
**Components:** `AutomationService`, `WorkflowExecutionEngine`, 7 `StepExecutor`s, `VariableResolver`, `AutomationGeneratorService`, `DemoConnectionSeeder`.
**UI:** Automations (`/automations`), `AutomationEditor` (React Flow), `GenerateModal`.
**APIs:** `/automations` CRUD, `/automations/{id}/run`, `/automations/run/{slug}` (public webhook), `/automations/generate*`.
**Tables:** `nexus_automation_workflow`, `nexus_automation_execution`.
**Dependencies:** connection registry, shared SQL execution (ungoverned), AI client.
**Consumers:** external systems (webhook).
**Status:** implemented; outside the governance chain; SQL-injection surface on the public webhook (documented).
**Documentation:** ✅ Complete — [capabilities/workflow-automation.md](../capabilities/workflow-automation.md)

### Scheduled Reports

**Purpose:** recurring natural-language question digests delivered by email/Slack, executed through the conversational pipeline.
**Components:** `ScheduledReportService` (own 60s scheduler), `ReportScheduleHelper`, `ReportHtmlComposer`.
**UI:** Reports page (`/reports`).
**APIs:** `/reports` CRUD, `/reports/{key}/run`.
**Tables:** `nexus_scheduled_report`.
**Dependencies:** conversational pipeline (the engine), SMTP, Slack webhooks.
**Consumers:** email/Slack recipients.
**Status:** implemented; no run history or artifacts; standing creator impersonation.
**Documentation:** ✅ Complete — [capabilities/scheduled-reports.md](../capabilities/scheduled-reports.md)

### Alerts (Proactive Intelligence)

**Purpose:** rule-governed notifications over baseline anomaly detection — in-app bell, Slack, email.
**Components:** `AlertService`, `AlertComposerService` (fail-open AI), `NotificationDeliveryService` (the platform's only email/Slack delivery service), rule/delivery repositories.
**UI:** Alert Rules tab (Temporal page), `NotificationPanel` bell.
**APIs:** `/alert-rules` CRUD + test, `/alerts` history/unread/read.
**Tables:** `nexus_alert_rule`, `nexus_alert_delivery`.
**Dependencies:** Temporal Intelligence (sole trigger), SMTP, AI client.
**Consumers:** the notification bell; humans on external channels.
**Status:** implemented; outbound outcomes unpersisted; test alerts start real cooldowns.
**Documentation:** ✅ Complete — [capabilities/alerts.md](../capabilities/alerts.md)

### Temporal Intelligence (Baselines & Anomalies)

**Purpose:** learn each metric's normal range from steward-defined measurement SQL and detect z-score anomalies; supply anomaly context to chat prompts.
**Components:** `BaselineService` (hourly multi-tenant scheduler), `AnomalyDetector`, `TemporalRepository`, `TemporalController`.
**UI:** Temporal page (`/temporal`) — baselines, anomalies, alert rules.
**APIs:** `/temporal/baselines` (+ `/refresh`), `/temporal/anomalies` (+ PATCH status/finding link; single-anomaly GET deliberately returns 501).
**Tables:** operational-baseline and anomaly-event stores.
**Dependencies:** SQL safety validation (creation-time), shared SQL execution, connection registry.
**Consumers:** Alerts (the delivery half), conversational context (anomaly block), operational findings.
**Status:** implemented; naive mean/stddev statistics; manual refresh bypasses alerting.
**Documentation:** ◐ Partial — measurement half covered inside [alerts.md](../capabilities/alerts.md). Suggested: `capabilities/temporal-intelligence.md`.

---

## Data Platform & Semantics

### Enterprise Map (Metadata Registry)

**Purpose:** the discovered physical-schema registry — data objects, columns, semantic roles, value domains, operational notes — that semantics bind to and context assembly renders from.
**Components:** `EnterpriseMapService`/`Repository`/`Controller`, `DataObject`, `DataColumn` (semantic role + role source), `ValueDomain`, `OperationalNote`, `SampleContentClassifier` (content-safety gate).
**UI:** Enterprise Map page (`/enterprise`).
**APIs:** `/enterprise-map/*`.
**Tables:** data-object/column stores, `nexus_value_domain`, operational notes.
**Dependencies:** connection registry, onboarding/discovery scans.
**Consumers:** context assembly, BLR index derivation, discovery decisions (roles).
**Status:** implemented; only `ENUM` value-domain source live.
**Documentation:** ◐ Partial — role/domain aspects in [semantic-foundation.md](../architecture/semantic-foundation.md). Suggested: `platform/enterprise-map.md`.

### Connection Registry

**Purpose:** tenant-approved database connections — the single data-access registry every execution path shares.
**Components:** `ConnectionController`/`Repository`, `NexusConnection`, `DynamicSqlService` + `SqlSafetyService` (shared execution/validation, in `sql` package).
**UI:** Connections page (`/connections`).
**APIs:** `/connections`.
**Tables:** `nexus_connection`.
**Dependencies:** none (foundation).
**Consumers:** chat reasoning, agents, workflows, baselines, schema discovery.
**Status:** implemented; **connection secrets are stored unencrypted** (code comment: "PRODUCTION: decrypt via Vault here").
**Documentation:** ✗ Not documented. Suggested: `platform/connections.md`.

### Business Domains

**Purpose:** the tenant's business-domain registry — the scoping unit for agents, memory, semantics, and baselines.
**Components:** `DomainController`/`Repository`, `Domain`.
**UI:** Domains page (`/domains`).
**APIs:** `/domains`.
**Tables:** `nexus_domain`.
**Consumers:** nearly everything (scoping).
**Status:** implemented.
**Documentation:** ✗ Not documented. Suggested: `platform/domains.md`.

### Chat Routing Agents (NexusAgent)

**Purpose:** the chat pipeline's domain-scope holders — routed by explicit key, single-active, or LLM routing; carry playbooks, KPIs, and versions.
**Components:** `AgentController`/`Service`/`Repository`, `NexusAgent`, `AgentPlaybook` (injected into reasoning context), `AgentKpi`, `AgentVersion`.
**UI:** none dedicated (managed via API; referenced in chat).
**APIs:** `/agents`.
**Tables:** agent/playbook/KPI/version stores.
**Consumers:** conversational pipeline (routing + domain scope + playbooks).
**Status:** implemented; overlaps conceptually with Zevra Agents (see [Findings](#findings)). KPI and version records appear lightly used.
**Documentation:** ◐ Partial — routing role covered in [conversational-analytics.md](../capabilities/conversational-analytics.md); the entity itself undocumented.

### Industry Packs

**Purpose:** versioned vendor catalogs of vertical content — entities, vocabulary, KPI definitions, agent definitions, alert templates — instantiated into tenant stores with pack provenance.
**Components:** `IndustryPackService`/`Controller`/`Repository`, `PackEntityMapper`, `PackRecommendationService`, pack content records.
**UI:** surfaced through onboarding.
**APIs:** `/industry-packs`.
**Tables:** pack catalog + `TenantPack` application records.
**Consumers:** semantic stores (instantiation target), onboarding.
**Status:** implemented (Phase 4); vocabulary provenance not yet stamped (see Semantic Foundation limitations).
**Documentation:** ✗ Not documented. Suggested: `platform/industry-packs.md`.

### Onboarding

**Purpose:** guided tenant setup — connection registration, schema discovery, metadata registration, pack recommendation/application, tenant settings.
**Components:** `OnboardingService`/`Controller`, `MetadataRegistrationService`, `TenantSettingsRepository`.
**UI:** Onboarding Wizard.
**APIs:** `/onboarding/*`.
**Tables:** tenant settings; writes into enterprise-map and semantic stores.
**Consumers:** first-run tenants.
**Status:** implemented.
**Documentation:** ✗ Not documented. Suggested: `getting-started/onboarding.md` + `platform/metadata-lifecycle.md`.

---

## Governance & Security

### SQL Governance Chain

**Purpose:** the execution gate for conversational SQL — safety validation, cost-based routing, contracts, row-level security, column masking, audit.
**Components:** `SqlSafetyService`, `QueryGovernanceService` (routing SYNC/ASYNC/BLOCK, row/cost/join/column limits), `DataContractService`, `RowLevelSecurityService`, `ColumnMaskingService`, `GovernanceAuditService` (+ 90-day audit retention purge), `UserAttributesRepository`, `GovernanceController`.
**UI:** Governance page (`/governance`) — contracts, RLS policies, column policies, audit events.
**APIs:** `/governance/*`.
**Tables:** data contracts, RLS policies, column policies, audit events.
**Dependencies:** user attributes/roles.
**Consumers:** conversational reasoning loop (fully), baseline creation (safety validation only); **not** agents or workflows.
**Status:** implemented; engaged only on the conversational path.
**Documentation:** ◐ Partial — documented in action in [conversational-analytics.md](../capabilities/conversational-analytics.md); no dedicated page. Suggested: `architecture/sql-governance.md` (the corpus's most-cited planned page).

### Async Query Execution

**Purpose:** queue for heavy queries routed async by governance (or `/async`); 5-second poller executes with extended limits.
**Components:** `QueryExecutionController`/`Repository`, poller inside `QueryGovernanceService`.
**APIs:** `/chat/async/{executionKey}` (ownership-verified).
**Tables:** query-execution records.
**Status:** implemented; no completion notification to the conversation.
**Documentation:** ◐ Partial — mentioned in conversational-analytics. Suggested: fold into `architecture/sql-governance.md`.

### Authentication

**Purpose:** identity — native signup/login with JWT sessions plus Supabase JWT verification.
**Components:** `AuthController`/`Service`/`Repository`, `NexusAuthFilter`, `SupabaseAuthFilter`, `UserAccount`, `UserSession` (24h expiry), `TenantDomain`/`TenantDomainRepository` (email-domain → tenant mapping), `SecurityConfig`.
**UI:** Login, SetNewPassword.
**APIs:** `/auth/signup`, `/auth/login` (public); the rest authenticated.
**Tables:** `nexus_user_account`, user sessions, tenant domains.
**Status:** implemented (dual-path: native JWT + Supabase). No SSO/SAML/OIDC beyond Supabase; no API keys; no MFA.
**Documentation:** ✗ Not documented. Suggested: `platform/authentication.md`.

### User Management

**Purpose:** tenant user administration — profiles, roles.
**Components:** `UserManagementController`/`Service`, `UserProfile`/`Repository`.
**UI:** Users page (`/users`).
**APIs:** `/auth/users`.
**Status:** implemented; role model is coarse (role string consumed by governance audit context and admin checks).
**Documentation:** ✗ Not documented. Suggested: `platform/users-and-roles.md`.

### Impersonation

**Purpose:** admin impersonation of tenant users (support/debugging).
**Components:** `ImpersonationFilter`.
**Status:** implemented (recently fixed — "Fixed Tenant personation" commit); a hidden platform service with no UI surface discovered.
**Documentation:** ✗ Not documented. Suggested: cover in `platform/authentication.md` + `operations/support-access.md`.

### Tenant Management & Provisioning

**Purpose:** multi-tenancy itself — tenant registry, schema-per-tenant provisioning, per-request schema routing, migration catch-up.
**Components:** `TenantController` (`/admin/tenants`, ADMIN-gated), `TenantProvisioningService`, `TenantSchemaMigrator` (startup catch-up), `TenantAwareDataSource` (search_path per request), `TenantContext`, `TenantRepository`.
**UI:** Tenant Admin page (`/tenants`).
**APIs:** `/admin/tenants/*`.
**Tables:** `nexus_tenant`, `public.nexus_session_index`.
**Consumers:** every capability (isolation model).
**Status:** implemented; schema-per-tenant with search_path routing.
**Documentation:** ◐ Partial — cited by every capability page. Suggested: `platform/tenancy-and-isolation.md` (second-most-cited planned page).

---

## Knowledge & User Experience

### Chat Attachments

**Purpose:** per-conversation file uploads — extracted text injected as reference data into planning.
**Components:** `AttachmentController` (`/chat/attachments`), `AttachmentProcessingService` (Tika/POI extraction + summary; hourly cleanup job), `ChatAttachment`.
**Tables:** attachment records.
**Consumers:** conversational pipeline (enriched question).
**Status:** implemented; content unscreened (prompt-injection surface, documented).
**Documentation:** ◐ Partial — behavior in conversational-analytics. Suggested: fold there permanently.

### Knowledge Gaps

**Purpose:** the record of what the platform couldn't answer — filed by the orchestrator and by user feedback; steward triage queue.
**Components:** `KnowledgeGapController`/`Repository`, `KnowledgeGap`.
**UI:** Knowledge Gaps page (`/gaps`).
**APIs:** `/knowledge-gaps`.
**Status:** implemented.
**Documentation:** ✗ Not documented. Suggested: `capabilities/knowledge-gaps.md` (small).

### Integration Templates

**Purpose:** integration request templates (ServiceNow-style) behind chat's `/request-source` flow.
**Components:** `IntegrationTemplateController`/`Service`.
**UI:** Templates page (`/templates`).
**APIs:** `/templates`.
**Status:** implemented; light.
**Documentation:** ✗ Not documented. Suggested: `capabilities/integration-templates.md` (small).

### Reasoning Investigations Surface

**Purpose:** browse reasoning sessions/findings outside chat; SSE stream endpoint for live traces.
**Components:** `ReasoningController` (`/reasoning`), `ReasoningStreamController` (`/chat/runs` SSE subscribe), `OperationalFinding` store.
**UI:** Reasoning page (`/reasoning`).
**Status:** implemented.
**Documentation:** ◐ Partial — trace/SSE covered in conversational-analytics; the standalone page undocumented.

### Notification Center

**Purpose:** the in-app bell — unread alert deliveries, mark-read.
**Components:** `NotificationPanel.jsx` over the alerts API.
**Status:** implemented; alerts-only (no other capability posts to it).
**Documentation:** ✅ Covered in [alerts.md](../capabilities/alerts.md).

### Settings

**Purpose:** tenant-facing settings surface (brief schedule/timezone/agent selection and related configuration).
**Components:** `Settings.jsx` over capability config APIs.
**Status:** implemented (Phase 5).
**Documentation:** ◐ Partial — per-capability config documented on capability pages.

---

## Administration & Operations

### Usage Tracking

**Purpose:** token/cost attribution for every AI call, tagged by context (`chat`, `agent`, `brief`, `report`) and user/agent.
**Components:** `UsageContext` (thread-local), `UsageService`/`Repository`/`Controller`.
**UI:** Usage page (`/usage`).
**APIs:** `/usage`.
**Status:** implemented; alert composition calls are untagged (documented gap).
**Documentation:** ✗ Not documented. Suggested: `operations/usage-and-cost.md`.

### Background Jobs (complete inventory)

| Job | Cadence | Owner |
|---|---|---|
| Async query queue poller | every 5s | `QueryGovernanceService` |
| Scheduled report runner | every 60s | `ScheduledReportService` |
| Morning brief scheduler | every 60s | `MorningBriefScheduler` |
| SSE emitter/buffer cleanup | every 60s | `ReasoningEventBus` |
| Baseline refresh + anomaly detection | hourly | `BaselineService` |
| Attachment cleanup | hourly | `AttachmentProcessingService` |
| Conversation retention purge | 02:00 UTC | `ConversationRetentionService` |
| Governance audit retention purge | 02:15 | `GovernanceAuditService` |
| Learned-mapping promote/purge | 02:45 | `SemanticLearningService` |

Plus async (non-scheduled) executors: document indexing, audit writes, semantic learning capture, brief generation (common pool).

### Migrations & Schema Management

**Purpose:** Flyway migrations on the platform schema (V001–V034+) plus per-tenant schema replay: provisioning migrates at creation, `TenantSchemaMigrator` catches up all tenant schemas at startup.
**Status:** implemented.
**Documentation:** ✗ Not documented. Suggested: `operations/migrations.md`.

### Health & Metrics

**Purpose:** Spring Actuator: `health` (details always shown), `info`, `metrics`, `prometheus` — all publicly exposed (`permitAll`).
**Status:** implemented; no custom health indicators discovered; mail health check disabled.
**Documentation:** ✗ Not documented. Suggested: `operations/monitoring.md`.

### Demo Seeding

**Purpose:** out-of-the-box explorability — a `local-db` demo connection (startup seeder) and two demo workflows (migrations V026/V027), plus demo schema/seed SQL in `sei-nexus-db`.
**Status:** implemented; runs in every environment (no environment gate discovered).
**Documentation:** ✗ Not documented. Suggested: note in `operations/`.

### Platform Configuration

**Purpose:** the full runtime configuration surface (`application.yml`): server/context-path (`/api/v1`), Supabase datasource + Hikari pool (max 5), SMTP, Supabase auth keys, CORS (`*` default), storage path, memory/embedding settings, OpenAI models (`gpt-4o`, `text-embedding-ada-002`), JWT secret (default `change-me-in-production`), context budget, query-governance limits (timeouts 10/30/90s; rows 100/500/10000; cost 500/10000; joins 4; columns 30), retention (3-day conversations, 90-day audit), app URLs.
**Status:** implemented; several insecure defaults (CORS `*`, placeholder JWT secret, hardcoded app URL).
**Documentation:** ✗ Not documented. Suggested: `operations/configuration.md` (most-suggested page in this inventory).

---

## Findings

Discovery observations only — no recommendations.

**Incomplete / partially implemented**

- Brief `email_to`: stored and editable, no delivery path exists.
- Alert delivery `FAILED` status: constrained in schema, never written by code.
- Automation execution `TIMEOUT` status: in schema, never assigned.
- Value-domain sources `CHECK`/`DOMAIN`/`OBSERVED`/`MANUAL`: modeled, not produced (only `ENUM` live).
- Vocabulary tier provenance: resolver seam exists, table column does not.
- `metric` resolution kind: reserved in the grammar, never emitted.
- `GET /temporal/anomalies/{key}`: deliberately returns 501.
- Learned-mapping promotion: threshold-automatic; the constitution's review gate is unimplemented.

**Dead / unused code and surfaces**

- `AgentRunner` injects `SemanticService` and `KnowledgeGraphService` — both unused.
- `ScheduledReportService` imports `NotificationDeliveryService` — unused (delivery re-implemented locally).
- `ChatService.generateInvestigationPlan` / `parsePlan` / `generateHypothesisText` — appear superseded by the iterative `ReasoningEngine`; no live call path found on the main pipeline.
- UI `pages/mockup/` (`HomePageMockup`, `MockupSidePanel`) and static `data/actions.ts`, `data/investigations.ts` — mockup/demo artifacts shipped in the app bundle.

**Duplicate / overlapping capabilities**

- **Two agent systems:** `NexusAgent` (chat routing agents with playbooks/KPIs/versions) and `ZevraAgent` (autonomous ReAct agents). Both are "agents" to users; only Zevra Agents have a management UI. `AgentKpi` and `AgentVersion` appear lightly used.
- **Three independent schedulers with the same pattern** (brief, reports, baselines) and **two independent email/Slack delivery implementations** (alerts, reports) sharing only configuration.
- **Two SQL execution postures** on one shared service: governed (chat) and ungoverned (agents, workflows) — plus creation-time-only validation (baselines).

**Hidden platform services (no UI surface)**

- Admin impersonation (`ImpersonationFilter`).
- Tenant-domain → tenant mapping (`TenantDomain`).
- Chat routing agents' playbooks/KPIs/versions (API-only).
- The async query queue (pollable by key only from chat responses).
- Demo seeding (runs unconditionally at startup).

**Commonly-expected enterprise features not present** (absence noted for SaaS-readiness scoping, not as recommendations): SSO/SAML/OIDC beyond Supabase JWT, MFA, API keys / service credentials, feature flags, licensing/entitlements, rate limiting, secret encryption for connection credentials (stored plaintext), object storage (documents on local disk), backup/restore tooling, and run-level timeouts/cancellation on synchronous AI paths.

---

## Capability Coverage

Status: **Implemented** (works end to end as documented) / **Implemented\*** (works with material documented gaps). Review columns reflect reality today: no formal architecture, enterprise, or stabilization review has been performed on any capability — the stabilization checklists on each documented page are the queued work.

| Capability | Status | Documentation | Architecture Reviewed | Enterprise Reviewed | Stabilized |
|---|---|---|---|---|---|
| Conversational Analytics | Implemented | ✅ Complete | No | No | No |
| Autonomous Agents | Implemented\* | ✅ Complete | No | No | No |
| AI Memory | Implemented | ✅ Complete | No | No | No |
| Executive Brief | Implemented\* | ✅ Complete | No | No | No |
| Semantic Foundation | Implemented\* | ✅ Complete | No | No | No |
| Knowledge Graph | Implemented | ◐ Partial | No | No | No |
| Workflow Automation | Implemented\* | ✅ Complete | No | No | No |
| Scheduled Reports | Implemented\* | ✅ Complete | No | No | No |
| Alerts | Implemented\* | ✅ Complete | No | No | No |
| Temporal Intelligence | Implemented | ◐ Partial | No | No | No |
| Enterprise Map | Implemented | ◐ Partial | No | No | No |
| Connection Registry | Implemented\* | ✗ None | No | No | No |
| Business Domains | Implemented | ✗ None | No | No | No |
| Chat Routing Agents | Implemented | ◐ Partial | No | No | No |
| Industry Packs | Implemented | ✗ None | No | No | No |
| Onboarding | Implemented | ✗ None | No | No | No |
| SQL Governance Chain | Implemented | ◐ Partial | No | No | No |
| Async Query Execution | Implemented | ◐ Partial | No | No | No |
| Authentication | Implemented | ✗ None | No | No | No |
| User Management | Implemented | ✗ None | No | No | No |
| Impersonation | Implemented | ✗ None | No | No | No |
| Tenant Management & Provisioning | Implemented | ◐ Partial | No | No | No |
| Chat Attachments | Implemented | ◐ Partial | No | No | No |
| Knowledge Gaps | Implemented | ✗ None | No | No | No |
| Integration Templates | Implemented | ✗ None | No | No | No |
| Reasoning Investigations Surface | Implemented | ◐ Partial | No | No | No |
| Notification Center | Implemented | ✅ (in Alerts) | No | No | No |
| Settings | Implemented | ◐ Partial | No | No | No |
| Usage Tracking | Implemented\* | ✗ None | No | No | No |
| Background Jobs | Implemented | ◐ Partial (per capability) | No | No | No |
| Migrations & Schema Management | Implemented | ✗ None | No | No | No |
| Health & Metrics | Implemented | ✗ None | No | No | No |
| Demo Seeding | Implemented | ✗ None | No | No | No |
| Platform Configuration | Implemented\* | ✗ None | No | No | No |
