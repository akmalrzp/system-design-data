---
type: table-type
title: "Crosswalk table"
tags: [proposal,unverified]
sources: [osdm-integration-summary]
created: 2026-10-09
updated: 2026-10-21
---

# Crosswalk table

Maps an external code or entry to an internal ID. Two uses:
- **External reference sets** where there is no stable code, or the organisation's definition differs from the external one ([external-reference-sets](../decisions/external-reference-sets.md)).
- **Legacy-system codes** for records migrated from a legacy system (for example an old HRIS unit code that appears in archived documents).

**Columns** (illustrative): `internal_id`, `external_authority` (authority or legacy system), `external_code`, `external_edition`, `valid_from`, `valid_to`, `mapping_type` (exact, broader, narrower), `approved_by`.

**When a crosswalk, when not**
- One legacy code per record, with no time validity: a plain attribute on the master is enough ([edge-case-business-key-vs-id](../modules/edge-case-business-key-vs-id.md)).
- Several legacy systems, or a mapping that changes over time: a crosswalk.
- A mapping only one consumer needs (for example B's bank branch codes per unit): that consumer's local extension table keyed by the owner's ID, not a crosswalk.

**Rules**
- Owned and published by the entity's owner (for external sets, the reference set owner), so every module resolves a code the same way; changes need steward approval. Consumers do not build their own crosswalk for a code that more than one module uses.
- Never delete a mapping: close it with `valid_to`.
- Keep both definitions when they differ and map explicitly.
