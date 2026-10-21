---
type: module-type
title: "Data owner and broadcaster (source of truth)"
tags: [unverified]
sources: [osdm-integration-summary,api-design-for-system-integration]
created: 2026-10-09
updated: 2026-10-21
---

# Data owner and broadcaster (source of truth)

**Role.** Creates, changes and retires records of its entities, issues their IDs ([opaque-permanent-id](../concepts/opaque-permanent-id.md)), and publishes changes.

**Must provide**
- Master reference tables ([table-master-reference](../tables/table-master-reference.md)).
- A change feed: versioned events through a transactional outbox ([flow-events-plus-read-api](../patterns/flow-events-plus-read-api.md)).
- A read API for lookup by ID and cursor-paginated catch-up by `updated_after`, sorted by (`updated_at`, `id`) ([flow-bootstrap-and-catch-up](../patterns/flow-bootstrap-and-catch-up.md)).
- A crosswalk from legacy-system codes to its IDs, when records were migrated from a legacy system ([table-crosswalk](../tables/table-crosswalk.md)).
- A published, versioned, additive contract ([rule-additive-only-contract](../patterns/rule-additive-only-contract.md)).
- Lifecycle states instead of hard delete ([rule-deletes-as-state](../patterns/rule-deletes-as-state.md)), with `replaced_by` for merge and split.
- Field scoping per consumer for sensitive data ([payload-sensitive-fields](../patterns/payload-sensitive-fields.md)).

**Must not**
- Publish data it does not own (A publishes IDs of units, not unit names).
- Expose its internal model or database.
- Reuse or change an ID.

**Edge cases**: [edge-case-field-ownership](edge-case-field-ownership.md), [edge-case-hierarchy-changes](edge-case-hierarchy-changes.md), [edge-case-owner-unavailable](edge-case-owner-unavailable.md), [edge-case-business-key-vs-id](edge-case-business-key-vs-id.md).

Example: C (Unit, Position), and A for Employee ([worked-example](../concepts/worked-example.md)).
