---
type: decision
title: "Decision: delete policy and the erasure exception"
tags: [option,proposal]
sources: [author-decisions-2026-10-09,osdm-integration-summary,api-design-for-system-integration]
created: 2026-10-09
updated: 2026-10-21
---

# Decision: delete policy and the erasure exception

**Default**: no hard delete. The owner sets a lifecycle status with a new version; consumers keep the state ([rule-deletes-as-state](../patterns/rule-deletes-as-state.md)).

**Exception (allowed by the author)**: legal erasure or anonymisation of personal data, and correction of records created in error.

| Case | Mechanism | Notes |
|---|---|---|
| Normal retirement | Status `inactive` or `terminated` | Reversible; history kept |
| Merge / split | Status plus `replaced_by` | See [uc-merge](../usecases/uc-merge.md), [uc-split](../usecases/uc-split.md) |
| Created in error | Mark `void`; hard delete only if never published | If published, treat as erasure |
| Personal-data erasure | Anonymise fields in place, keep the ID, issue a new version; or hard delete with an erasure event | Event must reach every copy; see below |

**Erasure design options**
- *Anonymise in place* (recommended starting point, proposal): replace personal fields with null/placeholder, keep the ID and status, and increment the version. Consumers apply it as a normal update, so no stale copy can resurrect the data, and references stay valid.
- *Hard delete plus erasure event*: remove the row and send an event telling consumers to remove theirs. Use when the law or policy requires the identifier itself to go. Consumers must then handle dangling references ([consumer-attribute-user](../modules/consumer-attribute-user.md), [consumer-index-user](../modules/consumer-index-user.md)), and the "never reuse an ID" rule must still hold.

**Required for either**
- The erasure always carries a new version. An erasure at the stored version is ignored by every consumer's version guard as a duplicate, and the data stays.
- Caches and projections erase through [rule-idempotent-versioned-apply](../patterns/rule-idempotent-versioned-apply.md), like any update.
- Logs, outbox payloads, snapshots and backups are not reached by the apply path. A purge job in each module, triggered by the same erasure event, clears them ([table-transaction](../tables/table-transaction.md) snapshots; [accountability-default](../tables/accountability-default.md) logs). Completion is tracked per consumer; ad-hoc scripts leave no record.
- Reconciliation confirms no copy remains ([flow-reconciliation](../patterns/flow-reconciliation.md)).
- Legal requirements (UU PDP, GDPR) were not researched; confirm with legal counsel ([security-and-erasure-propagation](../patterns/security-and-erasure-propagation.md)).

Use case: [uc-delete](../usecases/uc-delete.md).
