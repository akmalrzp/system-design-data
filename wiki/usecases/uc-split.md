---
type: usecase
title: "Use case: split"
tags: [proposal]
sources: [osdm-integration-summary]
created: 2026-10-09
updated: 2026-10-21
---

# Use case: split

**Applicability**: any entity where business rules allow one record to become several (a unit divided into two units). Decide per entity in the [data-dictionary-and-adr](../ops/data-dictionary-and-adr.md) entry.

**Owner steps**
1. Create the new records with new IDs.
2. Retire the original: status `inactive` (or `split`), `replaced_by = [new IDs]`, new version, `effective_to`.
3. Publish events for the retired and new records.

**Effect by module type**
- Owner: original kept as history; new rows.
- Type 1: cache shows the original as inactive and links to successors; records keep pointing to the original until reassigned, with a visible status.
- Type 2: **does not move records silently.** The records indexed by the original are flagged for review; a person decides how to distribute them (for example D splits a requirement). A, as holder of assignments, reassigns affected employees to the new units (rule-driven or manual).

**Rules**
- The split mapping is rarely one-to-one for children; the owner's `replaced_by` states possible successors, not the distribution.
- A consumer that stores quantities against the original (D's required headcount) needs a business decision, not an automatic rule.
- Closed or past records keep the original ID, so history stays true. Open records are moved by the module that owns them: it closes the old record with `effective_to` and creates a new effective-dated one on a successor, with `previous_*` fields in its event. It never edits the old record in place. UIs follow `replaced_by` to link to successors.

**Edge cases**: split in progress (successors created before the original is retired); partial reassignment; reversal (do not reuse the original; create a new record or [uc-merge](uc-merge.md)).
