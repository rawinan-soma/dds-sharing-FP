# SeaORM over the existing hand-written SQL schema

- **Question:** Can SeaORM take on the schema in `apps/api/src/db/migrations/*.sql` (12 Drizzle files: enums, constraints, grants, a seed, plus advisory locks taken in code)? Three ways to compare: `sea-orm-cli generate entity` from a live database, running the raw SQL files through `sea-orm-migration`, and rewriting the migrations in SeaORM's DSL. What can SeaORM not express, and how does raw SQL sit alongside entities?
- **Issue:** #4 (map #1; blocks #9, "Schema and migrations in the Port")
- **Date:** 2026-10-26
- **Versions examined:** `sea-orm` / `sea-orm-migration` / `sea-orm-cli` / `sea-orm-codegen` **2.0.4** (released 2026-09-27, the current stable release per the crates.io API), `sea-query` **1.0.2**, `sea-schema` **0.18.1** (pinned by `sea-orm-cli` 2.0.4), `sqlx-postgres` **0.9.0** (what `sea-orm` 2.0.4 depends on). On the Drizzle side, `drizzle-orm` 0.44.7 (`apps/api/package.json` asks for `^0.44.5`).
- **Primary sources:** each crate's published source, downloaded from `crates.io/api/v1/crates/<name>/<version>/download` and read directly. Citations look like `sea-orm-migration 2.0.4: src/migrator/exec.rs:242` and refer to that tarball (the same files are on docs.rs under "Source"). Also the SeaORM 2.0.x docs page [Writing Migration](https://www.sea-ql.org/SeaORM/docs/migration/writing-migration/), `drizzle-orm` 0.44.7's `pg-core/dialect.js`, and this repo. ⚠️ marks anything I did not verify.

This note reports facts and makes no decision. #9 makes the decision.

## Answer in brief

- **Entities over the existing schema work.** `sea-orm-cli generate entity` discovers every table and type this schema uses. Each Postgres enum becomes a `DeriveActiveEnum`. Entities never carry CHECK constraints, grants, roles, standalone sequences or non-unique indexes, and they don't need to: an entity is a query model, not the schema.
- **The migrations can run unchanged.** A `sea-orm-migration` migration can `execute_unprepared` the contents of each `.sql` file. On Postgres that call uses the simple-query protocol, so a whole multi-statement file, `DO $$…$$` block included, runs in one call. Drizzle's `--> statement-breakpoint` markers are plain SQL comments. Each migration runs in its own transaction.
- **The DSL rewrite falls short.** `sea-query` can express the enums, the named CHECKs, unique and partial indexes, FKs, defaults and the seed insert. It cannot express `CREATE ROLE`, `GRANT`/`REVOKE` (including the column-level `GRANT UPDATE (…)`), `CREATE SEQUENCE` or a `DO` block. About half the statements are grants, so a DSL rewrite still needs raw SQL in most files.
- **Existing databases need adoption.** A live database migrated by Drizzle records its history in `drizzle.__drizzle_migrations`. `sea-orm-migration` reads only `seaql_migrations`, and there is no import between the two. #9 has to choose how an existing database gets adopted.
- **Advisory locks have no SeaORM API.** You write them as raw `SELECT pg_advisory_*` statements. Transaction-scoped locks work inside a SeaORM transaction. The tick's session-scoped lock needs one pinned connection, which SeaORM exposes only as a transaction or through the underlying `sqlx::PgPool`.

## 1. Inventory: what the 12 migrations actually use

Read from `apps/api/src/db/migrations/0000`–`0011`, plus `advisory-locks.ts`, `scheduler/tick.ts`, `requests/requests.service.ts`, `reviewer/reviewer-accounts.ts` and `test/support/scratch-database.ts`.

| Feature | Where | Detail |
|---|---|---|
| **Postgres enum types** (9) | 0001, 0004, 0007, 0008 | `actor_type`, `request_event_type` (26 values), `reviewer_event_type`, `request_state`, `extraction_job_status`, `mail_delivery_status`, `mail_kind`, `token_lookup_kind`, `token_lookup_outcome` |
| **`ALTER TYPE … ADD VALUE`** | 0010 | `reviewer_event_type` gains `password_reset`, `totp_reset` |
| **Named CHECK constraints** (11) | 0001, 0002, 0004, 0005, 0009 | Cross-column (`(actor_type = 'reviewer') = (reviewer_id IS NOT NULL)`), `IN (…) OR (… IS NULL AND … IS NULL)`, regex `~ '^[0-9]{2}$'` and `~ '^[a-z0-9._-]{3,32}$'`, `BETWEEN 1 AND 13`, `cardinality(report_codes) > 0`, date arithmetic `end_date - start_date <= 365`, single-row `id = 1` |
| **Unique constraints** (4) | 0004, 0005, 0008 | `request.reference`, `reviewer.username`, `reviewer_session.token_hash`, `download_token.token_hash` |
| **Plain b-tree indexes** (7) | 0005, 0007, 0008 | All on FK or lookup columns |
| **Partial index** | 0004, **dropped in 0011** | `request_event_submitted_ip … WHERE type = 'submitted'`. None remains in the current schema |
| **Foreign keys** (10) | 0004, 0005, 0007, 0008 | All `ON DELETE no action ON UPDATE no action`, added by `ALTER TABLE … ADD CONSTRAINT` |
| **Standalone sequence** | 0004 | `request_reference_seq`; the app calls `nextval()` |
| **`bigserial`** PKs | 0001, 0008 | `request_event`, `reviewer_event`, `token_lookup`. Grants name the implicit `*_id_seq` |
| **Column defaults** | many | `now()`, `gen_random_uuid()` (core since PG 13; compose runs `postgres:16-alpine`), `'{}'::jsonb`, enum literals, `true`, `0`, `1` |
| **Column types** | — | `uuid`, `text`, `char(2)`, `smallint`, `integer`, `bigint`, `boolean`, `date`, `timestamptz`, `jsonb`, `inet`, `text[]`, `char(2)[]` |
| **Role** | 0001 | `CREATE ROLE dds_app NOLOGIN` inside an idempotent `DO $$ … IF NOT EXISTS (SELECT FROM pg_roles …) $$` |
| **Grants / revokes** | 0001, 0002, 0004–0009 | `REVOKE ALL … FROM PUBLIC`; table-level `GRANT SELECT, INSERT[, UPDATE, DELETE]`; **column-level `GRANT UPDATE (col, …)`** on `request`, `reviewer`, `extraction_job`, `download_token`, `mail_delivery`, `scheduler_heartbeat`; `GRANT USAGE ON SEQUENCE`; `GRANT USAGE ON SCHEMA public`. The e2e suites check these grants (`test/audit-spine.e2e-spec.ts:391`, `test/scheduler-tick.e2e-spec.ts:766,787`) |
| **Seed data** | 0003 | One 77-row `INSERT INTO province`, generated from `docs/provinces.csv` (the header says never to edit it by hand) |
| **No-op baseline** | 0000 | A comment only |
| **Triggers, generated columns, views, functions, RLS, extensions** | — | **None used** |
| **Advisory locks** (in code) | — | `pg_advisory_xact_lock(64)` (reviewer-accounts); `pg_advisory_xact_lock(hashtextextended('request-submit:'‖email, 0))` (requests); **session-level** `pg_try_advisory_lock(7_215_003)` / `pg_advisory_unlock` on one pooled client (tick); session-level `pg_advisory_lock(6_425_001)` around migrations in tests, because the role's check-then-create races |

The Drizzle side, for reference: `drizzle-orm` 0.44.7 `pg-core/dialect.js:44–72` creates `drizzle.__drizzle_migrations (id serial, hash text, created_at bigint)`. It applies every journal entry whose `when` (`meta/_journal.json`) is later than the last recorded `created_at`, and it runs **all pending files in one transaction**. `migrator.js:16` splits each file on `--> statement-breakpoint` and sends each piece separately. `schema.ts` mirrors the tables for Drizzle's query builder; it adds no constraint that the SQL lacks.

## 2. Option A: `sea-orm-cli generate entity` from a live database

**How it works.** The CLI discovers the schema with `sea-schema` 0.18.1. It turns each table into a `sea_query::TableCreateStatement` and hands that to `sea-orm-codegen`.

**Kept:**

- **Columns and types** (`sea-orm-codegen 2.0.4: src/entity/column.rs:41–98`). `text` and `char(n)` become `String` (char keeps `ColumnType::Char(Some(n))`). `smallint`, `integer` and `bigint` become `i16`, `i32` and `i64`. `timestamptz` becomes `DateTimeWithTimeZone` (chrono) or `TimeDateTimeWithTimeZone` (time). `date` becomes `Date`. `jsonb` becomes `Json`. `uuid` becomes `Uuid`. `inet` becomes `IpNetwork`, which needs the `with-ipnetwork` feature. Arrays become `Vec<…>`, which needs `postgres-array`. Nullable columns become `Option<_>`. Any type not on the list hits `unimplemented!()` (line 98), and nothing in this schema does.
- **Enums.** Each Postgres enum becomes a Rust enum in `sea_orm_active_enums.rs`, annotated with `#[derive(…, EnumIter, DeriveActiveEnum)] #[sea_orm(rs_type = "Enum", db_type = "Enum", enum_name = "<pg name>")]`. Every variant gets `#[sea_orm(string_value = "<label>")]` (`src/entity/active_enum.rs:86–95`). Labels that aren't valid Rust identifiers get renamed with a warning. This schema's labels are all snake_case ASCII.
- **PKs, unique constraints** (as `unique` / `unique_key`, `src/entity/transformer.rs:39–74`) and **FKs**, which become `Relation`s. A FK whose column is also unique becomes `has_one` instead of `has_many` (`transformer.rs:146–169`). That would apply to `request_contact.request_id`, which is both PK and FK.

**Dropped.** The discovery layer reads more than codegen uses. `sea-schema 0.18.1: src/postgres/def/schema.rs:16–28` holds `check_constraints` and `exclusion_constraints`, but `src/postgres/writer/table.rs:4–21` writes only columns, PKs, unique constraints and FKs. So the generated entities carry **no CHECK constraints, no non-unique indexes, no grants and no role**. The standalone **`request_reference_seq`** doesn't appear either, because it isn't a column default.

**Defaults.** ⚠️ I did not check whether codegen emits `default_value` / `default_expr` for `now()` or `gen_random_uuid()`. Inserts would leave those columns `NotSet` in any case.

**Table filters.** `--ignore-tables` defaults to `seaql_migrations` (`sea-orm-cli 2.0.4: src/cli.rs:199–202`). `__drizzle_migrations` lives in schema `drizzle`, so discovery doesn't see it when it targets `public`.

**What it gives.** The entity code itself, not migrations. The database stays the source of truth.

## 3. Option B: the existing SQL files through `sea-orm-migration`

**Mechanism.** A `MigrationTrait` (`sea-orm-migration 2.0.4: src/lib.rs:26–43`) whose `up` calls `manager.get_connection().execute_unprepared(include_str!("…/0004_request.sql"))`. SeaORM's own docs show `execute_unprepared` "if the SQL statement doesn't have value bindings" and note that raw SQL gives up multi-backend support ([Writing Migration](https://www.sea-ql.org/SeaORM/docs/migration/writing-migration/)). The docs don't mention `include_str!`, which is plain Rust.

**Multiple statements in one call.** On Postgres, `execute_unprepared` calls `conn.execute(sqlx::AssertSqlSafe(sql))` with no arguments (`sea-orm 2.0.4: src/driver/sqlx_postgres.rs:197–205`). In `sqlx-postgres 0.9.0: src/connection/executor.rs` (`run`), the prepared or extended path runs only `if let Some(arguments)`. Without arguments it sends a simple `Query` message (`queue_simple_query`, ~line 284). Postgres accepts several `;`-separated statements in one simple query, `DO $$…$$` included. `--> statement-breakpoint` starts with `--`, so it is a comment. **A file can therefore run as-is, with no splitting.**

⚠️ I did not run this against a live database.

**Transactions.** By default each migration runs in **its own transaction on Postgres**, and the `seaql_migrations` row is inserted in that same transaction (`src/migrator/exec.rs:200–205, 242–277`; the docs say Postgres migrations run "atomically"). `use_transaction() -> Some(false)` turns this off for a migration, and `manager.begin()` then gives manual control (`src/manager.rs:53–70`). Drizzle runs all pending files in one transaction. Under either tool, `0010`'s `ALTER TYPE … ADD VALUE` runs inside a transaction. PG ≥12 allows that, but the new label can't be used until the transaction commits ([PG docs, ALTER TYPE notes](https://www.postgresql.org/docs/16/sql-altertype.html)). Nothing in 0010 uses the new labels.

**Tracking and adoption.**
- History lives in `seaql_migrations (version text PK, applied_at bigint)` (`src/seaql_migrations.rs`), and `MigratorTrait::migration_table_name` can rename it (`src/migrator.rs:58–61`).
- A migration whose row exists but whose code is missing is a hard error: "Migration file of version '…' is missing" (`src/migrator/exec.rs:78`).
- Nothing reads `drizzle.__drizzle_migrations`. A database that Drizzle has already migrated therefore looks to SeaORM as if it has no migrations applied, and replaying 0001 would fail on `CREATE TYPE` (no `IF NOT EXISTS`).
- To adopt one, someone has to pre-insert `seaql_migrations` rows, or the first migration has to detect the existing schema. That is a #9 decision.
- ⚠️ Whether a pilot or production database has already been migrated by Drizzle is a deploy fact I did not check. The repo history mentions a pilot deploy.

**No concurrency lock.** `Migrator::up` takes no advisory lock (no `lock` or `advisory` in `src/migrator*`). Drizzle doesn't either. The tests' `migrateSerialised` exists because two migrators would race on `CREATE ROLE`. That race stays the same under SeaORM.

**`fresh` / `refresh` / `reset`.** `drop_everything` drops every table, and on Postgres every type, cascading (`src/migrator/exec.rs:122–197`). It does **not** drop standalone sequences or roles. After a `fresh`, re-running 0004's `CREATE SEQUENCE request_reference_seq` would fail because the sequence already exists. ⚠️ I inferred this from the code and did not run it. `down` defaults to an error (`src/lib.rs:31–33`).

**Running the migrations.** `Migrator::up(&db, None)` runs in-process, or `sea-orm-cli migrate up` runs them from a migration crate.

## 4. Option C: rewrite in SeaORM's migration DSL (`sea-query` 1.0.2)

| Inventory item | DSL? | Source |
|---|---|---|
| `CREATE TYPE … AS ENUM` | ✅ `Type::create().as_enum(name).values([...])`, via `manager.create_type` | `sea-query 1.0.2: src/extension/postgres/types.rs:82,124,133`; `sea-orm-migration: src/manager.rs:89` |
| `ALTER TYPE … ADD VALUE [BEFORE/AFTER] [IF NOT EXISTS]` | ✅ `Type::alter().name(..).add_value(..)` | `types.rs:92,284,309,319,344` |
| Named CHECK | ✅ `Table::create().check(Check::named(name, expr))`; also column `.check(..)` | `src/table/constraint.rs:4–29`, `src/table/create.rs:168`, `src/table/column.rs:793` |
| The CHECK bodies (regex `~`, `cardinality()`, date minus date, `IN`, `IS NULL` equality) | ⚠️ partly. `Expr` covers `IN`, `IS NULL`, `BETWEEN` and comparisons. The regex operator, `cardinality()` and date arithmetic would most likely go in as `Expr::cust("…")` raw fragments | not traced operator by operator |
| UNIQUE, b-tree index, FK | ✅ `Index::create().unique()`, `ForeignKey::create()` | `src/index/create.rs:289` |
| Partial index (`WHERE`) | ✅ `IndexCreateStatement` implements `ConditionalStatement` (`and_where`) | `src/index/create.rs:159,218,380–390`. None remains after 0011 |
| Defaults `now()`, `gen_random_uuid()`, `'{}'::jsonb` | ✅ via `.default(Expr::cust(..))` or the built-in fns | ⚠️ exact helper names not checked |
| `inet`, arrays, `char(n)`, `jsonb`, `bigserial` | ✅ `ColumnType::{Inet, Array, Char, JsonBinary}`; big-integer auto-increment | codegen maps them back (§2) |
| `CREATE SEQUENCE` | ❌ no sequence statement in `sea-query` (grep over `src/` finds none) | — |
| `CREATE ROLE`, `DO $$…$$` | ❌ | — |
| `GRANT` / `REVOKE`, table-, column- or sequence-level | ❌ | — |
| Seed `INSERT` | ✅ `Query::insert()` or entity `ActiveModel::insert`. The docs recommend "Seed data with Entity" | Writing Migration page |

**Result.** Every migration except 0000, 0003, 0010 and 0011 contains grants. A DSL rewrite would therefore mix the DSL with `execute_unprepared` for grants and roles in **8 of the 12 files**, and for the sequence as well. The DSL is portable, but this schema is Postgres-only (enums, `inet`, arrays, grants), so portability buys nothing here.

**Entity-first schema sync.** SeaORM 2.0 also has `SchemaBuilder::sync` behind the `schema-sync` feature. It is documented as "experimental and exempt from semver", it only adds tables, columns, unique keys and FKs, and it never alters or drops (`sea-orm 2.0.4: src/schema/builder.rs:77–90`). It can't produce grants either.

## 5. How raw SQL sits alongside entities at runtime

- **Raw statements on any connection or transaction.** `ConnectionTrait::execute_unprepared(&str)` handles statements without binds. `execute_raw` / `query_one_raw` / `query_all_raw` take a `Statement::from_sql_and_values(DbBackend::Postgres, sql, values)` for bound `$1` values (`src/database/connection.rs:33`; the docs example). Both `DatabaseConnection` and `DatabaseTransaction` implement it (`src/database/transaction.rs:474`).
- **sqlx in the same process.** `DatabaseConnection::get_postgres_connection_pool() -> &sqlx::PgPool` (`src/database/db_connection.rs:841`) exposes the pool. `SqlxPostgresConnector::from_sqlx_postgres_pool(PgPool)` (`src/driver/sqlx_postgres.rs:169`) builds a SeaORM connection from an existing pool. Both sides must use the same `sqlx` version, which is **0.9** for `sea-orm` 2.0.4 (`Cargo.toml: [dependencies.sqlx] version = "0.9.0"`).

**Postgres enums at runtime (`DeriveActiveEnum`).**
- A `db_type = "Enum", enum_name = "…"` enum maps to the Postgres type by its string labels (`src/entity/active_enum.rs`; test fixture `src/tests_cfg/sea_orm_active_enums.rs:5`).
- **Adding a label is two edits**: the SQL `ALTER TYPE … ADD VALUE` and the Rust variant. Nothing ties them together.
- ⚠️ Reading a label the Rust enum lacks should fail to decode the row. That is the expected `TryGetable` behaviour; I did not test it.

**Advisory locks.** SeaORM has no advisory-lock API (`advisory` appears nowhere in `sea-orm` `src/`). Each lock is a raw `SELECT`:
- **Transaction-scoped** (`pg_advisory_xact_lock`, reviewer-accounts and request submit): run it as the first statement inside `db.begin()`. A `DatabaseTransaction` holds one connection until commit or rollback, which is what the current Drizzle `tx.execute(sql…)` does. `hashtextextended(…)` works as plain SQL with a bound value.
- **Session-scoped** (`pg_try_advisory_lock` / `pg_advisory_unlock` in `scheduler/tick.ts:126–140`, and the tests' migration lock): lock and unlock must run on the **same** connection. SeaORM's `DatabaseConnection` is a pool, and each `execute` may acquire a different connection (`sqlx_postgres.rs:200` acquires per call). Two ways to pin one connection:
  - hold a `DatabaseTransaction` open for the whole pass, which changes the pass's transaction shape;
  - take a `sqlx::pool::PoolConnection` from `get_postgres_connection_pool().acquire()` and run lock and unlock on it, outside SeaORM.

  ⚠️ I found no public SeaORM API that pins a non-transactional connection.
- The fixed keys in `advisory-locks.ts` (64, 7_215_003) and the test key 6_425_001 can be carried over as constants; nothing about them depends on the ORM.

## 6. Facts #9 will need

1. Entities can be generated from the database or written by hand. Either way, CHECKs, grants, the role, the sequence and non-unique indexes live only in SQL migrations.
2. The 12 files can run verbatim as 12 `sea-orm-migration` migrations through `execute_unprepared(include_str!(…))`. Each runs in its own transaction, unlike Drizzle's single transaction for the whole batch.
3. A DSL rewrite still needs raw SQL in 8 of 12 files (grants and role) and for `CREATE SEQUENCE`.
4. Adopting an already-migrated database means mapping `drizzle.__drizzle_migrations` to `seaql_migrations`. No tool does this for you.
5. Neither migrator serialises concurrent runs. The test harness's advisory-lock wrapper is still needed.
6. `fresh` leaves `request_reference_seq` and `dds_app` in place.
7. Session-level advisory locks need a pinned connection, which means a transaction or raw sqlx.
