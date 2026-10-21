---
type: pattern
title: "Rule: additive-only contract"
tags: [partly-verified]
sources: [osdm-integration-summary,api-design-for-system-integration]
created: 2026-10-09
updated: 2026-10-21
---

# Rule: additive-only contract

**Safe changes**: add optional fields (with a defined default or meaning when absent), endpoints, enum values.
**Breaking changes**: remove, rename or retype a field; change resource names, value formats or defaults; add a new required field.

Practices:
- Contract first: write and review the spec before implementing.
- Expose the minimum set of fields at first. What is exposed and relied on is effectively permanent.
- Keep the published model separate from the internal model.
- Consumers read only the fields they need, ignore unknown fields and unknown enum values, and fail loudly on missing or malformed required data (tolerant reader, bounded by the position that blanket tolerance is harmful).
- For events, enforce compatibility in a registry; FULL or FULL_TRANSITIVE lets either side upgrade first (the TRANSITIVE preference is an inference).

Retire fields through [rule-deprecation-and-sunset](rule-deprecation-and-sunset.md). Verify with [rule-contract-testing](rule-contract-testing.md). Style choice: [decision-contract-versioning-style](../decisions/decision-contract-versioning-style.md).
