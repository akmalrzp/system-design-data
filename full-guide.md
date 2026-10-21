# System design guide: integrating independent OSDM modules

Technology-neutral guide for architects and developers. Generated from the atomic pages in `wiki/` by `scripts/build_guide.py`; edit the pages, not this file.

## Contents

- [0. Purpose, scope and principles](#guide-00-overview)
  - [Problem statement](#problem-statement)
  - [Core principles](#principles)
  - [System of record](#system-of-record)
  - [Opaque permanent ID](#opaque-permanent-id)
  - [Anti-corruption layer](#anti-corruption-layer)
  - [Eventual consistency and freshness](#eventual-consistency)
  - [Glossary](#glossary)
  - [Worked example: modules A, B, C, D](#worked-example)
- [1. Module types](#guide-01-module-types)
  - [Module scope criteria](#module-scope-criteria)
  - [Feature placement test](#feature-placement-test)
  - [Data owner and broadcaster (source of truth)](#data-owner-broadcaster)
  - [Consumer type 1: reference data as attributes](#consumer-attribute-user)
  - [Consumer type 2: reference data as index](#consumer-index-user)
  - [Hybrid module (owner and consumer)](#hybrid-module)
  - [Projection holder](#projection-module)
  - [Owner of external reference sets](#external-reference-owner)
  - [Edge case: field ownership disputes](#edge-case-field-ownership)
  - [Edge case: sensitive fields](#edge-case-sensitive-fields)
  - [Edge case: hierarchy changes](#edge-case-hierarchy-changes)
  - [Edge case: circular dependencies](#edge-case-circular-dependencies)
  - [Edge case: owner unavailable](#edge-case-owner-unavailable)
  - [Edge case: business key vs ID](#edge-case-business-key-vs-id)
  - [Edge case: divergent reference caches](#edge-case-divergent-reference-caches)
- [2. Table types and accountability](#guide-02-table-types)
  - [Master reference table (owner)](#table-master-reference)
  - [Cache of master reference (consumer)](#table-master-cache)
  - [Transaction table](#table-transaction)
  - [Projection table](#table-projection)
  - [Crosswalk table](#table-crosswalk)
  - [Accountability options compared](#accountability-options)
  - [Accountability default (recommended)](#accountability-default)
  - [Table naming convention](#table-naming-convention)
- [3. Inter-system communication patterns](#guide-03-communication)
  - [Flow: events plus read API](#flow-events-plus-read-api)
  - [Flow: bootstrap and catch-up](#flow-bootstrap-and-catch-up)
  - [Flow: reconciliation](#flow-reconciliation)
  - [Payload: thin vs fat events](#payload-thin-vs-fat-events)
  - [Payload: event envelope](#payload-event-envelope)
  - [Payload: sensitive fields](#payload-sensitive-fields)
  - [Technology requirements (neutral)](#technology-neutral-requirements)
  - [Rule: no shared databases](#rule-no-shared-database)
  - [Rule: additive-only contract](#rule-additive-only-contract)
  - [Rule: idempotent, version-guarded apply](#rule-idempotent-versioned-apply)
  - [Rule: ordering per entity](#rule-ordering-per-entity)
  - [Rule: validate IDs against the local cache](#rule-validate-ids-locally)
  - [Rule: deletes are explicit versioned states](#rule-deletes-as-state)
  - [Rule: deprecation and sunset](#rule-deprecation-and-sunset)
  - [Rule: contract testing](#rule-contract-testing)
  - [Security and erasure propagation](#security-and-erasure-propagation)
- [4. Use cases](#guide-04-use-cases)
  - [Use case: initial master broadcast to empty caches](#uc-initial-broadcast)
  - [Use case: new record](#uc-new-record)
  - [Use case: update, rename, reparent](#uc-update-rename-reparent)
  - [Use case: split](#uc-split)
  - [Use case: merge](#uc-merge)
  - [Use case: delete](#uc-delete)
  - [Use case: late joiner, missed or out-of-order events](#uc-late-joiner-and-gaps)
  - [Use case: external reference edition change](#uc-external-edition-change)
- [5. Operations and governance](#guide-05-operations)
  - [External reference sets](#external-reference-sets)
  - [Data dictionary and ADRs](#data-dictionary-and-adr)
  - [Freshness and drift monitoring](#freshness-and-drift-monitoring)
- [6. Appendices: decisions, checklist, reliability](#guide-06-appendices)
  - [Decision: subscribe to events or call the owner on demand](#decision-subscribe-vs-call-on-demand)
  - [Decision: delete policy and the erasure exception](#decision-delete-policy-and-erasure)
  - [Decision: contract versioning style](#decision-contract-versioning-style)
  - [Design checklist for a new module](#design-checklist-new-module)
  - [Source reliability](#source-reliability)

<a id="guide-00-overview"></a>

# 0. Purpose, scope and principles

This guide explains how to design independent modules that share reference data, for the Organisasi & SDM domain. It is technology-neutral and written for architects and developers. Modules A, B, C, D are examples ([Worked example: modules A, B, C, D](#worked-example)).

Reading order: this section first; then module types, tables, communication, use cases; operations and appendices as reference. The sections are ordered so that each depends only on earlier ones (communication patterns come before the use cases that use them).

Sections: [1. Module types](#guide-01-module-types), [2. Table types and accountability](#guide-02-table-types), [3. Inter-system communication patterns](#guide-03-communication), [4. Use cases](#guide-04-use-cases), [5. Operations and governance](#guide-05-operations), [6. Appendices: decisions, checklist, reliability](#guide-06-appendices).

Mapping to the author's starting outline: its section 1 is guide section 1, its 2 is 2, its 4 (communication) is now section 3, and its 3 (use cases) is now section 4.

Evidence is uneven; see [Source reliability](#source-reliability). Pages tagged `proposal` are reasoned recommendations.

<a id="problem-statement"></a>

## Problem statement

System A produces data A1 and system B uses it. Both must keep the same reference to A1 while staying independent: separate release cycles, separate databases, and tolerance for the other being down or changing.

The guide answers four questions:
1. What kinds of modules exist, and where does each feature belong? See [1. Module types](#guide-01-module-types).
2. What kinds of tables exist, and how is each one made accountable? See [2. Table types and accountability](#guide-02-table-types).
3. How do modules talk without becoming coupled? See [3. Inter-system communication patterns](#guide-03-communication).
4. What happens to each module and table in common situations (bootstrap, new record, split, merge, delete)? See [4. Use cases](#guide-04-use-cases).

The answer rests on [Core principles](#principles). Vocabulary is in [Glossary](#glossary). Examples use [Worked example: modules A, B, C, D](#worked-example).

<a id="principles"></a>

## Core principles

1. **One owner per record.** Exactly one module creates and changes a record: the [System of record](#system-of-record). Everyone else holds a derived copy.
2. **Permanent opaque ID.** The owner issues an identifier that never changes, is never reused, and carries no business meaning. See [Opaque permanent ID](#opaque-permanent-id).
3. **Consumers store the ID plus only the fields they need**, as a read-only cache. Fields that must be authoritative in a consumer belong to that consumer, not to the owner.
4. **A small, versioned, additive contract**, separate from the owner's internal model. See [Rule: additive-only contract](#rule-additive-only-contract).
5. **Changes flow as events; a read API serves lookup and catch-up.** See [Flow: events plus read API](#flow-events-plus-read-api).
6. **Assume duplicates, disorder, and loss.** Apply changes idempotently and only when the version is newer. See [Rule: idempotent, version-guarded apply](#rule-idempotent-versioned-apply).
7. **Translate at the edge.** Each consumer converts the owner's model into its own. See [Anti-corruption layer](#anti-corruption-layer).
8. **No shared databases, no cross-module foreign keys.**
9. **State changes, not absence.** Lifecycle changes (inactive, replaced, deleted) are explicit versioned states. See [Rule: deletes are explicit versioned states](#rule-deletes-as-state).
10. **Accept eventual consistency** and show freshness. See [Eventual consistency and freshness](#eventual-consistency).

Evidence: items 1, 2, 3, 7 are well-established practice but not source-verified (see [Source reliability](#source-reliability)).

<a id="system-of-record"></a>

## System of record

For each entity exactly one module is the system of record: the only place where the entity is created, changed, retired. All other copies are derived and read-only.

Consequences:
- Ownership is decided once per entity, and recorded in the data dictionary ([Data dictionary and ADRs](#data-dictionary-and-adr)).
- A consumer that needs an authoritative field of its own owns that field in its own table, rather than editing the owner's copy.
- Field-level disputes: see [Edge case: field ownership disputes](#edge-case-field-ownership).

Related: [Data owner and broadcaster (source of truth)](#data-owner-broadcaster), [Opaque permanent ID](#opaque-permanent-id). Evidence: established practice, not source-verified.

<a id="opaque-permanent-id"></a>

## Opaque permanent ID

The owner issues an identifier at record creation. Rules:
- **Immutable**: never changes.
- **Never reused**: not even after the record is deleted.
- **Opaque**: no name, unit, date, or national-ID content, so it cannot go stale or leak.
- **Not a business key**: a human-facing number (for example `employee_number`) is an ordinary attribute and is not the join key between modules.
- **Global enough** to be unique across owners (UUID or equivalent). The format is a technology choice.

Consumers store it as a foreign reference with a unique constraint in their own cache or transaction table.

Evidence: established practice, not source-verified (see [Source reliability](#source-reliability)).

<a id="anti-corruption-layer"></a>

## Anti-corruption layer

An adapter at the consumer's edge converts the owner's published shape into the consumer's internal model. Only the adapter knows the owner's vocabulary, so a contract change touches one place.

Practical rules:
- Parse and validate in one place.
- Ignore unknown additive fields and unknown enum values; reject malformed or missing required data (see [Rule: additive-only contract](#rule-additive-only-contract)).
- Never expose the owner's types inside the consumer's domain code.

The alternative, one enterprise-wide canonical model that every module must follow, couples all modules to one change process. This guide prefers a small published model per owner. Evidence: not source-verified (see [Source reliability](#source-reliability)).

<a id="eventual-consistency"></a>

## Eventual consistency and freshness

Copies in consumers lag the owner. Design for it:
- Show an "as of" timestamp wherever a consumer displays cached or projected data (counts, names).
- Validate against the local cache, not a live call, unless exactness at decision time is required (see [Decision: subscribe to events or call the owner on demand](#decision-subscribe-vs-call-on-demand)).
- Detect and repair drift with [Flow: reconciliation](#flow-reconciliation) and [Freshness and drift monitoring](#freshness-and-drift-monitoring).
- A consumer must tolerate an unknown ID: store it, show a placeholder, fetch from the owner on the miss.

<a id="glossary"></a>

## Glossary

- **Module**: an independently deployable system with its own database and release cycle (A, B, C, D in examples).
- **Data owner / system of record**: the module allowed to create and change a record. See [System of record](#system-of-record).
- **Broadcaster**: an owner in its role of publishing change events and exposing a read API.
- **Consumer**: a module that holds references to or copies of another module's data.
- **Reference data**: data owned by one module and referred to by others (unit, position, region code).
- **Master reference table**: the owner's authoritative table. See [Master reference table (owner)](#table-master-reference).
- **Cache (of master)**: a consumer's read-only local copy. See [Cache of master reference (consumer)](#table-master-cache).
- **Transaction table**: a consumer's or owner's own business records that reference IDs. See [Transaction table](#table-transaction).
- **Projection**: derived data (for example counts per unit) built from events, not copied records. See [Projection table](#table-projection).
- **Crosswalk**: mapping between internal IDs and external or legacy-system codes. See [Crosswalk table](#table-crosswalk).
- **Contract**: the published, versioned shape of API responses and events.
- **Event envelope**: metadata fields on every event. See [Payload: event envelope](#payload-event-envelope).
- **Version**: a monotonic number per record, increased on every change.
- **Eventual consistency**: copies converge with the owner once changes stop and missed events are repaired by reconciliation; not a guarantee on its own. See [Eventual consistency and freshness](#eventual-consistency).
- **Tombstone**: a retained marker that a record is deleted or inactive; here a versioned state, not an absence.
- **`replaced_by`**: pointer from a retired record to its successor(s) after merge or split.
- **Effective dating**: `effective_from` / `effective_to` on records whose meaning depends on a date.
- **Anti-corruption layer**: adapter translating an external model into the local one. See [Anti-corruption layer](#anti-corruption-layer).
- **Data steward**: named person accountable for a reference set. See [External reference sets](#external-reference-sets).
- **Outbox**: a table where the owner writes a change event in the same transaction as the business change; a relay publishes it later.
- **Idempotent**: safe to apply more than once; the result is the same as applying it once.
- **Version-guarded apply**: only apply an event when its version is newer than the last stored version.
- **Reconciliation**: scheduled comparison between a consumer copy and the owner, followed by repair.
- **Bootstrap**: filling an empty cache from the owner’s read API or replay of current state.
- **Catch-up**: fetching changes missed since a cursor or last event.
- **Cursor pagination**: paging through a list using an opaque token rather than page numbers.
- **Optimistic concurrency**: preventing lost updates by checking a version or ETag before writing.
- **ETag / `If-Match`**: HTTP headers used for optimistic concurrency; a mismatch returns 412.
- **Schema registry**: service that stores and enforces event schema compatibility.
- **FULL / FULL_TRANSITIVE**: schema compatibility modes that allow either side to upgrade first.
- **Thin event**: an event with only the identifier and change type; the consumer fetches details on demand.
- **Fat event**: an event containing the data values the consumer needs without a follow-up call.
- **Sensitive fields**: data such as salary or national identifiers that are not broadcast by default.
- **Drift**: a lasting mismatch between a consumer copy and the owner.
- **Lag**: the delay between an owner change and the consumer copy being updated.
- **Placeholder**: a temporary local record for an unknown ID, resolved later.
- **Snapshot**: a stored copy of referenced values so a document or report can reproduce the past.
- **Lineage columns**: fields such as `source_version`, `updated_at`, and `last_synced_at` that show where a row came from.
- **Processed-event log**: a consumer record of which event IDs or versions were already applied.
- **Bitemporal history**: history tracking both valid time and recorded time.
- **Event store**: a system where events are the source of truth.
- **Saga**: a coordinated sequence of local transactions with compensating actions.
- **Partition key**: the key used to keep related events in order, usually by entity ID.
- **At-least-once delivery**: a message may be delivered more than once.
- **Exactly-once delivery**: a message is processed once; this is not assumed across separate systems.
- **Tolerant reader**: a consumer that ignores unknown fields and enum values.
- **Deprecation / sunset**: headers or markers warning that a contract field or endpoint will be removed.
- **Consumer-driven contract testing**: consumers record the responses they depend on and replay them against the owner.
- **Bi-directional contract testing**: comparing the owner’s contract spec with the consumers’ recorded usage.
- **Erasure**: removing or anonymising personal data when legally required.
- **Anonymisation**: replacing personal data with nulls or placeholders.
- **Data dictionary**: a record of each entity’s owner, fields, lifecycle, consumers, and contract.
- **ADR**: architecture decision record, documenting context, options, decision, consequences, and date.
- **SLO**: service-level objective, such as maximum acceptable lag.
- **Dead-letter queue**: holding area for events that cannot be processed.
- **Circuit breaker**: a pattern that stops calls to a failing dependency.
- **mTLS / OAuth**: security mechanisms for service identity and authorisation.
- **Business key**: a human-facing identifier such as employee number or unit code.
- **Hybrid module**: a module that owns some entities and consumes others.
- **External reference set**: reference data coming from outside the organisation, such as region or classification codes.
- **OSDM**: Organisasi & SDM, the Organisation & Human Resources domain used in this guide.
- **Transactional outbox**: the pattern of writing business changes and outgoing event records in one database transaction.
- **Read API**: endpoint used for lookup, bootstrap, and catch-up of current state.
- **Attribute user**: a consumer that uses another module’s data as descriptive labels rather than structural definitions.
- **Index user**: a consumer that uses another module’s ID as a business key or classification axis.
- **Projection holder**: a module that computes aggregate views from events without copying full master records.
- **Processed event log**: a consumer's durable record of the event IDs or versions it has already applied.

<a id="worked-example"></a>

## Worked example: modules A, B, C, D

The modules are illustrative. Replace them with real modules when applying the guide.

| Module | Role | Owns | Uses |
|---|---|---|---|
| A (HR Core) | owner, and consumer of C | Employee, assignment of an employee to a unit and position | Unit, Position from C |
| B (Payroll) | consumer type 1 (attributes) | pay grade, tax status | Employee from A; Unit, Position names from C |
| C (Org structure) | owner | Unit, Position | none |
| D (Requirements) | consumer type 2 (index) and projection holder | required headcount per unit | Unit from C; employee counts per unit from A |

Pattern summary:
- A broadcasts `employee_id`, `unit_id`, `position_id`, not names. Names belong to C.
- C broadcasts `unit.changed`, `position.changed`. A, B, D keep local caches.
- B resolves names by joining on IDs in its own cache of C.
- D uses `unit_id` as the business key of its requirement records, so unit lifecycle changes in C affect D directly.

Roles are defined in [1. Module types](#guide-01-module-types). Sensitive fields (salary, national ID) stay out of events; see [Payload: sensitive fields](#payload-sensitive-fields).

<a id="guide-01-module-types"></a>

# 1. Module types

Modules are classified per entity by their relationship to the data: owner, consumer using data as attributes, consumer using data as an index. A real module is usually several at once ([Hybrid module (owner and consumer)](#hybrid-module)). This section also gives measurements for scoping modules and a test for placing features, then lists special cases.

<a id="module-scope-criteria"></a>

## Module scope criteria

**Proposal** (assistant's suggestion, not from a source; revise to fit the organisation). Use these measurements to decide where a module boundary falls.

| # | Criterion | Question | Boundary signal |
|---|---|---|---|
| 1 | Decision authority | Who decides the value of this data? | Different decision-makers mean different modules |
| 2 | Accountability | Who answers for its correctness and audit? | One accountable role per entity |
| 3 | Rate and reason of change | Does it change for the same reasons and at the same pace as its neighbours? | Different change drivers mean different modules |
| 4 | Lifecycle | Is it created, retired and archived on the same schedule? | Different lifecycles mean different modules |
| 5 | Access and sensitivity | Is the audience or authorisation the same? | Different confidentiality means a separate module or separate fields |
| 6 | Consistency need | Must the data change atomically with its neighbours? | Atomic need means same module |
| 7 | Consumer pattern | Do consumers need it live, or can they work from a lagged copy? | Lag-tolerant data can live elsewhere |
| 8 | Team and release independence | Can one team release it without others? | Separate teams suggest separate modules |

Rules of thumb:
- Entities that must change in one transaction belong together (criterion 6).
- If two entities differ on criteria 1, 2 and 5, split them.
- If a boundary forces constant synchronous calls to work, it is probably drawn wrongly.

See [Feature placement test](#feature-placement-test) for assigning single features.

<a id="feature-placement-test"></a>

## Feature placement test

**Proposal.** Apply in order to decide whether a feature belongs in a given module. Stop at the first decisive answer.

1. **Does the feature create or change a record that this module owns?** If yes, it belongs here (owner).
2. **Does it create or change a record owned by another module?** Then it does not belong here. Call the owner's contract or ask the owner to add a capability.
3. **Does it only read another module's data to display or validate?** It belongs here, using the local cache ([Cache of master reference (consumer)](#table-master-cache)).
4. **Does it need a value that must be authoritative here but derived from elsewhere?** Make it a projection ([Projection table](#table-projection)) or own the field here.
5. **Does it need data the owner has not published?** Request a contract addition; do not read the owner's database.
6. **Does it mix data from two owners in one atomic write?** Redesign: separate each write per owner, coordinated by a compensating process, or move the entities into one module.

Record the result in the [Data dictionary and ADRs](#data-dictionary-and-adr) entry for the entity. Scope criteria: [Module scope criteria](#module-scope-criteria).

<a id="data-owner-broadcaster"></a>

## Data owner and broadcaster (source of truth)

**Role.** Creates, changes and retires records of its entities, issues their IDs ([Opaque permanent ID](#opaque-permanent-id)), and publishes changes.

**Must provide**
- Master reference tables ([Master reference table (owner)](#table-master-reference)).
- A change feed: versioned events through a transactional outbox ([Flow: events plus read API](#flow-events-plus-read-api)).
- A read API for lookup by ID and cursor-paginated catch-up by `updated_after`, sorted by (`updated_at`, `id`) ([Flow: bootstrap and catch-up](#flow-bootstrap-and-catch-up)).
- A crosswalk from legacy-system codes to its IDs, when records were migrated from a legacy system ([Crosswalk table](#table-crosswalk)).
- A published, versioned, additive contract ([Rule: additive-only contract](#rule-additive-only-contract)).
- Lifecycle states instead of hard delete ([Rule: deletes are explicit versioned states](#rule-deletes-as-state)), with `replaced_by` for merge and split.
- Field scoping per consumer for sensitive data ([Payload: sensitive fields](#payload-sensitive-fields)).

**Must not**
- Publish data it does not own (A publishes IDs of units, not unit names).
- Expose its internal model or database.
- Reuse or change an ID.

**Edge cases**: [Edge case: field ownership disputes](#edge-case-field-ownership), [Edge case: hierarchy changes](#edge-case-hierarchy-changes), [Edge case: owner unavailable](#edge-case-owner-unavailable), [Edge case: business key vs ID](#edge-case-business-key-vs-id).

Example: C (Unit, Position), and A for Employee ([Worked example: modules A, B, C, D](#worked-example)).

<a id="consumer-attribute-user"></a>

## Consumer type 1: reference data as attributes

**Role.** Uses another module's records as descriptive attributes of its own records. The reference does not define the consumer's records.

Example: B (Payroll) stores `employee_id` and shows the unit and position name. The payroll record exists with or without a particular unit.

**Design**
- Keep a cache of the master ([Cache of master reference (consumer)](#table-master-cache)) with the ID as unique key and only the fields needed.
- Own records ([Transaction table](#table-transaction)) hold the foreign ID; no cross-system foreign key constraint.
- Resolve names by joining on IDs in the local cache.
- Unknown ID: store it, show a placeholder, fetch from the owner on the miss.
- Inactive or replaced reference: keep the record, display the status, follow `replaced_by` for display.

**Impact of owner lifecycle**: low. Renames change display only; retirement changes display and flags.

Contrast with [Consumer type 2: reference data as index](#consumer-index-user). Use cases: [4. Use cases](#guide-04-use-cases).

<a id="consumer-index-user"></a>

## Consumer type 2: reference data as index

**Role.** Uses another module's ID as the business key or classification axis of its own records. The reference defines what the consumer's records mean.

Example: D stores "required employees per unit". `unit_id` identifies the requirement.

**Design (same cache pattern as type 1, plus)**
- Validate the referenced entity exists and is active before accepting a record.
- Effective-date records (`effective_from`, `effective_to`) so they stay meaningful after the reference changes.
- Treat owner lifecycle events as business events:
  - Inactive: flag the record for review or close it with `effective_to`.
  - Merge or split: flag the records and let a person decide. Never move them silently. See [Use case: merge](#uc-merge), [Use case: split](#uc-split).
  - Hierarchy change: roll-up totals change; read `parent_unit_id` from the local cache ([Edge case: hierarchy changes](#edge-case-hierarchy-changes)).
- If it needs derived data from another owner (headcount per unit from A), keep a projection, not copies ([Projection table](#table-projection)).
- Show "as of" timestamps on projections ([Eventual consistency and freshness](#eventual-consistency)).
- If another module emits the events the projection counts, its cache of the owner may differ from yours: [Edge case: divergent reference caches](#edge-case-divergent-reference-caches).

**Impact of owner lifecycle**: high. Agree the lifecycle contract with the owner early.

Contrast with [Consumer type 1: reference data as attributes](#consumer-attribute-user).

<a id="hybrid-module"></a>

## Hybrid module (owner and consumer)

Most real modules own some entities and consume others. A owns Employee and consumes Unit and Position from C ([Worked example: modules A, B, C, D](#worked-example)).

Rules:
- Classify per entity, not per module. A is an owner for Employee and a type 1 or 2 consumer for Unit.
- Keep owned tables and caches physically and logically separate ([Master reference table (owner)](#table-master-reference) vs [Cache of master reference (consumer)](#table-master-cache)).
- Validate an assignment (employee to unit and position) against the local cache before accepting it.
- Never republish a consumed entity's attributes (A publishes `unit_id`, not the unit's name).
- Watch for dependency cycles ([Edge case: circular dependencies](#edge-case-circular-dependencies)).

<a id="projection-module"></a>

## Projection holder

A module that needs facts derived from another owner's data, not the records themselves. Example: D needs active headcount per unit; it subscribes to A and keeps `unit_id -> active count`, not employee records.

Rules:
- The events must carry what the projection needs, including previous values (a transfer event includes `previous_unit_id`, so the old unit is decremented).
- The projection must be rebuildable from the owner's events or API ([Projection table](#table-projection)).
- Display "as of" timestamps.
- Do not reject the owner's events against your own cache of a third module; see [Edge case: divergent reference caches](#edge-case-divergent-reference-caches).
- Alternative: call the owner's read API when exactness at read time matters. See [Decision: subscribe to events or call the owner on demand](#decision-subscribe-vs-call-on-demand).

<a id="external-reference-owner"></a>

## Owner of external reference sets

Reference sets that come from outside the organisation (region codes, tax status, education levels) have no inter-system owner. Assign one internal module as owner and a named data steward ([External reference sets](#external-reference-sets)).

The owner:
- imports from the external authority and validates;
- publishes versioned events or a dataset plus a read API, exactly like any [Data owner and broadcaster (source of truth)](#data-owner-broadcaster);
- keeps old codes resolvable and never hard-deletes retired entries;
- announces edition changes with a cutoff date ([Use case: external reference edition change](#uc-external-edition-change)).

Consumers reference entries by code or ID, never by text.

<a id="edge-case-field-ownership"></a>

## Edge case: field ownership disputes

Situation: consumer B needs a field to be authoritative (for example a corrected display name) but A owns the record.

Resolution, in order:
1. Ask whether B really needs authority. If not, B uses A's value.
2. If B needs its own value, B owns that field in its own table, keyed by the same ID. It is not added to A's contract.
3. If both are legitimately needed, name them differently (`legal_name` vs `payroll_display_name`). One name must never carry two meanings.

Related: [System of record](#system-of-record), [Feature placement test](#feature-placement-test).

<a id="edge-case-sensitive-fields"></a>

## Edge case: sensitive fields

Salary, national ID and similar fields are not broadcast. The event carries non-sensitive fields; a consumer with a need reads the sensitive field from an authorised endpoint that checks authorisation on every read.

Consequences: the consumer cannot hold a cached copy by default; if it must, the copy is scoped, justified, and covered by [Security and erasure propagation](#security-and-erasure-propagation).

See [Payload: sensitive fields](#payload-sensitive-fields).

<a id="edge-case-hierarchy-changes"></a>

## Edge case: hierarchy changes

Reparenting a unit in C changes roll-ups in every consumer that aggregates by hierarchy (D's totals).

Rules:
- C publishes `parent_unit_id` and a version; a reparent is a normal update ([Use case: update, rename, reparent](#uc-update-rename-reparent)).
- Consumers read the hierarchy from their local cache and recompute roll-ups; they do not store roll-ups as master data.
- Effective-date the change if history matters (totals "as of" a past date need the parent as of that date).
- Cycles in the hierarchy are rejected by the owner.

<a id="edge-case-circular-dependencies"></a>

## Edge case: circular dependencies

A consumes C; C must not consume A's entity in a way that A needs to start. Circular data dependencies prevent bootstrap and make ordering undecidable.

Rules:
- Draw the dependency graph of entities (not modules). It must be acyclic per entity.
- If two modules each need the other's data, move the shared data to a third owner, or use IDs only on one side with lazy resolution.
- Bootstrap in dependency order ([Use case: initial master broadcast to empty caches](#uc-initial-broadcast)).

<a id="edge-case-owner-unavailable"></a>

## Edge case: owner unavailable

A consumer working from its local cache continues to operate; it cannot create new references to IDs it has never seen. A consumer calling the owner on demand fails with the owner.

Rules:
- Prefer local cache for validation and display.
- Queue events for later application ([Rule: idempotent, version-guarded apply](#rule-idempotent-versioned-apply)).
- For writes that need an unseen reference, either reject with a clear error or accept as pending and re-validate later; decide per business rule.
- Document the owner's availability expectation in the contract.

See [Decision: subscribe to events or call the owner on demand](#decision-subscribe-vs-call-on-demand).

<a id="edge-case-business-key-vs-id"></a>

## Edge case: business key vs ID

Business keys (employee number, unit code) are human-meaningful and may be reissued, reformatted, or corrected. IDs ([Opaque permanent ID](#opaque-permanent-id)) are not.

Rules:
- Join on IDs. Expose business keys as attributes only.
- Enforce uniqueness of the business key in the owner, with effective dating if it can be reused over time.
- Legacy-system codes are attributes too; use a [Crosswalk table](#table-crosswalk) in the owner when there are several legacy systems or the mapping changes over time.
- External standard codes may be the key for external reference sets under the conditions in [External reference sets](#external-reference-sets).

<a id="edge-case-divergent-reference-caches"></a>

## Edge case: divergent reference caches

Two consumers cache the same owner's entity, and one of them emits events that the other interprets through its own copy. They can disagree for a while.

Example: A and D both subscribe to C's units. A validates assignments against its cache of C and emits `employee.changed` with `unit_id`. D keeps a headcount projection `unit_id -> active count` ([Projection holder](#projection-module)) and reads unit status and `parent_unit_id` from its own cache of C.

This page is design reasoning, not from a cited source.

### Who is authoritative
- C is authoritative for what a unit is (exists, active, name, parent).
- A is authoritative for which unit an employee is assigned to.
- Neither A's nor D's cache of C is authoritative. Both are read-only copies ([Cache of master reference (consumer)](#table-master-cache)).
- A does not send unit data to D. It sends `unit_id` only. D builds the projection itself.

Both caches come from the same versioned stream, so a mismatch is normally lag: one copy is behind. Lasting divergence means a cache missed events.

### Case 1: A's cache is ahead of D's
C creates a unit. A learns of it first, accepts an assignment, and emits an event with a `unit_id` that D does not know yet.

- D applies the count, keyed by `unit_id`. It does not reject A's event against its own cache: A already validated it. [Rule: validate IDs against the local cache](#rule-validate-ids-locally) applies to writes D accepts from its users, not to events from an owner.
- D shows a placeholder for the unit and calls C's read API on the miss.
- Roll-ups cannot place the count until D has the unit's `parent_unit_id` ([Edge case: hierarchy changes](#edge-case-hierarchy-changes)).

### Case 2: D's cache is ahead of A's
C marks a unit `inactive` or sets `replaced_by`. D learns of it first. A, still on the older copy, keeps assigning to the old unit.

- D's count shows employees on an inactive unit. That matches what A holds, so the count is correct; the assignments are what need correcting.
- D flags the unit. It does not move counts via `replaced_by` itself ([Use case: merge](#uc-merge), [Use case: split](#uc-split)).
- When A reassigns, its events carry `previous_unit_id` and the count moves.
- If A keeps assigning to the unit beyond normal lag, A's cache is stale or A is not processing C's lifecycle events. Alert on it ([Freshness and drift monitoring](#freshness-and-drift-monitoring)).

### Case 3: lasting divergence
A missed event or a bug leaves a cache wrong indefinitely.

- A and D each reconcile their cache against C ([Flow: reconciliation](#flow-reconciliation)). Never reconcile one consumer's cache against another's.
- D also reconciles its projection against A per unit (A's headcount read API, or a recount by range).

### Optional: carry the version A validated against
A adds the unit version it validated against (for example `unit_version`) to its assignment event. D compares it with its own cache:
- A's version is newer: D re-fetches the unit from C.
- D's version is newer: A acted on an older view; D can flag the record.

This lets D tell lag from a real conflict without guessing. It is an additive field ([Rule: additive-only contract](#rule-additive-only-contract)).

### Rule
The projection trusts A for who sits where and C for what the unit is. D never checks A's facts against its own cache of C.

<a id="guide-02-table-types"></a>

# 2. Table types and accountability

Each table type has different accountability needs. The comparison of mechanisms is followed by a recommended default; no audit requirement exists yet, so the default is revisable.

<a id="table-master-reference"></a>

## Master reference table (owner)

The authoritative table in the [Data owner and broadcaster (source of truth)](#data-owner-broadcaster).

**Typical columns** (illustrative)
- `id` (opaque permanent), business key as attribute, `version` (monotonic), `status` (`active`, `inactive`, plus domain states), `replaced_by` (list), `effective_from`, `effective_to`, `updated_at`.
- Attributes owned by this module.
- Parent references for hierarchies (`parent_id`).

**Rules**
- Only the owner writes.
- Every change increments `version`.
- Retirement is a status, not a delete (exception: [Decision: delete policy and the erasure exception](#decision-delete-policy-and-erasure)).
- Every change writes an outbox record in the same transaction ([Flow: events plus read API](#flow-events-plus-read-api)).

**Accountability**: transaction log or outbox for every change, plus history where past values matter. See [Accountability options compared](#accountability-options) and [Accountability default (recommended)](#accountability-default).

<a id="table-master-cache"></a>

## Cache of master reference (consumer)

A consumer's read-only local copy of an owner's records.

**Typical columns**
- Local primary key, `source_id` (the owner's ID, unique), `source_version`, only the cached fields needed, `status`, `replaced_by`, `last_synced_at`.

**Rules**
- Written only by the apply path ([Rule: idempotent, version-guarded apply](#rule-idempotent-versioned-apply)); never edited by users or other processes.
- Apply only if event version is greater than `source_version`; insert when no row exists.
- Stores deletion and inactive states, not absence ([Rule: deletes are explicit versioned states](#rule-deletes-as-state)).
- Unknown IDs: placeholder row or lazy fetch. A placeholder row has no `source_version`, so the first real event or fetch always fills it.
- Local extensions go in a separate table keyed by the owner's ID, not in the cache.

**Accountability**: processed-event log (event ID or entity ID plus version, committed in the same transaction as the change) and reconciliation results ([Flow: reconciliation](#flow-reconciliation)). Lineage columns `source_version` and `last_synced_at` show which owner state the copy reflects.

<a id="table-transaction"></a>

## Transaction table

A module's own business records: assignments, requirements, payroll runs. They reference IDs from caches or from their own master tables.

**Rules**
- Hold foreign IDs, not copied descriptive text, unless the record must be reproducible as it was (then store a snapshot; see below).
- No cross-system foreign key constraint. Validate against the local cache on write ([Rule: validate IDs against the local cache](#rule-validate-ids-locally)).
- Effective-date records whose meaning depends on time (`effective_from`, `effective_to`).
- Append-only or versioned for records that must be auditable.

**Snapshot or reference?** Store the reference when the current name is wanted; store a snapshot (value plus the source version) when a legal or business need requires the document to reproduce what was true at the time (payslips, decision letters). Snapshots are in erasure scope ([Decision: delete policy and the erasure exception](#decision-delete-policy-and-erasure)).

**Accountability**: record who, when and why for each write; keep an append-only log when the record is a business fact (for example an assignment). See [Accountability options compared](#accountability-options).

<a id="table-projection"></a>

## Projection table

Derived facts from another owner's events, for example `unit_id -> active employee count` in D ([Projection holder](#projection-module)).

**Rules**
- Rebuildable from the owner's events or API (full rebuild path documented and tested).
- Updated by the same idempotent apply path as caches; events must carry previous values where needed (for example `previous_unit_id`).
- Carries `as_of` timestamp and source versions processed.
- Never treated as master data.

**Accountability**: rebuild-and-compare against the owner is the audit ([Flow: reconciliation](#flow-reconciliation)); the processed-event log shows what was applied.

<a id="table-crosswalk"></a>

## Crosswalk table

Maps an external code or entry to an internal ID. Two uses:
- **External reference sets** where there is no stable code, or the organisation's definition differs from the external one ([External reference sets](#external-reference-sets)).
- **Legacy-system codes** for records migrated from a legacy system (for example an old HRIS unit code that appears in archived documents).

**Columns** (illustrative): `internal_id`, `external_authority` (authority or legacy system), `external_code`, `external_edition`, `valid_from`, `valid_to`, `mapping_type` (exact, broader, narrower), `approved_by`.

**When a crosswalk, when not**
- One legacy code per record, with no time validity: a plain attribute on the master is enough ([Edge case: business key vs ID](#edge-case-business-key-vs-id)).
- Several legacy systems, or a mapping that changes over time: a crosswalk.
- A mapping only one consumer needs (for example B's bank branch codes per unit): that consumer's local extension table keyed by the owner's ID, not a crosswalk.

**Rules**
- Owned and published by the entity's owner (for external sets, the reference set owner), so every module resolves a code the same way; changes need steward approval. Consumers do not build their own crosswalk for a code that more than one module uses.
- Never delete a mapping: close it with `valid_to`.
- Keep both definitions when they differ and map explicitly.

<a id="accountability-options"></a>

## Accountability options compared

How to make every table type answerable for "who changed what, when, and why". No audit requirement exists yet (*Author decisions and scope, 2026-10-09* (`raw/author-decisions-2026-10-09.md`)); this page compares the options. The recommended default is in [Accountability default (recommended)](#accountability-default).

| Option | What it is | Best for | Cost | Weakness |
|---|---|---|---|---|
| Lineage columns | `version`, `source_version`, `updated_by`, `updated_at`, `last_synced_at` on rows | Caches, quick traceability | Very low | Shows state only, not history |
| Transaction log / outbox | Append-only record of every change, written in the same transaction | Owner master tables | Low to medium | Needs retention and indexing |
| Processed-event log | Consumer records event ID (or entity ID plus version) applied | Caches, projections | Low | Grows; needs pruning policy |
| Versioned history table | Prior row versions with validity intervals | Entities where past values matter (assignment, unit name) | Medium | Query complexity |
| Bitemporal history | Both valid time and recorded time | Retroactive corrections with legal weight | High | Complexity; use only if required |
| Event store | Events are the source of truth | Rebuild-heavy projections | High | Whole-system commitment |
| Snapshot on write | Copy of referenced values stored with the transaction | Documents that must reproduce the past | Low | Duplicates data on purpose |
| Reconciliation record | Result of comparing copy to owner | Detecting drift | Low | Detects, does not explain |

Select per table type: [Master reference table (owner)](#table-master-reference), [Cache of master reference (consumer)](#table-master-cache), [Transaction table](#table-transaction), [Projection table](#table-projection), [Crosswalk table](#table-crosswalk).

<a id="accountability-default"></a>

## Accountability default (recommended)

**Proposal, revisable** once an audit or regulatory requirement is known.

| Table type | Default mechanism |
|---|---|
| Master reference ([Master reference table (owner)](#table-master-reference)) | Outbox / transaction log written with every change; `version`, `updated_by`, `updated_at` columns; effective-dated history only for entities where past values matter |
| Cache ([Cache of master reference (consumer)](#table-master-cache)) | `source_version`, `last_synced_at`; processed-event log committed with each apply; reconciliation record |
| Transaction ([Transaction table](#table-transaction)) | `created_by`, `created_at`, change reason; append-only log for business facts; snapshot where reproduction is required |
| Projection ([Projection table](#table-projection)) | `as_of`; rebuild-and-compare; processed-event log |
| Crosswalk ([Crosswalk table](#table-crosswalk)) | Never delete; `valid_from` / `valid_to`; approval recorded |

Upgrade to versioned or bitemporal history only when a requirement demands it. Retention of logs must be reconciled with erasure obligations ([Decision: delete policy and the erasure exception](#decision-delete-policy-and-erasure)).

Options: [Accountability options compared](#accountability-options).

<a id="table-naming-convention"></a>

## Table naming convention

Table names show which module owns a table, what kind of table it is, and which table it serves. The convention applies to the physical schema inside one module; it is not part of any public contract.

**Pattern**

| Table kind | Name | Example |
|---|---|---|
| Entity table (master reference, transaction, cache, projection) | `<Module>_<Entity>` | `Kmp_Kompetensi`, `Kmp_StandarKompetensiJabatan` |
| Reference set (fixed list or [External reference sets](#external-reference-sets)) | `<Module>_Ref<Name>` | `Kmp_RefJenisKompetensi`, `Kmp_RefUrusan` |
| Support table that exists only to serve one owner table | `<Module>_<Owner><Role>` | `Kmp_KompetensiLevel`, `Kmp_KompetensiLineage`, `Kmp_KompetensiChangeEvent`, `Kmp_StandarKompetensiJabatanLineage` |

**Rules**
- `Ref` is a prefix because it names a kind of table, not an owner.
- A support table puts its owner's full name first and its role last. Sorted by name, an owner's tables sit together, and the role word stays fixed and searchable.
- Use one role word per role across modules: `Level` (child rows of an aggregate), `Lineage` (`replaced_by` links for merge and split, see [Use case: merge](#uc-merge) and [Use case: split](#uc-split)), `ChangeEvent` (outbox, see [Flow: events plus read API](#flow-events-plus-read-api)), `History` (past versions, see [Accountability options compared](#accountability-options)), `Crosswalk` (external or legacy codes, see [Crosswalk table](#table-crosswalk)).
- Derived names follow the table name: the primary key column is `Id<Table>`, and constraints are `<kind><Module>_<Table>_<detail>` (`pkKmp_KompetensiLineage`, `fkKmp_KompetensiLineage_Lama`, `ixKmp_KompetensiLineage_Baru`).
- A support table with its own business identity, which other tables reference on their own, is an entity table and takes `<Module>_<Entity>` (`Kmp_StandarKompetensiJabatan`, not `Kmp_JabatanStandarKompetensi`).

**Why**: entity names in Indonesian put the main word first (`LevelKompetensi`, "level of kompetensi"); role words in English put it last (`KompetensiChangeEvent`). Mixing the two orders puts sibling tables in different places. The rule fixes the order whatever the language of the words.

**Trade-off**: an owner-first name can differ from the business term (`KompetensiLevel` rather than `LevelKompetensi`). Keep the business term in the data dictionary entry ([Data dictionary and ADRs](#data-dictionary-and-adr)), not in the table name.

**Example**: `outputs/hr_kompetensi_mssql.curr.sql` (Microsoft SQL Server, PascalCase) applies this convention.

<a id="guide-03-communication"></a>

# 3. Inter-system communication patterns

Flow, payload, technology requirements, and the rules that preserve data integrity and independence. Technology-neutral: capabilities are specified, products are not.

<a id="flow-events-plus-read-api"></a>

## Flow: events plus read API

**Default flow.** The owner writes the business change and an outbox record in one local transaction. A relay publishes the outbox record as a change event. Consumers apply the event to their cache. A read API gives lookup by ID and catch-up by cursor.

```
Owner: [business change + outbox row] -> relay -> event channel -> Consumer: [idempotent apply]
Consumer --(lookup / catch-up)--> Owner read API
```

**Why both.** Events give timeliness and independence; the read API gives recovery, bootstrap and on-demand lookup.

**Properties to assume**
- At-least-once delivery. The relay may publish twice, so consumers must be idempotent ([Rule: idempotent, version-guarded apply](#rule-idempotent-versioned-apply)).
- Per-entity order, not global order ([Rule: ordering per entity](#rule-ordering-per-entity)).
- Exactly-once across separate systems is not assumed.

**Alternatives by need** (shared database ruled out first)
- Answer needed now: synchronous API with timeouts and a circuit breaker; consumer depends on owner availability.
- React to change or keep a copy: events (this page).
- External organisation consumer: signed webhooks, retries, list endpoint for reconciliation.
- Bulk or analytical: batch files or a governed data product, separate from the operational interface.

Evidence: outbox and delivery semantics are partly verified (Confluent, microservices.io); the style comparison is unverified ([Source reliability](#source-reliability)). Related: [Flow: bootstrap and catch-up](#flow-bootstrap-and-catch-up), [Flow: reconciliation](#flow-reconciliation), [Decision: subscribe to events or call the owner on demand](#decision-subscribe-vs-call-on-demand).

<a id="flow-bootstrap-and-catch-up"></a>

## Flow: bootstrap and catch-up

A new consumer, a rebuilt consumer, or a consumer that missed events uses the same path:

1. Before the first page, record the event-stream position (or start consuming events).
2. Page through the owner's list endpoint with `updated_after` (or from the start), using opaque cursors.
3. Apply each record through the normal version-guarded apply ([Rule: idempotent, version-guarded apply](#rule-idempotent-versioned-apply)).
4. Consume events from the position recorded in step 1. The version check discards anything older than what paging already applied.
5. Run reconciliation ([Flow: reconciliation](#flow-reconciliation)).

Do not subscribe only after paging ends: changes made during paging to records already paged past would be lost until reconciliation. The event channel must therefore retain events at least as long as a full bootstrap takes ([Technology requirements (neutral)](#technology-neutral-requirements)).

**Cursor.** The owner sorts the list by (`updated_at`, `id`) and the opaque cursor encodes both. A cursor on `updated_at` alone skips rows that share a timestamp at a page boundary.

Using one apply path for both bootstrap and events removes a class of divergence bugs. Details in [Use case: initial master broadcast to empty caches](#uc-initial-broadcast) and [Use case: late joiner, missed or out-of-order events](#uc-late-joiner-and-gaps).

Snapshot-plus-change-feed bootstrap had no usable source in the research ([Source reliability](#source-reliability)); treat the ordering in steps 1 and 4 as design reasoning.

<a id="flow-reconciliation"></a>

## Flow: reconciliation

A scheduled comparison between a consumer's copy and the owner.

1. Compare counts or hashes per range (for example per ID range or per `updated_at` window).
2. For mismatching ranges, re-fetch records from the owner.
3. Apply through the normal version-guarded path.
4. Record the result (ranges checked, mismatches, repairs) for [Freshness and drift monitoring](#freshness-and-drift-monitoring).

Reconcile each cache against the owner, never against another consumer's copy ([Edge case: divergent reference caches](#edge-case-divergent-reference-caches)).

Frequency: nightly as a starting point; tune to freshness needs. Standard practice, unsourced in the research ([Source reliability](#source-reliability)).

<a id="payload-thin-vs-fat-events"></a>

## Payload: thin vs fat events

- **Thin (notification)**: ID and change type only. The consumer calls back for details. The owner stays authoritative and checks authorisation on each read; the consumer cannot finish work while the owner is down.
- **Fat (state transfer)**: carries the fields consumers need. Consumers work without calling the owner; the contract is wider, data is duplicated, and consistency is eventual.

**Recommended split** (reasoned, not sourced): fat events for non-sensitive attributes; thin event plus authorised read for sensitive attributes ([Payload: sensitive fields](#payload-sensitive-fields)).

**Rule**: an owner publishes only data it owns. A broadcasts `unit_id` and `position_id`, not their names ([Worked example: modules A, B, C, D](#worked-example)).

Envelope fields: [Payload: event envelope](#payload-event-envelope). Keep a read API alongside either shape ([Flow: events plus read API](#flow-events-plus-read-api)).

<a id="payload-event-envelope"></a>

## Payload: event envelope

Fields every event should carry (names illustrative, technology-neutral):

| Field | Purpose |
|---|---|
| `event_id` | Unique per event; duplicate detection |
| `entity_type`, `entity_id` | What changed; partition key |
| `version` | Monotonic per entity; ordering guard |
| `occurred_at` | When the change committed |
| `change_type` | Created, updated, status changed, replaced |
| `status` | Explicit lifecycle state (active, inactive, terminated, deleted) |
| `replaced_by` | For merge and split |
| `previous_*` | Previous values of fields consumers must decrement or re-index (for example `previous_unit_id`) |
| `effective_from` / `effective_to` | When the change applies, if different from `occurred_at` |
| `schema_version` | Contract version of the payload |
| fields | The non-sensitive attributes consumers need |

Rules: additive-only evolution ([Rule: additive-only contract](#rule-additive-only-contract)); unknown fields ignored by consumers; required fields validated.

<a id="payload-sensitive-fields"></a>

## Payload: sensitive fields

Keep salary, national ID and similar data out of events. The owner exposes them through an endpoint that authorises each read and scopes fields per consumer.

Reasons: replicated sensitive data multiplies erasure and breach obligations ([Security and erasure propagation](#security-and-erasure-propagation)) and defeats per-read authorisation.

Edge case: [Edge case: sensitive fields](#edge-case-sensitive-fields). Grounding: API object-property-level authorisation and unsafe API consumption are named risks in the OWASP API Top 10 (2023), per the research report; the field-scoping rule is an inference.

<a id="technology-neutral-requirements"></a>

## Technology requirements (neutral)

The guide does not pick products. Choose any technology that provides these capabilities. Examples are examples only.

| Capability | Requirement | Example (illustrative) |
|---|---|---|
| Atomic change plus outbox | Business change and outbox record in one local transaction | Relational transaction with an outbox table |
| Event channel | At-least-once delivery; ordering per entity key; retention long enough for catch-up and at least as long as a full bootstrap; consumer-side offset or equivalent | A message broker or log |
| Schema registry / contract store | Enforce compatibility; stored versions; FULL or FULL_TRANSITIVE preferred | A schema registry |
| Request/response contract | Machine-readable, reviewed before implementation | An OpenAPI document |
| Read API | Lookup by ID; cursor-paginated list with `updated_after`, sorted by (`updated_at`, `id`) | REST or RPC |
| Error format | Standard, machine-readable | `application/problem+json` |
| Idempotency | Idempotency key policy for retried writes (documented by the team; the IETF draft expired) | Header-based key |
| Concurrency | Optimistic concurrency on writes | ETag with `If-Match` (412 on mismatch) |
| Identity and authorisation | Per-consumer authorisation, field scoping | Team's standard; not researched |
| Observability | Trace and correlation IDs, lag metrics | Not researched |

Not covered by any source in the research: retries and backoff, dead-letter handling, circuit breakers, tracing, SLOs, OAuth/mTLS ([Source reliability](#source-reliability)). Choose these deliberately and record them in an ADR ([Data dictionary and ADRs](#data-dictionary-and-adr)).

<a id="rule-no-shared-database"></a>

## Rule: no shared databases

A consumer never reads or writes another module's database, and no cross-system foreign keys exist. A shared database makes the owner's schema a public contract, so every internal change becomes a cross-team change.

Consumers get data only through the owner's contract ([Rule: additive-only contract](#rule-additive-only-contract)) and keep their own copy ([Cache of master reference (consumer)](#table-master-cache)).

<a id="rule-additive-only-contract"></a>

## Rule: additive-only contract

**Safe changes**: add optional fields (with a defined default or meaning when absent), endpoints, enum values.
**Breaking changes**: remove, rename or retype a field; change resource names, value formats or defaults; add a new required field.

Practices:
- Contract first: write and review the spec before implementing.
- Expose the minimum set of fields at first. What is exposed and relied on is effectively permanent.
- Keep the published model separate from the internal model.
- Consumers read only the fields they need, ignore unknown fields and unknown enum values, and fail loudly on missing or malformed required data (tolerant reader, bounded by the position that blanket tolerance is harmful).
- For events, enforce compatibility in a registry; FULL or FULL_TRANSITIVE lets either side upgrade first (the TRANSITIVE preference is an inference).

Retire fields through [Rule: deprecation and sunset](#rule-deprecation-and-sunset). Verify with [Rule: contract testing](#rule-contract-testing). Style choice: [Decision: contract versioning style](#decision-contract-versioning-style).

<a id="rule-idempotent-versioned-apply"></a>

## Rule: idempotent, version-guarded apply

The consumer's only way to change a cache or projection.

1. Read the event (or fetched record).
2. In one transaction: compare `version` with stored `source_version`.
   - No stored row: insert; treat the stored version as 0. A placeholder row stores no version, so the first real event or fetch fills it.
   - Greater: apply; store new version; record processed event ID.
   - Equal: duplicate; ignore.
   - Lower: late; ignore.
3. Commit.

Properties:
- Safe under duplicates, reordering and replays.
- Deleted or inactive state is a versioned state, so a stale update cannot resurrect a record ([Rule: deletes are explicit versioned states](#rule-deletes-as-state)).
- Same path for events and for bootstrap or repair ([Flow: bootstrap and catch-up](#flow-bootstrap-and-catch-up)).
- Unknown IDs on dependent events: store or fetch, do not drop silently.
- An implementation as a bare `UPDATE ... WHERE source_version < version` updates zero rows when no row exists and loses the event; handle the insert explicitly.

Storing the processed event ID in the same transaction as the change is the relational adaptation of the "commit state with offset" idea (inference).

<a id="rule-ordering-per-entity"></a>

## Rule: ordering per entity

Global order is not needed. Partition events by entity ID so per-entity order holds, and carry a monotonic `version` so consumers can detect and discard late events ([Rule: idempotent, version-guarded apply](#rule-idempotent-versioned-apply)).

For writes through APIs, use optimistic concurrency (ETag and `If-Match`; 412 on mismatch). The client refetches and reconciles, not retries blindly.

Cross-entity ordering (for example the unit exists before an assignment references it) is handled by validation and lazy resolution, not by global ordering ([Rule: validate IDs against the local cache](#rule-validate-ids-locally)).

<a id="rule-validate-ids-locally"></a>

## Rule: validate IDs against the local cache

Before accepting a write that references another module's entity, validate the ID against the local cache ([Cache of master reference (consumer)](#table-master-cache)): it exists and is active (or valid on the effective date).

- A validates `unit_id` and `position_id` against its cache of C before accepting an assignment.
- D validates the unit before accepting a requirement.
- Unknown ID: reject, or accept as pending and re-validate, per business rule ([Edge case: owner unavailable](#edge-case-owner-unavailable)).
- No cross-system foreign key constraints.
- This rule covers writes a module accepts from its users. It does not apply to events from an owner: those were validated by the owner ([Edge case: divergent reference caches](#edge-case-divergent-reference-caches)).

<a id="rule-deletes-as-state"></a>

## Rule: deletes are explicit versioned states

- The owner marks records `inactive` (or `terminated`, `deleted`) with a new `version`; it does not remove the row by default.
- Merge and split use `replaced_by`.
- A null-value tombstone in a compacted log means earlier records may be removed and does not protect a late joiner; a retained, versioned status does (the second point is an inference).
- Consumers keep the state so late or stale updates cannot resurrect the record.

Exception for erasure: [Decision: delete policy and the erasure exception](#decision-delete-policy-and-erasure). Use cases: [Use case: delete](#uc-delete).

<a id="rule-deprecation-and-sunset"></a>

## Rule: deprecation and sunset

To remove a field or endpoint:
1. Mark deprecated in the spec.
2. Send `Deprecation` and `Sunset` headers (hints; clients must work without them).
3. Watch usage until it reaches zero.
4. Remove only then, and for incompatible changes publish a new major version.

Release pipeline: a contract check blocks removals, renames, retypings and new required fields ([Rule: contract testing](#rule-contract-testing)). Related: [Rule: additive-only contract](#rule-additive-only-contract), [Decision: contract versioning style](#decision-contract-versioning-style).

<a id="rule-contract-testing"></a>

## Rule: contract testing

Catch breaking changes before release.

- **Spec diffing in CI**: compare the new spec with the previous one; simplest when there are many or unknown consumers.
- **Consumer-driven contract tests**: consumers record what they use and replay it against the owner; viable with one producer and one known consumer. Classic tools fit event-driven systems poorly.
- **Bi-directional contract testing**: compares the owner's spec with consumers' recorded usage; lighter, weaker guarantees.

Choice by consumer count is an inference. Sources were secondary ([Source reliability](#source-reliability)).

<a id="security-and-erasure-propagation"></a>

## Security and erasure propagation

- **Scope fields per consumer** so a copy holds only what it needs.
- **Validate the owner's payloads** instead of trusting them.
- **Inventory interfaces and versions.**
- **Idempotency keys** are high-entropy and cached under a composite key so one client cannot read another's cached response.
- **Erasure**: when the owner deletes or anonymises a person, the change must reach every consumer's copy, including caches, projections, backups and logs. The change carries a new version so version guards apply it; caches and projections take it through the apply path, and a tracked purge job covers logs, snapshots and backups. Applicable law (Indonesia's UU PDP, GDPR) was not researched; confirm with legal counsel.

Design hook: erasure is the one case where a hard delete or anonymisation is allowed ([Decision: delete policy and the erasure exception](#decision-delete-policy-and-erasure)); [Accountability default (recommended)](#accountability-default) logs must be consistent with it.

<a id="guide-04-use-cases"></a>

# 4. Use cases

What happens to each module type and table type in common situations. Split and merge apply to any entity whose business rules allow them. Each page states effects on the owner, type 1 consumer, type 2 consumer, and tables.

<a id="uc-initial-broadcast"></a>

## Use case: initial master broadcast to empty caches

**Trigger**: a new owner goes live, or a consumer is added with an empty cache.

**Steps**
1. Order the entities by dependency (referenced before referencing; see [Edge case: circular dependencies](#edge-case-circular-dependencies)).
2. Owner populates its master with IDs and `version`.
3. Consumers record the event-stream position, then backfill through the list endpoint ([Flow: bootstrap and catch-up](#flow-bootstrap-and-catch-up)), or the owner replays current state as events (one event per record, current version).
4. Consumers apply through the idempotent path ([Rule: idempotent, version-guarded apply](#rule-idempotent-versioned-apply)).
5. Reconcile ([Flow: reconciliation](#flow-reconciliation)).
6. Only then enable validations that depend on the cache (for example assignments).

**Effect by module type**
- Owner ([Data owner and broadcaster (source of truth)](#data-owner-broadcaster)): serves list and events; no schema change.
- Consumer type 1 ([Consumer type 1: reference data as attributes](#consumer-attribute-user)): cache fills; placeholders resolve; no business impact.
- Consumer type 2 ([Consumer type 2: reference data as index](#consumer-index-user)): validate that existing index records (D's requirements) match cached IDs; list unmatched ones for review.

**Effect by table type**: master gains rows; cache gains rows with `source_version`; transaction tables unchanged; projections are rebuilt ([Projection table](#table-projection)).

**Edge cases**: records arriving before their parent (hierarchy: store, resolve later); large volume (page by cursor on `updated_at` plus `id`); events arriving mid-backfill (consumed from the recorded position; version guard discards older).

<a id="uc-new-record"></a>

## Use case: new record

**Steps**
1. Owner validates and creates the record, issues the ID ([Opaque permanent ID](#opaque-permanent-id)), `version = 1`, writes the outbox row.
2. Event published; consumers apply it as an insert in their cache.
3. Consumer records referencing the new ID can now pass local validation ([Rule: validate IDs against the local cache](#rule-validate-ids-locally)).

**Effect by module type**
- Owner: new master row.
- Type 1: new cache row; nothing else changes.
- Type 2: new possible index value; no existing records change. D may prompt for a requirement for a new unit.

**Edge cases**
- A consumer sees an ID in a dependent event before the record event (order across entities is not guaranteed): store a placeholder or fetch on miss ([Eventual consistency and freshness](#eventual-consistency)).
- Duplicate event: ignored by version check.
- Duplicate real-world entity (created twice): handle by [Use case: merge](#uc-merge).

<a id="uc-update-rename-reparent"></a>

## Use case: update, rename, reparent

**Steps**: owner changes the record, increments `version`, publishes; consumers apply if newer.

**Effect by module type**
- Owner: master row updated (history row where kept; [Accountability default (recommended)](#accountability-default)).
- Type 1: cache updated; displays change. Historical documents show the new name unless a snapshot was stored ([Transaction table](#table-transaction)).
- Type 2: rename has no business effect. Reparent changes roll-up totals ([Edge case: hierarchy changes](#edge-case-hierarchy-changes)); effective-dated requirements remain valid.
- Projections: events must carry previous values when a field changes the grouping (transfer between units; [Projection holder](#projection-module)).

**Edge cases**: names needed "as of" a past date require effective-dated names in the owner; cycles in hierarchy rejected by the owner; late events ignored by version.

<a id="uc-split"></a>

## Use case: split

**Applicability**: any entity where business rules allow one record to become several (a unit divided into two units). Decide per entity in the [Data dictionary and ADRs](#data-dictionary-and-adr) entry.

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

**Edge cases**: split in progress (successors created before the original is retired); partial reassignment; reversal (do not reuse the original; create a new record or [Use case: merge](#uc-merge)).

<a id="uc-merge"></a>

## Use case: merge

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

**Edge cases**: conflicting attribute values between merged records (owner decides); a merge reversed later (treat as a [Use case: split](#uc-split)); duplicates created in error in a consumer's own table (its own matter).

<a id="uc-delete"></a>

## Use case: delete

Policy: [Decision: delete policy and the erasure exception](#decision-delete-policy-and-erasure).

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

<a id="uc-late-joiner-and-gaps"></a>

## Use case: late joiner, missed or out-of-order events

- **Late joiner or rebuilt consumer**: [Flow: bootstrap and catch-up](#flow-bootstrap-and-catch-up); a retained delete state is needed so history is not lost ([Rule: deletes are explicit versioned states](#rule-deletes-as-state)).
- **Missed events**: reconciliation detects the range ([Flow: reconciliation](#flow-reconciliation)); re-fetch and apply through the version check.
- **Out-of-order**: older versions ignored ([Rule: idempotent, version-guarded apply](#rule-idempotent-versioned-apply)).
- **Event for an unknown dependent ID**: placeholder or fetch on miss.
- **Poison event** (malformed): reject and route to a holding area for review; do not block the entity stream. Dead-letter handling was not researched ([Source reliability](#source-reliability)).

<a id="uc-external-edition-change"></a>

## Use case: external reference edition change

An external authority publishes a new edition of a code set ([External reference sets](#external-reference-sets)).

1. The steward reviews the change and approves.
2. The owner imports the new edition, adds new entries, closes changed ones with `valid_to`, and updates the [Crosswalk table](#table-crosswalk).
3. The owner publishes the new edition with a cutoff date for the old one.
4. Consumers apply it; old codes stay resolvable.
5. Consumers with local extensions ([Transaction table](#table-transaction)) check their keys.

**Edge cases**: entries split or merged by the authority (use `replaced_by` like [Use case: split](#uc-split), [Use case: merge](#uc-merge)); the organisation's definition differs from the new edition (keep both, map explicitly).

<a id="guide-05-operations"></a>

# 5. Operations and governance

Governance of reference data (including external sets) and monitoring of freshness.

<a id="external-reference-sets"></a>

## External reference sets

Reference data not produced by an internal module (region codes, tax status, education levels). Give each set one internal owner ([Owner of external reference sets](#external-reference-owner)) and a named data steward.

### Keying decision

| Situation | Action |
|---|---|
| Stable public standard code (ISO etc.) | Use the external code as the shared key. Store the standard and edition. |
| Authority changes codes over time | Key by code plus edition or effective date. Keep old codes resolvable. |
| No stable code or free-text list | Owner mints an internal ID and keeps a [Crosswalk table](#table-crosswalk) to the external entry. |
| Organisation's definition differs from the external one | Keep both and map explicitly. |

### Process
- Owner imports from the source, validates, and publishes versioned events or a dataset plus a read API.
- Other systems subscribe and reference entries by code, never by text.
- Write each set into a data dictionary entry (authority, edition, owner, update schedule, change approval); record the source choice in an ADR ([Data dictionary and ADRs](#data-dictionary-and-adr)).
- Pin editions; announce edition changes with a cutoff date for the old one ([Use case: external reference edition change](#uc-external-edition-change)).
- Local extensions go in the consumer's own table keyed by the code.
- One name must never carry two meanings; if meanings conflict, split into two named sets.

### Do not
Let each system download the external list itself, store only name text, or hard-delete retired entries.

Open item: identify the actual sets in use and map each to the table above.

<a id="data-dictionary-and-adr"></a>

## Data dictionary and ADRs

Each entity and each reference set gets a data dictionary entry, and each design choice gets an architecture decision record (ADR).

**Data dictionary entry** (minimum): entity name; owner module and data steward; ID format; business keys; fields published and their sensitivity; lifecycle states; split/merge applicability; consumers and their type; contract location and version; for external sets: authority, edition, update schedule, change approval.

**ADR** (minimum): context, options considered, decision, consequences, date. Record at least: technology choices ([Technology requirements (neutral)](#technology-neutral-requirements)), the choices in [Decision: subscribe to events or call the owner on demand](#decision-subscribe-vs-call-on-demand), [Decision: delete policy and the erasure exception](#decision-delete-policy-and-erasure), [Decision: contract versioning style](#decision-contract-versioning-style), and the keying choice for each external set ([External reference sets](#external-reference-sets)).

Verify the unverified research sections before an ADR relies on them ([Source reliability](#source-reliability)).

<a id="freshness-and-drift-monitoring"></a>

## Freshness and drift monitoring

Measure, per consumer and entity:
- **Lag**: time from the owner's `occurred_at` to the consumer's `last_synced_at`.
- **Drift**: mismatches found by [Flow: reconciliation](#flow-reconciliation) and repair counts.
- **Rejects**: events refused for malformed required data.
- **Placeholders**: unknown-ID rows awaiting resolution.

Display "as of" timestamps in UIs ([Eventual consistency and freshness](#eventual-consistency)). Alerting thresholds, SLOs, tracing and dead-letter handling had no usable source ([Source reliability](#source-reliability)); define them in ADRs ([Data dictionary and ADRs](#data-dictionary-and-adr)).

<a id="guide-06-appendices"></a>

# 6. Appendices: decisions, checklist, reliability

Open choices with options and the conditions that favour each, a design checklist, and the reliability of the evidence.

<a id="decision-subscribe-vs-call-on-demand"></a>

## Decision: subscribe to events or call the owner on demand

Open point in the source summary. Both options are valid; the conditions decide.

| | Option 1: subscribe and keep a local cache | Option 2: call the owner's read API on demand |
|---|---|---|
| Availability | Consumer works if owner is down | Consumer fails with the owner |
| Freshness | Eventual; show "as of" | Exact at read time |
| Complexity | Higher: cache, apply path, reconciliation | Lower: no local copy |
| Coupling | Contract-level only | Runtime coupling plus latency |
| Load on owner | Low, steady | Proportional to reads |
| Joins and reporting | Local joins | Cross-system calls per row |
| Data duplication | Yes (erasure scope grows) | No |

**Choose option 1 when**
- the consumer must keep operating when the owner is down;
- it joins or filters by the data locally (names in lists, reports, roll-ups);
- read volume is high;
- the data changes rarely relative to reads (reference data such as units and positions).

**Choose option 2 when**
- exactness at decision time is required (for example a permission or an amount);
- the data is sensitive and should not be replicated ([Payload: sensitive fields](#payload-sensitive-fields));
- volume is low and the owner is highly available;
- the team cannot yet operate caches and reconciliation.

**Hybrid (common)**: cache reference data (units, positions); call on demand for sensitive or exactness-critical data; call on a cache miss.

Applies to B and D in [Worked example: modules A, B, C, D](#worked-example); for D's headcount see [Projection holder](#projection-module). Related: [Flow: events plus read API](#flow-events-plus-read-api), [Edge case: owner unavailable](#edge-case-owner-unavailable).

<a id="decision-delete-policy-and-erasure"></a>

## Decision: delete policy and the erasure exception

**Default**: no hard delete. The owner sets a lifecycle status with a new version; consumers keep the state ([Rule: deletes are explicit versioned states](#rule-deletes-as-state)).

**Exception (allowed by the author)**: legal erasure or anonymisation of personal data, and correction of records created in error.

| Case | Mechanism | Notes |
|---|---|---|
| Normal retirement | Status `inactive` or `terminated` | Reversible; history kept |
| Merge / split | Status plus `replaced_by` | See [Use case: merge](#uc-merge), [Use case: split](#uc-split) |
| Created in error | Mark `void`; hard delete only if never published | If published, treat as erasure |
| Personal-data erasure | Anonymise fields in place, keep the ID, issue a new version; or hard delete with an erasure event | Event must reach every copy; see below |

**Erasure design options**
- *Anonymise in place* (recommended starting point, proposal): replace personal fields with null/placeholder, keep the ID and status, and increment the version. Consumers apply it as a normal update, so no stale copy can resurrect the data, and references stay valid.
- *Hard delete plus erasure event*: remove the row and send an event telling consumers to remove theirs. Use when the law or policy requires the identifier itself to go. Consumers must then handle dangling references ([Consumer type 1: reference data as attributes](#consumer-attribute-user), [Consumer type 2: reference data as index](#consumer-index-user)), and the "never reuse an ID" rule must still hold.

**Required for either**
- The erasure always carries a new version. An erasure at the stored version is ignored by every consumer's version guard as a duplicate, and the data stays.
- Caches and projections erase through [Rule: idempotent, version-guarded apply](#rule-idempotent-versioned-apply), like any update.
- Logs, outbox payloads, snapshots and backups are not reached by the apply path. A purge job in each module, triggered by the same erasure event, clears them ([Transaction table](#table-transaction) snapshots; [Accountability default (recommended)](#accountability-default) logs). Completion is tracked per consumer; ad-hoc scripts leave no record.
- Reconciliation confirms no copy remains ([Flow: reconciliation](#flow-reconciliation)).
- Legal requirements (UU PDP, GDPR) were not researched; confirm with legal counsel ([Security and erasure propagation](#security-and-erasure-propagation)).

Use case: [Use case: delete](#uc-delete).

<a id="decision-contract-versioning-style"></a>

## Decision: contract versioning style

The guides read disagree. The choice is a team convention, not a correctness issue.

| Style | Source position | Notes |
|---|---|---|
| Major version in the URL path (`/v1/`), no minor versions | Google API guidance | Simple; visible; new major only for incompatible change |
| Avoid versions; media-type versioning only for incompatible change; no URL versions | Zalando guidelines | Pushes toward compatible evolution |
| Date-based version pinned per account, responses transformed backward | Stripe (2017 post, may be dated) | Heavy for two internal systems |

**Suggestion for a few internal modules** (inference): additive change within one major version; a new major version only for a truly incompatible change.

For events, use schema compatibility in a registry ([Rule: additive-only contract](#rule-additive-only-contract)). Retirement: [Rule: deprecation and sunset](#rule-deprecation-and-sunset).

<a id="design-checklist-new-module"></a>

## Design checklist for a new module

**Scope**
- [ ] Entities owned and consumed listed; each classified ([Data owner and broadcaster (source of truth)](#data-owner-broadcaster), [Consumer type 1: reference data as attributes](#consumer-attribute-user), [Consumer type 2: reference data as index](#consumer-index-user), [Hybrid module (owner and consumer)](#hybrid-module)).
- [ ] Scope criteria applied ([Module scope criteria](#module-scope-criteria)); features placed ([Feature placement test](#feature-placement-test)).
- [ ] Dependency graph per entity is acyclic.

**Tables**
- [ ] Each table typed ([2. Table types and accountability](#guide-02-table-types)); accountability chosen ([Accountability default (recommended)](#accountability-default)).
- [ ] No cross-system foreign keys; ID columns unique; business keys as attributes.

**Communication**
- [ ] Owner: ID format, versioning, outbox, events, read API, contract, sensitive-field scoping.
- [ ] Consumer: apply path idempotent and version-guarded; adapter at the edge; unknown-ID handling.
- [ ] Bootstrap and reconciliation planned.
- [ ] Owner offline: the module keeps reading and validating from its cache; writes that need an unseen ID are rejected or held as pending, per business rule ([Edge case: owner unavailable](#edge-case-owner-unavailable)).

**Lifecycle**
- [ ] Statuses, `replaced_by`, effective dating defined; split/merge applicability decided.
- [ ] Type 2 consumers have a review process for flagged records.
- [ ] Erasure path defined, including a tracked purge job for logs, snapshots and backups ([Decision: delete policy and the erasure exception](#decision-delete-policy-and-erasure)).

**Governance**
- [ ] Data dictionary entry and ADRs written ([Data dictionary and ADRs](#data-dictionary-and-adr)).
- [ ] Monitoring defined ([Freshness and drift monitoring](#freshness-and-drift-monitoring)).

<a id="source-reliability"></a>

## Source reliability

How far each part of this wiki can be trusted. Tags on pages: `unverified`, `partly-verified`, `proposal`.

| Area | Status | Pages |
|---|---|---|
| Ownership, IDs, system of record, anti-corruption layer | Not source-verified (background knowledge) | [Opaque permanent ID](#opaque-permanent-id), [System of record](#system-of-record), [Anti-corruption layer](#anti-corruption-layer) |
| Integration styles, thin vs fat events, coupling | Entirely unverified | [Payload: thin vs fat events](#payload-thin-vs-fat-events), [Flow: events plus read API](#flow-events-plus-read-api) |
| Delivery semantics, outbox, idempotency, ETags, tombstones, sagas | Partly verified (Confluent, microservices.io, Stripe, Azure, Debezium, Ed-Fi) | [Rule: idempotent, version-guarded apply](#rule-idempotent-versioned-apply), [Rule: ordering per entity](#rule-ordering-per-entity), [Rule: deletes are explicit versioned states](#rule-deletes-as-state) |
| Contracts, compatibility, versioning, deprecation, errors | Partly verified (Zalando, Google AIPs, Confluent, RFC 9457/9745/9413) | [Rule: additive-only contract](#rule-additive-only-contract), [Rule: deprecation and sunset](#rule-deprecation-and-sunset), [Decision: contract versioning style](#decision-contract-versioning-style) |
| Units, positions, headcount, external reference patterns | Design reasoning, not researched | [Consumer type 2: reference data as index](#consumer-index-user), [Projection holder](#projection-module), [External reference sets](#external-reference-sets), use cases |
| Scope criteria, feature placement, accountability default | Assistant proposals | [Module scope criteria](#module-scope-criteria), [Feature placement test](#feature-placement-test), [Accountability default (recommended)](#accountability-default) |

**No usable source**: retries and backoff, dead-letter queues, circuit breakers, tracing, SLOs, drift detection, OAuth/mTLS, personal-data erasure law (UU PDP, GDPR; confirm with legal), snapshot-plus-change-feed bootstrap, event-contract testing.

**Other cautions**: the Idempotency-Key header is an expired IETF draft, not a standard; Stripe's versioning post is from 2017; page fetching was largely blocked during the research, so verify the unverified sections before relying on them in an ADR.

Origin: *API design for system integration (research report)* (`raw/api-design-for-system-integration.md`), *OSDM system integration summary* (`raw/osdm-integration-summary.md`).
