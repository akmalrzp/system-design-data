---
type: usecase
title: "Use case: delete"
tags: [proposal]
sources: [osdm-integration-summary,author-decisions-2026-10-09]
created: 2026-10-09
updated: 2026-10-21
---

# Use case: delete

Policy: [decision-delete-policy-and-erasure](../decisions/decision-delete-policy-and-erasure.md).

**Normal retirement**
1. Owner sets status `inactive`/`terminated`, new version, `effective_to`; publishes.
2. Consumers apply the status; records referencing the entity remain.

**Effect by module type**
- Owner: row remains.
- Type 1: shows status; new records may not reference it (validation) unless history entry.
- Type 2: flag or close records with `effective_to`; do not delete.

**Erasure (exception)**
1. Owner anonymises in place (keeping the ID) or hard-deletes, and publishes an erasure event with a new version.
2. Every consumer applies it to caches and projections through the version-guarded apply path.
3. Each module's purge job, triggered by the same event, clears logs, outbox payloads, snapshots and backups, and reports completion.
4. Reconcile to confirm.

**Edge cases**: stale update after delete (rejected by version; state is retained); delete of an entity still referenced by open records (block or archive per business rule); resurrection by an old batch (version guard); consumer unreachable during erasure (queue; track completion).
