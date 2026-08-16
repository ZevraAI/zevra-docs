---
description: How a new customer connects their data and how Zevra builds the Business World it reasons over.
---

# Zevra Customer Onboarding

Onboarding is how a new Zevra tenant goes from "an empty workspace" to "a Business World Zevra can reason over." It happens once per data source, is guided end to end, and every result is reviewed by the customer before it becomes part of their tenant. This page describes what a customer sees and does; it is not an implementation reference.

## What onboarding is

Onboarding connects Zevra to a customer's own database and turns the tables it finds there into named, described business concepts — a **Business World** — that later powers conversations, reports, and agents. Nothing is guessed at and silently kept: every concept Zevra proposes is shown to the customer for review before it's saved.

Onboarding is typically a short, guided session: connect a data source, let Zevra look at it, review what it found, and confirm. A customer with a handful of core tables can usually finish in a few minutes; larger schemas take a bit longer only because there's more to review, not because the process itself is more complex.

## Connecting customer data

The customer supplies connection details for their database (or accepts a pre-configured connection for a demo/trial). Zevra never requires the customer to describe their schema up front — it reads the schema itself once connected.

## Discover from DB

Once connected, Zevra scans the customer's tables and columns directly from their database. This is a read-only inspection: Zevra sees table names, column names, and data types, and forms an initial picture of what's available. Nothing is written to the customer's Business World at this stage — discovery only produces a list of candidate tables for the customer to consider.

## How Zevra builds the Business World

For each table the customer chooses to bring in, Zevra proposes a named business concept — a **Business Object** — such as "Purchase Order" or "Store," along with a short description of what it represents and how it's typically used. These proposals are drafted for the customer's review, not saved automatically.

Once a proposal is approved, Zevra records two things together: the business concept itself, and its permanent link back to the exact physical table it came from. That link is what lets Zevra later retrieve the right data with certainty whenever that business concept is needed — the business name is for people (and, eventually, for the AI) to reason with; the link underneath is what makes retrieval exact.

## Business concepts

A Business Object carries:

- **A business name** a person recognizes — "Purchase Order," not `po_hdr_tbl`.
- **A short description** of what it represents and when it's used.
- **A domain** — a grouping such as "Purchasing" or "Inventory" that organizes related concepts together.
- **A link to its source table**, so every reference to the concept resolves to real customer data, not a guess.

Zevra also captures the business vocabulary that goes with a concept — terms, abbreviations, and phrasing a customer's own team uses — so common shorthand is understood without the customer having to spell it out every time.

## Industry Pack recommendations

For common business domains — retail, healthcare, logistics, and others — Zevra maintains starter packs of well-known business concepts and vocabulary for that industry. After discovery, Zevra checks whether the customer's tables look like a good match for one of these packs and, if so, recommends it.

## Applying Industry Packs

Applying a pack is optional and reviewable, just like any other onboarding step. When applied, Zevra matches each concept in the pack (for example, "Product," "Supplier," "Warehouse") to the customer's actual, already-discovered tables and creates the corresponding Business Objects — pre-populated with industry-standard names, descriptions, and vocabulary, so the customer isn't starting from a blank page. A pack concept that doesn't match any of the customer's tables is left out rather than created without a real connection to their data.

## How Zevra connects concepts to customer data

Whether a Business Object comes from ordinary discovery or from an applied pack, it is always tied back to one specific table in the customer's database. That tie is exact and permanent — not a best guess re-evaluated every time a question is asked. It's what lets Zevra retrieve the right data deterministically once a business concept has been identified, instead of re-interpreting the customer's schema on every question.

## Repeated discovery

Customers can re-run discovery at any time — after adding new tables, reconnecting a data source, or simply to review the current state. Running discovery again does not create duplicate business concepts for tables Zevra already represents: it recognizes the table it's looking at and reuses the existing Business Object rather than creating a second one. Only genuinely new tables produce new Business Objects.

## What the customer reviews and approves

Nothing from discovery, pack matching, or AI-drafted descriptions becomes part of the Business World without the customer seeing it first. For each candidate the customer can:

- Accept it as proposed.
- Edit the name, description, or grouping before accepting.
- Skip it — the underlying table stays available to bring in later.

Approval happens once per candidate, not once per field — the customer reviews a complete proposal, not a form.

## What onboarding completion means

When the customer finishes reviewing and approving, their Business World reflects exactly what they approved — nothing more. From that point on, the customer's connected data is available to Zevra's conversational analytics and agents through the business concepts just created, and the customer can return to onboarding at any time to bring in more tables, review an Industry Pack, or make changes.

## What the customer should expect

- **Nothing is exposed without review.** Every business concept, whether discovered or pack-suggested, is a proposal until approved.
- **Business names are for people; the underlying link is exact.** A concept's friendly name can be edited freely — the connection to the customer's real data underneath does not depend on that name matching anything.
- **Re-running discovery is safe.** It will not multiply Business Objects for tables the customer already has represented.
- **Onboarding is incremental.** Customers are not required to bring in their entire schema at once — additional tables can be onboarded later through the same guided flow.

---

## Internal Product Validation

*(Internal note — not part of the customer-facing onboarding experience.)*

Before the current onboarding foundation is used as the source for Zevra's Global Business World List (the mechanism that will let the reasoning engine, rather than backend code, decide which business concepts a question needs), a clean-tenant validation run is planned: onboarding a brand-new tenant against a real customer-shaped source database, end to end — discovery, review, an applied Industry Pack, and a second discovery pass — to confirm the current implementation behaves as verified in code review before any customer-facing Business World List work begins. This validation does not change anything described above for customers; it confirms the foundation those steps already produce.
