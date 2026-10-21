---
type: ops
title: "Design checklist for a new module"
tags: [proposal]
sources: [osdm-integration-summary,api-design-for-system-integration,author-decisions-2026-10-09]
created: 2026-10-09
updated: 2026-10-21
---

# Design checklist for a new module

**Scope**
- [ ] Entities owned and consumed listed; each classified ([data-owner-broadcaster](../modules/data-owner-broadcaster.md), [consumer-attribute-user](../modules/consumer-attribute-user.md), [consumer-index-user](../modules/consumer-index-user.md), [hybrid-module](../modules/hybrid-module.md)).
- [ ] Scope criteria applied ([module-scope-criteria](../modules/module-scope-criteria.md)); features placed ([feature-placement-test](../modules/feature-placement-test.md)).
- [ ] Dependency graph per entity is acyclic.

**Tables**
- [ ] Each table typed ([guide-02-table-types](../guide/guide-02-table-types.md)); accountability chosen ([accountability-default](../tables/accountability-default.md)).
- [ ] No cross-system foreign keys; ID columns unique; business keys as attributes.

**Communication**
- [ ] Owner: ID format, versioning, outbox, events, read API, contract, sensitive-field scoping.
- [ ] Consumer: apply path idempotent and version-guarded; adapter at the edge; unknown-ID handling.
- [ ] Bootstrap and reconciliation planned.
- [ ] Owner offline: the module keeps reading and validating from its cache; writes that need an unseen ID are rejected or held as pending, per business rule ([edge-case-owner-unavailable](../modules/edge-case-owner-unavailable.md)).

**Lifecycle**
- [ ] Statuses, `replaced_by`, effective dating defined; split/merge applicability decided.
- [ ] Type 2 consumers have a review process for flagged records.
- [ ] Erasure path defined, including a tracked purge job for logs, snapshots and backups ([decision-delete-policy-and-erasure](../decisions/decision-delete-policy-and-erasure.md)).

**Governance**
- [ ] Data dictionary entry and ADRs written ([data-dictionary-and-adr](data-dictionary-and-adr.md)).
- [ ] Monitoring defined ([freshness-and-drift-monitoring](freshness-and-drift-monitoring.md)).
