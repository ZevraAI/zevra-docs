---
description: Canonical reference for Business Values — a tenant-owned semantic layer over the existing Value Domains. Additive metadata only; Runtime, Execution Contract, and the Metadata Registration Pipeline are unchanged.
---

# Business Value Architecture

**Status:** implemented (additive). Business Values are **semantic metadata only** — they reach the LLM prompt and never affect deterministic execution. AgentBrain reasons about Business Values; Runtime executes physical values.

## 1. Purpose

Observed/authoritative **Value Domains** already discover the *physical* values a column may hold (`10, 20, 30, 40`). They do not carry *meaning*. Business Values add a canonical concept (`Draft`, `Submitted`, `Approved`, `Closed`) on top of those physical values so the LLM can reason about the concept while the runtime still executes the physical code.

```
Business Object → Business Attribute → Value Domain → Observed Physical Value → Business Value
```

## 2. Metadata model (two new tenant-owned objects)

Everything below lives in the **tenant schema** (per-tenant isolation; no shared/global metadata). Physical values remain owned by `nexus_value_domain.domain_values` (a JSON array); **no physical value is stored twice** — mappings reference values by natural key.

### `nexus_business_value` — the canonical concept
| Column | Notes |
|---|---|
| `business_value_key` (PK) | identity |
| `business_attribute_key` | logical attribute the concept belongs to (scoping; keeps homonyms under different attributes distinct) |
| `name` | canonical concept, e.g. `Draft` |
| `description` | optional |
| `source` | `AI` \| `MANUAL` |
| `confidence` | 0.000–1.000, nullable |
| `approval_status` | `PENDING` \| `APPROVED` \| `REJECTED` |
| `created_by`, `approved_by`, `approved_at`, `created_at`, `updated_at` | audit |
| **UNIQUE** `(business_attribute_key, name)` | duplicate-concept prevention |

### `nexus_business_value_mapping` — physical value → concept
| Column | Notes |
|---|---|
| `mapping_key` (PK) | identity |
| `value_domain_key` | FK → `nexus_value_domain` (existing) |
| `physical_value` | natural key within the domain (an element of `domain_values`) |
| `business_value_key` | FK → `nexus_business_value` (ON DELETE CASCADE) |
| `source`, `confidence`, `approval_status` | provenance/governance |
| `cross_application` | when true, requires customer approval |
| audit fields | |
| **UNIQUE** `(value_domain_key, physical_value)` | exactly one concept per physical value |

Migration: `V037__business_value.sql` (runs in every tenant schema, additive).

## 3. Object relationships

```
nexus_data_object ─1─* nexus_data_column ──(value_domain_key)──► nexus_value_domain
                                                                        │  (domain_values JSON: "10","20","30","40")
                                                                        │
                              nexus_business_value_mapping ─(value_domain_key, physical_value)─┘
                                        │ (business_value_key)
                                        ▼
                              nexus_business_value  ── "Draft" (scoped to business_attribute_key)
```
Cross-application: one Business Value is referenced by many mappings across different Value Domains (RMS status `10`, EBS status `D`) — the concept is shared; the physical codes are per application.

## 4. Mapping rules / uniqueness constraints

- **Exactly one Business Value per `(value_domain_key, physical_value)`** — enforced by the DB unique constraint. `physical→canonical` is a function.
- **A Business Value may map to many physical values** — allowed (e.g. `Draft = '10' | '11'`). `canonical→physical` may be 1:N.
- **No duplicate concept name within a business attribute** — enforced by `UNIQUE(business_attribute_key, name)`; homonyms under *different* attributes are distinct.
- **No new physical-value storage** — mappings reference existing `domain_values` elements by natural key.

## 5. Onboarding flow (existing pipeline, extended — no new pipeline)

```
Scan ─► Observed/Authoritative Value Domain ─► (existing)
     ─► AI proposes Business Values + mappings   (analyzeForOnboarding)
     ─► Customer Review (confirm / edit)          (/discover review)
     ─► Persist Business Values + Mappings        (MetadataRegistrationService.register step 5)
```
`MetadataRegistrationService.register` gains **step 5**: for each approved entity it persists optional `businessValues[]` and `businessValueMappings[]` from the review output, applying the governance rules. Absent arrays ⇒ no-op (every existing payload is unaffected). Apply-payload shape (per entity):
```
businessValues:        [{ approved, businessValueKey?, businessAttributeKey, name, description?, source?, confidence?, approvalStatus? }]
businessValueMappings: [{ approved, valueDomainKey, physicalValue, businessValueKey, source?, confidence?, crossApplication? }]
```

## 6. Governance

Deterministic rules in `BusinessValueGovernance` (pure, unit-tested):
- **Duplicate detection** — same concept name under the same attribute is a duplicate (merge, don't create).
- **Conflict detection** — a physical value already mapped to a *different* concept is a conflict (also rejected by the DB constraint); surfaced as a failure, never silently overwritten.
- **Approval requirement** — **cross-application** mappings *always* require customer approval; low-confidence AI mappings (< 0.55) require review; **MANUAL** and high-confidence single-application AI mappings may auto-approve.

## 7. Approval workflow

| State | Meaning | Projected to LLM? |
|---|---|---|
| `PENDING` | proposed (AI), or cross-app/low-confidence awaiting review | **No** |
| `APPROVED` | customer-confirmed or auto-approved (high-confidence single-app) | **Yes** |
| `REJECTED` | declined | No |

Only **APPROVED** mappings are projected into the Execution Contract / prompt (`BusinessValueRepository.findApprovedMappingsByDomain`). Unconfirmed meaning never reaches the LLM, and never affects execution.

## 8. Projection to the LLM (Commit 2)

`EnterpriseSemanticAssembler.valueDomainOf` reads approved mappings for a domain and attaches them to the existing `ColumnValueDomain` as an additive `labels` map (`physicalValue → conceptName`). `PromptAssembler.renderValueDomain` renders them:

```
    • `status` (STATUS)  [observed values: 10 = Draft | 20 = Submitted | 30 = Approved | 40 = Closed]
```
`ColumnValueDomain` is **not persisted**; the labels are a per-query projection. `LiteralValidator` and the Execution Contract shape are unchanged — labels are presentation-only.

## 9. What is explicitly NOT changed

- **Runtime / SQL compilation / LiteralValidator** — untouched; validation is membership-only (`domain.values()`), never meaning.
- **Execution Contract architecture** — unchanged; labels ride the existing `ColumnValueDomain`.
- **Value Domains** — remain the physical source of truth; observed domains are append-only (retention preserves previously observed values so mappings stay stable).
- **No `ObservedValue`/`PhysicalValue`/`Alternate ValueDomain`/persisted `ColumnValueDomain`/parallel model.**

## 10. Future extension points (additive, no redesign)

- **Multilingual labels / aliases** — additive columns/rows on `nexus_business_value`.
- **Value ordering / lifecycle sequence** — an ordinal/relationship field for "before approval" reasoning.
- **Effective-dated mappings** — temporal columns for ERP migrations / historical accuracy.
- **1:N canonical→physical** at execution — IN-list translation (metadata already supports many mappings per concept).
- **ERP dictionary import** — an adapter feeding the same mapping table.

These are deliberately out of the initial implementation; the model absorbs each as additive metadata without touching AgentBrain, the Execution Contract, or the Runtime.
