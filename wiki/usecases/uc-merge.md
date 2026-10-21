---
type: usecase
title: "Use case: merge"
tags: [proposal]
sources: [osdm-integration-summary]
created: 2026-10-09
updated: 2026-10-21
---

# Use case: merge

**Applicability**: any entity where business rules allow several records to become one (two units merged; duplicate records resolved). Decide per entity.

**Owner steps**
1. Choose the surviving record (existing or new ID).
2. Retire the others: status `inactive` (or `merged`), `replaced_by = [surviving ID]`, new version, `effective_to`.
3. Publish events for all affected records.

**Effect by module type**
- Owner: absorbed records remain as history.
- Type 1: cache marks absorbed records and links successor; owner's holders (A) reassign affected employees; consumer records are updated by their own module's process, not by repointing blindly.
- Type 2: flags the requirements of absorbed units; a person decides whether to add, replace or keep them. Never silently re-key.
- Projections: counts for absorbed units move after reassignments; show "as of".

**Rules**
- Closed or past records keep the absorbed ID, so history stays true. Open records are moved by the module that owns them: it closes the old record with `effective_to` and creates a new effective-dated one on the survivor, with `previous_*` fields in its event. It never edits the old record in place. UIs follow `replaced_by` to link to the survivor.
- Never reuse the absorbed IDs. Resolve references through `replaced_by` chains (keep chain length bounded; resolve transitively).
- A merge can be chained with later changes; consumers apply by version.

**Edge cases**: conflicting attribute values between merged records (owner decides); a merge reversed later (treat as a [uc-split](uc-split.md)); duplicates created in error in a consumer's own table (its own matter).
