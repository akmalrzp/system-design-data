---
type: pattern
title: "Payload: thin vs fat events"
tags: [unverified]
sources: [osdm-integration-summary,api-design-for-system-integration]
created: 2026-10-09
updated: 2026-10-21
---

# Payload: thin vs fat events

- **Thin (notification)**: ID and change type only. The consumer calls back for details. The owner stays authoritative and checks authorisation on each read; the consumer cannot finish work while the owner is down.
- **Fat (state transfer)**: carries the fields consumers need. Consumers work without calling the owner; the contract is wider, data is duplicated, and consistency is eventual.

**Recommended split** (reasoned, not sourced): fat events for non-sensitive attributes; thin event plus authorised read for sensitive attributes ([payload-sensitive-fields](payload-sensitive-fields.md)).

**Rule**: an owner publishes only its own facts. A reference to another owner's entity is published as its ID only, never with that owner's attributes. A broadcasts `unit_id` and `position_id`, not their names ([worked-example](../concepts/worked-example.md)).

Envelope fields: [payload-event-envelope](payload-event-envelope.md). Keep a read API alongside either shape ([flow-events-plus-read-api](flow-events-plus-read-api.md)).
