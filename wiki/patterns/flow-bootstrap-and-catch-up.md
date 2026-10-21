---
type: pattern
title: "Flow: bootstrap and catch-up"
tags: [partly-verified]
sources: [osdm-integration-summary,api-design-for-system-integration]
created: 2026-10-09
updated: 2026-10-21
---

# Flow: bootstrap and catch-up

A new consumer, a rebuilt consumer, or a consumer that missed events uses the same path:

1. Before the first page, record the event-stream position (or start consuming events).
2. Page through the owner's list endpoint with `updated_after` (or from the start), using opaque cursors.
3. Apply each record through the normal version-guarded apply ([rule-idempotent-versioned-apply](rule-idempotent-versioned-apply.md)).
4. Consume events from the position recorded in step 1. The version check discards anything older than what paging already applied.
5. Run reconciliation ([flow-reconciliation](flow-reconciliation.md)).

Do not subscribe only after paging ends: changes made during paging to records already paged past would be lost until reconciliation. The event channel must therefore retain events at least as long as a full bootstrap takes ([technology-neutral-requirements](technology-neutral-requirements.md)).

**Cursor.** The owner sorts the list by (`updated_at`, `id`) and the opaque cursor encodes both. A cursor on `updated_at` alone skips rows that share a timestamp at a page boundary.

Using one apply path for both bootstrap and events removes a class of divergence bugs. Details in [uc-initial-broadcast](../usecases/uc-initial-broadcast.md) and [uc-late-joiner-and-gaps](../usecases/uc-late-joiner-and-gaps.md).

Snapshot-plus-change-feed bootstrap had no usable source in the research ([source-reliability](../ops/source-reliability.md)); treat the ordering in steps 1 and 4 as design reasoning.
