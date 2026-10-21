---
type: pattern
title: "Rule: contract testing"
tags: [partly-verified,proposal]
sources: [osdm-integration-summary,api-design-for-system-integration]
created: 2026-10-09
updated: 2026-10-21
---

# Rule: contract testing

Catch breaking changes before release.

- **Spec diffing in CI**: compare the new spec with the previous one; simplest when there are many or unknown consumers.
- **Consumer-driven contract tests**: consumers record what they use and replay it against the owner; viable with one producer and one known consumer. Classic tools fit event-driven systems poorly.
- **Bi-directional contract testing**: compares the owner's spec with consumers' recorded usage; lighter, weaker guarantees.

**Default** (proposal): spec diffing in CI for request/response specs, plus the schema registry's compatibility check for events ([rule-additive-only-contract](rule-additive-only-contract.md)). Add consumer-driven tests only for a critical owner-consumer pair. Bi-directional testing is not the default: it needs every consumer to publish its usage and a broker product.

Limits of the default: a spec diff catches structural breaks (removed, renamed or retyped fields, new required fields), not a change of meaning; and it does not show which consumer uses a field, so watch usage before removing one ([rule-deprecation-and-sunset](rule-deprecation-and-sunset.md)).

Choice by consumer count is an inference. Sources were secondary ([source-reliability](../ops/source-reliability.md)).
