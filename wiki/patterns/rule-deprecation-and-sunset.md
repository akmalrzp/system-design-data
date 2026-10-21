---
type: pattern
title: "Rule: deprecation and sunset"
tags: [partly-verified]
sources: [osdm-integration-summary,api-design-for-system-integration]
created: 2026-10-09
updated: 2026-10-21
---

# Rule: deprecation and sunset

To remove a field or endpoint:
1. Mark deprecated in the spec.
2. Send `Deprecation` and `Sunset` headers (hints; clients must work without them).
3. Watch usage until it reaches zero.
4. Remove only then, and for incompatible changes publish a new major version.

Release pipeline: a contract check (the spec diff in [rule-contract-testing](rule-contract-testing.md)) blocks removals, renames, retypings and new required fields. Related: [rule-additive-only-contract](rule-additive-only-contract.md), [decision-contract-versioning-style](../decisions/decision-contract-versioning-style.md).
