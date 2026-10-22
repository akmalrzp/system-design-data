**TL;DR: Integrating Independent OSDM Modules**

### Core Idea
Design independent modules that share reference data without coupling. Each entity has **exactly one owner** (system of record). Everyone else holds a derived, read-only copy. No shared databases, no cross-module foreign keys. Copies lag the owner, so design for eventual consistency and show freshness.

### 10 Core Principles
1. One owner per record.
2. Permanent opaque ID (immutable, never reused, no business meaning). Business keys such as `employee_number` are attributes, never join keys.
3. Consumers store only the ID + fields they need (read-only cache). A field that must be authoritative in a consumer is owned by that consumer.
4. Small, versioned, additive public contract, separate from the internal model.
5. Changes flow as events; a read API serves lookup, bootstrap and catch-up.
6. Assume duplicates, disorder and loss: apply idempotently with version guards.
7. Translate at the edge (anti-corruption layer).
8. No shared DBs or cross-module FKs.
9. Lifecycle changes are explicit versioned states, not absence.
10. Accept eventual consistency and show "as of" timestamps.

### Module Types (classified per entity, not per module)
| Type | Role | Example |
|------|------|---------|
| **Owner / Broadcaster** | Creates, changes, retires records; publishes events + read API; publishes only its own facts (other owners' entities go out as IDs only) | C owns Unit/Position; A owns Employee |
| **Consumer Type 1** (attributes) | Uses data as descriptive fields; low lifecycle impact | B (Payroll) shows unit/position names |
| **Consumer Type 2** (index) | Uses ID as business key of its own records; high lifecycle impact, effective-dated records | D stores "required headcount per unit" |
| **Hybrid** | Owns some entities, consumes others; owned tables and caches kept separate | A owns Employee, consumes Unit/Position |
| **Projection holder** | Derives facts from events (needs `previous_*` values); rebuildable; no full copies | D's unit -> active count |

Deciding boundaries (proposal): use the 8 **module scope criteria** (decision authority, accountability, change rate, lifecycle, sensitivity, consistency need, consumer pattern, team independence) and the **feature placement test**. A feature lives with the owner of the records it writes. It never writes another owner's records or reads its DB. An atomic write across two owners means redesign.

### Table Types
- **Master reference** (owner only): authoritative; `version` on every change; outbox row in the same transaction; status instead of delete.
- **Cache** (consumer): written only by the apply path; insert if missing, apply only newer versions; local extensions in a separate table.
- **Transaction**: the module's own records; hold foreign IDs validated against the local cache. Store a **snapshot** (value + source version) only when a document must reproduce the past (payslips, decision letters).
- **Projection**: rebuildable derived facts with `as_of`; never master data.
- **Crosswalk**: maps external or legacy codes to internal IDs; owned and published by the entity owner; closed with `valid_to`, never deleted.

**Accountability default** (proposal, no audit requirement yet): outbox + `version`/`updated_by` on masters; `source_version`, `last_synced_at`, processed-event log and reconciliation on caches; who/when/why + append-only log on transactions; rebuild-and-compare on projections; approval + validity dates on crosswalks.

**Naming**: `<Module>_<Entity>`; `<Module>_Ref<Name>` for reference sets; `<Module>_<Owner><Role>` for support tables (role words: `Level`, `Lineage`, `ChangeEvent`, `History`, `Crosswalk`).

### Communication Rules
- **Default flow**: owner writes business change + outbox in one transaction -> relay publishes event -> consumers apply idempotently. Read API (lookup by ID; cursor list by `updated_after`, sorted by (`updated_at`, `id`)) for bootstrap, catch-up and lookup.
- **Thin vs fat events**: fat for non-sensitive attributes; thin + authorised read for sensitive data (salary, national ID).
- **Event envelope**: `event_id`, `entity_type`/`entity_id`, `version`, `occurred_at`, `change_type`, `status`, `replaced_by`, `previous_*`, `effective_from`/`effective_to`, `schema_version`.
- **Key rules**:
  - Additive-only contracts: add optional fields, endpoints, enum values. Never remove, rename or retype a field, or add a required one.
  - Retire through deprecation and `Deprecation`/`Sunset` headers; remove only when usage reaches zero.
  - Contract testing default: spec diffing in CI, plus schema registry compatibility (FULL/FULL_TRANSITIVE) for events.
  - Version-guarded apply: no row = insert; newer = apply; equal or older = ignore. A bare `UPDATE ... WHERE` loses inserts.
  - Ordering per entity only (partition by ID + monotonic version).
  - Validate IDs against the local cache for **user writes**. Do not re-check an owner's events against your own cache of a third module.
  - Reconcile each cache against the owner, never against another consumer's copy.
  - Deletes = explicit status + version; `replaced_by` for merge/split. No hard deletes except legal erasure, or records created in error and never published.

### Common Use Cases (what happens)
- **Initial broadcast / bootstrap / late joiner**: record the event-stream position, page the owner's list API, apply through the same path, consume events from the recorded position, reconcile. Only then enable cache-based validation.
- **New record / update / rename / reparent**: owner increments version and publishes; consumers apply if newer. Reparent changes roll-ups.
- **Split / Merge**: owner creates successors or picks a survivor, and retires the rest with `replaced_by`. Closed records keep the old ID. Open records are moved by the module that owns them: close with `effective_to`, create a new one. Type 2 consumers **never auto-move** records: flag for human review.
- **Divergent caches** (A and D both cache C): a mismatch is normally lag. D's projection trusts A for who sits where and C for what the unit is.
- **Delete / Erasure**: retirement = status change. Erasure = anonymise in place (recommended) or hard delete, always with a **new version**. Caches take it through the apply path; a tracked purge job clears logs, outbox payloads, snapshots and backups; then reconcile.
- **External edition change**: steward approves; owner imports, closes changed entries, publishes with a cutoff date. Old codes stay resolvable.
- **Owner down**: consumers keep working from cache. Writes that need an unseen ID are rejected or held as pending.

### Operations Essentials
- Give every external reference set an internal owner and a data steward. Key it by stable code, code + edition, or minted ID + crosswalk.
- Write a data dictionary entry for every entity and an ADR for every design choice.
- Monitor lag, drift (via reconciliation), rejects and placeholders.
- Cache reference data locally. Call the owner on demand for sensitive or exactness-critical data, and on a cache miss.

### Design Checklist (quick)
- Classify every entity (owner / type 1 / type 2 / hybrid); apply scope criteria and the placement test.
- Acyclic dependency graph per entity.
- Every table typed, with accountability chosen.
- Owner provides: opaque IDs, versions, outbox, events, read API, additive contract, sensitive-field scoping.
- Consumer has: version-guarded apply path, anti-corruption layer, unknown-ID handling, bootstrap + reconciliation, owner-offline behaviour.
- Lifecycle states, `replaced_by` and effective dating defined; Type 2 review process for flagged records.
- Erasure path designed, including the purge job.
- Data dictionary, ADRs and monitoring in place.

**Bottom line**: Own once, publish a small additive contract via events + read API, let everyone else keep a version-guarded local cache, and treat lifecycle changes as explicit versioned states. Independence + eventual consistency by design.

## What is in the folder

- full-guide.md is the whole guide in one file, about 1,100 lines, with a table of contents. It is generated from the pages, so edit the pages, not this file
- wiki/ holds the atomic pages, plus index.md, log.md and SCHEMA.md. The page folders are sources, guide, concepts, modules, tables, patterns, decisions, usecases and ops.
- raw/ holds copies of two source files and a record of your answers in this chat.
- scripts/build_guide.py regenerates the index and the single-file guide after you edit pages.

## Things to check before relying on it

Pages tagged proposal are my suggestions, not from the sources. This includes the module scope criteria, the feature placement test, and the accountability default. Erasure law (UU PDP, GDPR) was not researched, so confirm it with legal.