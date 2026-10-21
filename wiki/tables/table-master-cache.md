---
type: table-type
title: "Cache of master reference (consumer)"
tags: [unverified]
sources: [osdm-integration-summary,api-design-for-system-integration]
created: 2026-10-09
updated: 2026-10-21
---

# Cache of master reference (consumer)

A consumer's read-only local copy of an owner's records.

**Typical columns**
- Local primary key, `source_id` (the owner's ID, unique), `source_version`, only the cached fields needed, `status`, `replaced_by`, `last_synced_at`.
- `effective_from`, `effective_to` when the consumer validates effective-dated records against it ([consumer-index-user](../modules/consumer-index-user.md)).

**Rules**
- Written only by the apply path ([rule-idempotent-versioned-apply](../patterns/rule-idempotent-versioned-apply.md)); never edited by users or other processes.
- Apply only if event version is greater than `source_version`; insert when no row exists.
- Stores deletion and inactive states, not absence ([rule-deletes-as-state](../patterns/rule-deletes-as-state.md)).
- Unknown IDs: placeholder row or lazy fetch. A placeholder row has no `source_version`, so the first real event or fetch always fills it.
- Local extensions go in a separate table keyed by the owner's ID, not in the cache.

**Accountability**: processed-event log (event ID or entity ID plus version, committed in the same transaction as the change) and reconciliation results ([flow-reconciliation](../patterns/flow-reconciliation.md)). Lineage columns `source_version` and `last_synced_at` show which owner state the copy reflects.
