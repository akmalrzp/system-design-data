# Wiki Log

Append-only record of operations. Parse with `grep "^## \[" log.md`.

---

## [2026-10-09] schema | Created SCHEMA.md with page types: source, guide, concept, module-type, table-type, pattern, usecase, decision, ops
## [2026-10-09] ingest | osdm-integration-summary (raw/osdm-integration-summary.md)
## [2026-10-09] ingest | api-design-for-system-integration (raw/api-design-for-system-integration.md)
## [2026-10-09] ingest | author-decisions-2026-10-09 (raw/author-decisions-2026-10-09.md)
## [2026-10-09] ingest | Compiled guide pages (concepts, modules, tables, patterns, use cases, decisions, ops) from the three sources; assistant proposals tagged `proposal`
## [2026-10-21] add | tables/table-naming-convention (proposal from author session; placed in tables/, not patterns/, because it is intra-module schema, not inter-module communication)
## [2026-10-21] review | full-guide-gemini.md compared with wiki; author decisions applied: bootstrap records event position before paging + (updated_at, id) cursor; apply inserts when no row; merge/split open-vs-past record rule; erasure increments version + tracked purge job; snapshot trigger legal or business; crosswalk covers legacy codes (owner-held); checklist owner-offline item; glossary eventual consistency; additive contract says optional fields
