---
type: table-type
title: "Table naming convention"
tags: [proposal]
sources: []
created: 2026-10-21
updated: 2026-10-21
---

# Table naming convention

Table names show which module owns a table, what kind of table it is, and which table it serves. The convention applies to the physical schema inside one module; it is not part of any public contract.

**Pattern**

| Table kind | Name | Example |
|---|---|---|
| Entity table (master reference, transaction, cache, projection) | `<Module>_<Entity>` | `Kmp_Kompetensi`, `Kmp_StandarKompetensiJabatan` |
| Reference set (fixed list or [external reference set](../decisions/external-reference-sets.md)) | `<Module>_Ref<Name>` | `Kmp_RefJenisKompetensi`, `Kmp_RefUrusan` |
| Support table that exists only to serve one owner table | `<Module>_<Owner><Role>` | `Kmp_KompetensiLevel`, `Kmp_KompetensiLineage`, `Kmp_KompetensiChangeEvent`, `Kmp_StandarKompetensiJabatanLineage` |

**Rules**
- `Ref` is a prefix because it names a kind of table, not an owner.
- A support table puts its owner's full name first and its role last. Sorted by name, an owner's tables sit together, and the role word stays fixed and searchable.
- Use one role word per role across modules: `Level` (child rows of an aggregate), `Lineage` (`replaced_by` links for merge and split, see [uc-merge](../usecases/uc-merge.md) and [uc-split](../usecases/uc-split.md)), `ChangeEvent` (outbox, see [flow-events-plus-read-api](../patterns/flow-events-plus-read-api.md)), `History` (past versions, see [accountability-options](accountability-options.md)).
- Derived names follow the table name: the primary key column is `Id<Table>`, and constraints are `<kind><Module>_<Table>_<detail>` (`pkKmp_KompetensiLineage`, `fkKmp_KompetensiLineage_Lama`, `ixKmp_KompetensiLineage_Baru`).
- A support table with its own business identity, which other tables reference on their own, is an entity table and takes `<Module>_<Entity>` (`Kmp_StandarKompetensiJabatan`, not `Kmp_JabatanStandarKompetensi`).

**Why**: entity names in Indonesian put the main word first (`LevelKompetensi`, "level of kompetensi"); role words in English put it last (`KompetensiChangeEvent`). Mixing the two orders puts sibling tables in different places. The rule fixes the order whatever the language of the words.

**Trade-off**: an owner-first name can differ from the business term (`KompetensiLevel` rather than `LevelKompetensi`). Keep the business term in the data dictionary entry ([data-dictionary-and-adr](../ops/data-dictionary-and-adr.md)), not in the table name.

**Example**: `outputs/hr_kompetensi_mssql.curr.sql` (Microsoft SQL Server, PascalCase) applies this convention.
