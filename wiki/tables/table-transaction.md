---
type: table-type
title: "Transaction table"
tags: [proposal]
sources: [osdm-integration-summary]
created: 2026-10-09
updated: 2026-10-21
---

# Transaction table

A module's own business records: assignments, requirements, payroll runs. They reference IDs from caches or from their own master tables.

**Rules**
- Hold foreign IDs, not copied descriptive text, unless the record must be reproducible as it was (then store a snapshot; see below).
- No cross-system foreign key constraint. Validate against the local cache on write ([rule-validate-ids-locally](../patterns/rule-validate-ids-locally.md)).
- Effective-date records whose meaning depends on time (`effective_from`, `effective_to`).
- Append-only or versioned for records that must be auditable.

**Snapshot or reference?** Store the reference when the current name is wanted; store a snapshot (value plus the source version) when a legal or business need requires the document to reproduce what was true at the time (payslips, decision letters). Snapshots are in erasure scope ([decision-delete-policy-and-erasure](../decisions/decision-delete-policy-and-erasure.md)).

**Accountability**: record who, when and why for each write; keep an append-only log when the record is a business fact (for example an assignment). See [accountability-options](accountability-options.md).
