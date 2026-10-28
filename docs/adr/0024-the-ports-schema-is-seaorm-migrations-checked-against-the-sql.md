# 24. The Port's schema is SeaORM migrations checked against the SQL

Date: 2026-10-28

## Status

Accepted. Decided by the repo owner while working the Port map's ticket *Schema
and migrations in the Port* (#9, map #1).

## Context

The Node service's schema is 12 hand-written SQL files in
`apps/api/src/db/migrations/`. They hold Postgres enums, named CHECKs, a standalone
sequence, the `dds_app` role and column-level grants that the e2e suites test.
The Port re-implements the service on axum + SeaORM, and during the
wire-compatible phase the Node and Rust servers must run against the same
schema.

Every Port database starts empty: local development, CI scratch databases, and
the deployed Port, which is a separate service from the Pilot (ADR 0023). No
database migrated by Drizzle is ever adopted.

`docs/research/seaorm-existing-schema.md` found that the `sea-query` DSL cannot
express `CREATE ROLE`, `GRANT`/`REVOKE`, `CREATE SEQUENCE` or a `DO` block, so a
DSL rewrite still needs raw SQL in 8 of the 12 files.

## Decision

**The Port's schema is written as SeaORM migrations in the `sea-query` DSL**,
with `execute_unprepared` only where the DSL cannot reach (the role, the grants,
the sequence).

- **One to one.** Twelve migrations, `m0000` to `m0011`, each a translation of
  the `.sql` file with the same number.
- **Checked against the SQL.** Until cutover, CI migrates one scratch database
  with Drizzle and one with the Rust migrator. Their normalised schema dumps,
  their grants and their `province` rows must be identical. Until then, a
  schema change is written in both notations. The test is deleted with the
  `.sql` files.
- **The province seed is generated.** A generator writes a committed
  `province_seed.generated.rs` from `docs/provinces.csv`. It feeds both the
  `m0003` insert and the boot checksum check. The CSV stays canonical.
- **Entities are generated.** `sea-orm-cli generate entity` runs against a
  freshly migrated database and its output is committed. CI regenerates the
  files and fails on any diff. Domain types live in their own modules and never
  edit generated files.
- **Forward-only.** No `down` bodies, and `fresh`, `refresh` and `reset` are not
  used (`fresh` leaves `request_reference_seq` and the role behind). A reset
  means recreating the database.

## Considered options

- **Keep the 12 `.sql` files verbatim**, each run through `execute_unprepared`:
  one source of truth, with no drift possible between the halves. Declined in
  favour of SeaORM-style migrations.
- **A fresh line**, with one squashed baseline: declined, because it loses the
  per-file comparison with the SQL.
- **Rely on the e2e suites, or on review, to catch drift**: declined, because
  they don't cover every CHECK and index.
- **Entities generated once and then hand-maintained, or hand-written**:
  declined, because enum labels and columns could drift from the migrations
  unseen.

## Consequences

Until cutover the schema exists in two notations, and the equivalence test is
what makes that safe. A failure points at one translation, because the
numbering lines up. Adding an enum label means a migration plus regenerated
entities, and CI catches a missed regeneration.
