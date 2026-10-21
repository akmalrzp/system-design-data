---
type: concept
title: "Glossary"
tags: [principle]
sources: [osdm-integration-summary,api-design-for-system-integration]
created: 2026-10-09
updated: 2026-10-21
---

# Glossary

- **Module**: an independently deployable system with its own database and release cycle (A, B, C, D in examples).
- **Data owner / system of record**: the module allowed to create and change a record. See [system-of-record](system-of-record.md).
- **Broadcaster**: an owner in its role of publishing change events and exposing a read API.
- **Consumer**: a module that holds references to or copies of another module's data.
- **Reference data**: data owned by one module and referred to by others (unit, position, region code).
- **Master reference table**: the owner's authoritative table. See [table-master-reference](../tables/table-master-reference.md).
- **Cache (of master)**: a consumer's read-only local copy. See [table-master-cache](../tables/table-master-cache.md).
- **Transaction table**: a consumer's or owner's own business records that reference IDs. See [table-transaction](../tables/table-transaction.md).
- **Projection**: derived data (for example counts per unit) built from events, not copied records. See [table-projection](../tables/table-projection.md).
- **Crosswalk**: mapping between internal IDs and external or legacy-system codes. See [table-crosswalk](../tables/table-crosswalk.md).
- **Contract**: the published, versioned shape of API responses and events.
- **Event envelope**: metadata fields on every event. See [payload-event-envelope](../patterns/payload-event-envelope.md).
- **Version**: a monotonic number per record, increased on every change.
- **Eventual consistency**: copies converge with the owner once changes stop and missed events are repaired by reconciliation; not a guarantee on its own. See [eventual-consistency](eventual-consistency.md).
- **Tombstone**: a retained marker that a record is deleted or inactive; here a versioned state, not an absence.
- **`replaced_by`**: pointer from a retired record to its successor(s) after merge or split.
- **Effective dating**: `effective_from` / `effective_to` on records whose meaning depends on a date.
- **Anti-corruption layer**: adapter translating an external model into the local one. See [anti-corruption-layer](anti-corruption-layer.md).
- **Data steward**: named person accountable for a reference set. See [external-reference-sets](../decisions/external-reference-sets.md).
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
