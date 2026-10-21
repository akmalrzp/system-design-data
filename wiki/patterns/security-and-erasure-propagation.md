---
type: pattern
title: "Security and erasure propagation"
tags: [proposal,unverified]
sources: [osdm-integration-summary,api-design-for-system-integration]
created: 2026-10-09
updated: 2026-10-21
---

# Security and erasure propagation

- **Scope fields per consumer** so a copy holds only what it needs.
- **Validate the owner's payloads** instead of trusting them.
- **Inventory interfaces and versions.**
- **Idempotency keys** are high-entropy and cached under a composite key so one client cannot read another's cached response.
- **Erasure**: when the owner deletes or anonymises a person, the change must reach every consumer's copy, including caches, projections, backups and logs. The change carries a new version so version guards apply it; caches and projections take it through the apply path, and a tracked purge job covers logs, snapshots and backups. Applicable law (Indonesia's UU PDP, GDPR) was not researched; confirm with legal counsel.

Design hook: erasure is the one case where a hard delete or anonymisation is allowed ([decision-delete-policy-and-erasure](../decisions/decision-delete-policy-and-erasure.md)); [accountability-default](../tables/accountability-default.md) logs must be consistent with it.
