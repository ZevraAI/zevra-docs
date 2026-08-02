---
description: What Zevra does today, what is deliberately out of scope by design, and what is planned next — kept strictly separate.
---

# 17 — Future Roadmap

This closes the documentation set. Every prior document described the architecture as it exists today. This one draws a clear line between that reality, deliberate current boundaries, and genuine forward direction — the three are easy to blur in conversation and are kept apart here on purpose.

## Purpose

An architecture review loses credibility the moment "what we have" and "what we intend" are presented as the same thing. This document exists so that every claim made earlier in this set can be trusted as current fact, and every forward-looking statement is clearly labeled as intent, not commitment.

## Current Capabilities

Everything described in this documentation set is implemented and operating today:

- The Executive Brief, generated on a schedule or on demand, synthesized from the tenant's own active agents investigating with real queries.
- The Investigation Workspace, with live and report modes, multi-turn follow-up, visible evidence, and a visible reasoning trace.
- The full reasoning pipeline: business-language resolution, the Agent Brain, the compiled Execution Contract, the bounded plan-validate-execute-evaluate loop, and governed, read-only execution.
- Scheduled Reports, delivering the same governed pipeline's answers as email or Slack digests on a recurring cadence.
- Alerts, driven by anomaly detection, delivered across in-app, email, and Slack channels.
- The shared response-composition core, serving the Brief, the Workspace, Reports, and Alerts from consistent, deterministic evidence interpretation.
- The full governance chain — safety, data contracts, row-level security, masking, routing, and audit — applied uniformly to every governed query regardless of which surface it came from.
- Schema-per-tenant isolation, Supabase Auth-based authentication, and tenant auto-provisioning by email domain.
- The design system and the executive experience layer described in [05](05-frontend-architecture.md), [06](06-design-system.md), and [07](07-executive-experience.md).

## Deliberate Current Boundaries

These are not gaps to be closed — they are intentional properties of the architecture, stated plainly so they are never mistaken for oversights:

- **Read-only, by design.** Zevra does not write to a tenant's operational systems and has no roadmap item to change that; awareness and action are deliberately kept separate.
- **No email delivery for the Executive Brief.** Recipient configuration exists, but the Brief is a UI-only surface today — deliberately distinct from Scheduled Reports, which do deliver externally.
- **The knowledge graph is advisory only.** It informs prompts; it does not participate in resolution or validation, by design.
- **Universal/common-knowledge language is never stored as tenant metadata.** Common terms are handled by the model itself and only enter tenant vocabulary through evidenced, validated use — this is what keeps the metadata layer model-independent.

## Genuinely Planned Work

Recorded as direction, not commitment — sequencing and scope may change:

- **Automated promotion review.** Learned vocabulary today promotes into curated language automatically at usage thresholds; a human review gate ahead of promotion is planned to close this gap.
- **Broader value-domain sources.** Only one discovery source for legal values is fully live today; additional sources are designed into the model and planned for activation.
- **Deeper agent-runtime alignment with the metadata layer.** Autonomous agents currently read a narrower slice of tenant metadata than the conversational pipeline does; bringing them onto the same resolution and validation guarantees is planned.
- **Citations as a shared capability.** Response composition does not yet track and surface citations from evidence to answer; this is a candidate for a future shared capability once the need is concrete enough to justify it, following the same evidence-based standard used for today's shared components.
- **Continued executive-experience investment.** The experience layer's roadmap continues to add further "ambient" behaviors on the same dependency-light, tokens-only, reduced-motion-respecting foundation already shipped.

## How to Read This Page During Review

If a question in the review is about what Zevra does *today*, the answer is in the Current Capabilities section above, and in the documents that precede this one. If the question is about direction, the Genuinely Planned Work section is the honest answer — framed as intent, never as an existing feature. Nothing in this document should be read back into the earlier documents in this set as something already built.

## References

- [Semantic Foundation — Current Limitations](../architecture/semantic-foundation.md#current-limitations)
- [Executive Brief capability — Current Limitations](../capabilities/executive-brief.md#current-limitations)
- [Roadmap section](../roadmap/index.md)
