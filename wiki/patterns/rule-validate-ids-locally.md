---
type: pattern
title: "Rule: validate IDs against the local cache"
tags: [proposal]
sources: [osdm-integration-summary]
created: 2026-10-09
updated: 2026-10-21
---

# Rule: validate IDs against the local cache

Before accepting a write that references another module's entity, validate the ID against the local cache ([table-master-cache](../tables/table-master-cache.md)): it exists and is active (or, for an effective-dated record, valid over the record's effective period).

- A validates `unit_id` and `position_id` against its cache of C before accepting an assignment.
- D validates the unit before accepting a requirement.
- Unknown ID: reject, or accept as pending and re-validate, per business rule ([edge-case-owner-unavailable](../modules/edge-case-owner-unavailable.md)).
- No cross-system foreign key constraints.
- This rule covers writes a module accepts from its users. It does not apply to events from an owner: those were validated by the owner ([edge-case-divergent-reference-caches](../modules/edge-case-divergent-reference-caches.md)).
