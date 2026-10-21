---
type: module-type
title: "Consumer type 2: reference data as index"
tags: [proposal,unverified]
sources: [osdm-integration-summary]
created: 2026-10-09
updated: 2026-10-21
---

# Consumer type 2: reference data as index

**Role.** Uses another module's ID as the business key or classification axis of its own records. The reference defines what the consumer's records mean.

Example: D stores "required employees per unit". `unit_id` identifies the requirement.

**Design (same cache pattern as type 1, plus)**
- Validate the referenced entity exists and is active before accepting a record; for effective-dated records, that it is valid over the record's effective period. This allows backdated corrections and forward planning ([rule-validate-ids-locally](../patterns/rule-validate-ids-locally.md)). The cache therefore keeps the entity's `effective_from` and `effective_to`.
- Effective-date records (`effective_from`, `effective_to`) so they stay meaningful after the reference changes.
- Treat owner lifecycle events as business events:
  - Inactive: flag the record for review or close it with `effective_to`.
  - Merge or split: flag the records and let a person decide. Never move them silently. See [uc-merge](../usecases/uc-merge.md), [uc-split](../usecases/uc-split.md).
  - Hierarchy change: roll-up totals change; read `parent_unit_id` from the local cache ([edge-case-hierarchy-changes](edge-case-hierarchy-changes.md)).
- If it needs derived data from another owner (headcount per unit from A), keep a projection, not copies ([table-projection](../tables/table-projection.md)).
- Show "as of" timestamps on projections ([eventual-consistency](../concepts/eventual-consistency.md)).
- If another module emits the events the projection counts, its cache of the owner may differ from yours: [edge-case-divergent-reference-caches](edge-case-divergent-reference-caches.md).

**Impact of owner lifecycle**: high. Agree the lifecycle contract with the owner early.

Contrast with [consumer-attribute-user](consumer-attribute-user.md).
