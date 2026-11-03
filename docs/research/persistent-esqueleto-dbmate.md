# persistent and Esqueleto over a dbmate-owned schema

- **Question:** How do persistent + Esqueleto work over a schema that dbmate owns as plain SQL? Running the 12 Drizzle `.sql` files through dbmate (file format, `schema_migrations`, a baseline off `drizzle.__drizzle_migrations`, `dbmate dump`); hand-written entities with `migrate` disabled and how to check they match the real schema; how persistent maps what this schema uses; the Esqueleto features the queries need; and whether a transaction exposes its raw connection for advisory locks.
- **Issue:** #37 (map #35; blocks #41, #42, #46). Rust predecessor: #4, `docs/research/seaorm-existing-schema.md`.
- **Date:** 2026-11-03
- **Versions examined:** `dbmate` **v2.37.0** (released 2026-10-18, the latest GitHub release), which pins `github.com/lib/pq` **v1.12.3**. `persistent` **2.18.1.0** and `persistent-postgresql` **2.14.3.0** (both uploaded 2026-01-14, Hackage's newest normal versions), `esqueleto` **3.6.0.3** (uploaded 2026-08-19), and `postgresql-simple` **0.7.0.1**, which `persistent-postgresql` builds on (`>=0.6.1 && <0.8`).
- **Primary sources:** the published sources. Hackage tarballs (`hackage.haskell.org/package/<name>-<ver>/<name>-<ver>.tar.gz`), the dbmate tag tarball for `v2.37.0` with its README and release notes, and `lib/pq` from the Go module proxy. Citations look like `persistent-postgresql 2.14.3.0: Database/Persist/Postgresql.hs:544` and point into those tarballs. The same files are on Hackage under "Source" and on GitHub at the tag. ⚠️ marks anything I inferred from code and did not run against a database.

This note reports facts and makes no decision.

## Answer in brief

- **dbmate can run the 12 files with SQL bodies unchanged, but not the files as they stand.** Every dbmate file must contain a `-- migrate:up` line *and* a `-- migrate:down` line, or parsing fails. None of the Drizzle files has either. The body itself runs as one simple-protocol `Exec`, so multi-statement files, the `DO $$…$$` block and `--> statement-breakpoint` comments all pass through. Each file runs in its own transaction, and its version (the leading digits, `0000`…`0011`) is recorded in `schema_migrations (version varchar primary key)` in the same transaction.
- **`dbmate dump` leaves the grants out.** It runs `pg_dump --schema-only --no-privileges --no-owner` and appends the `schema_migrations` rows, so `schema.sql` has no `GRANT`/`REVOKE`. A database built with `dbmate load` therefore has none of the `dds_app` grants. The role isn't in it either, because `pg_dump` never dumps roles.
- **persistent works without its migrator.** `mkMigrate` is optional, and `getMigration` and `migrateEntitiesStructured` return the SQL or a structured diff that persistent *would* run, without running it. That diff compares only tables, columns (type name, nullability, default text), **unique** constraints by name and FKs by name and cascade. It ignores CHECKs, non-unique indexes, grants, sequences and enum labels. An empty diff needs entity declarations that spell out this schema's names: `constraint=` for FK names, `sql=` for unique names, `sqltype=` for most column types and `OnDeleteNoAction OnUpdateNoAction` on every FK.
- **Several column types this schema uses have no built-in instance and need a hand-written `PersistField`**: Postgres enums, `uuid`, `inet`, and `text[]`/`char(2)[]` for writes. `timestamptz` ↔ `UTCTime`, `date` ↔ `Day`, `jsonb` ↔ aeson `Value` (or Esqueleto's `JSONB a`), and natural or non-`id` single-column keys are supported directly. `derivePersistField` doesn't fit Postgres enums: it writes `show` output and can't read the value back.
- **Esqueleto covers `FOR UPDATE [OF …] [SKIP LOCKED | NOWAIT]` and `ON CONFLICT … DO UPDATE/NOTHING … RETURNING`.** It has **no `UPDATE … RETURNING` or `DELETE … RETURNING`**: `updateCount` and `deleteCount` return only a row count, so anything more goes through `rawSql`. persistent's own `insert` uses `INSERT … RETURNING <id>`. Advisory locks have no API. Transaction-scoped locks are plain `rawSql`. A session-scoped lock can be held on one pooled connection with `runSqlPoolNoTransaction`. `getSimpleConn` exposes the underlying `postgresql-simple` `Connection`.

## 1. What the schema and the queries use

The inventory in `docs/research/seaorm-existing-schema.md` §1 still holds. I re-read `apps/api/src/db/migrations/0000`–`0011` and the query code. The facts below are the ones this question turns on.

| Feature | Where |
|---|---|
| 9 Postgres enums, one of them extended by `ALTER TYPE … ADD VALUE` | 0001, 0004, 0007, 0008, 0010 |
| `timestamp with time zone` everywhere; `date` | all tables; `request.start_date`/`end_date` |
| `text[]`, `char(2)[]` | `request.report_codes`, `request.provinces` (0004) |
| `jsonb DEFAULT '{}'::jsonb` and nullable `jsonb` | `request_event.payload`, `reviewer_event.payload` (0001); `extraction_job.result` (0007) |
| `inet` | `request_event.ip`, `reviewer_event.ip`, `token_lookup.ip`, and as the **primary key** of `download_throttle` (0008) |
| `uuid DEFAULT gen_random_uuid()` PKs | `request`, `reviewer`, `reviewer_session`, `extraction_job`, `download_token`, `mail_delivery` |
| `bigserial` PKs | `request_event`, `reviewer_event`, `token_lookup` |
| **Non-`id` / natural single-column PKs** | `province.province_id char(2)`, `request_contact.request_id uuid` (also the FK to `request`), `login_throttle.key text`, `download_throttle.ip inet` |
| Constant PK | `scheduler_heartbeat.id smallint DEFAULT 1` with `CHECK (id = 1)` |
| `smallint` | `province.health_region`, `scheduler_heartbeat.id` |
| Standalone sequence | `request_reference_seq`; the app calls `nextval('request_reference_seq')` (`requests.service.ts:91`) |
| FK names | Drizzle-style, e.g. `request_contact_request_id_request_id_fk`; all `ON DELETE no action ON UPDATE no action` |
| Unique names | `request_reference_unique`, `reviewer_username_unique`, `reviewer_session_token_hash_unique`, `download_token_token_hash_unique` |
| Role, grants, a `DO $$…$$` block | 0001 (role), 0001–0009 (grants) |
| **Not used** | **citext, composite keys**, triggers, views, functions, generated columns, extensions |

How the Node code queries it today (`apps/api/src`):

- **`SELECT … FOR UPDATE`**: `download-throttle.ts:30`, `login-throttle.ts:58`, `alerts.service.ts:154`, `download-tokens.repository.ts:47`, `in-flight.service.ts:205`, `reviewer-accounts.ts:254`. `decisions.service.ts:99` uses `FOR UPDATE OF request` inside a join.
- **`SKIP LOCKED`**: nowhere today. The ticket lists it for the queue the Port may add.
- **Upserts**: `ON CONFLICT (key|ip|id) DO UPDATE`, where the conflict target is the table's **primary key** (`login-throttle.ts:67`, `download-throttle.ts:35`, `scheduler/tick.ts:239`). `ON CONFLICT (username) DO NOTHING RETURNING id` targets a **unique** column (`reviewer-accounts.ts:102`).
- **`RETURNING`**: on inserts (`extraction-jobs.repository.ts:28`, `mail-delivery.repository.ts:21`, `requests.service.ts:111`, …), on updates (`move-request-state.ts:22`, `reviewer-auth.ts:184,293`, `download-tokens.repository.ts:61,86`, …) and on a delete that returns three columns (`reviewer-sessions.ts:175`).
- **Advisory locks**: `pg_advisory_xact_lock(64)` (`reviewer-accounts.ts:151`), `pg_advisory_xact_lock(hashtextextended(…, 0))` (`requests.service.ts:67`), and the session-level `pg_try_advisory_lock` / `pg_advisory_unlock` pair on one pooled client (`scheduler/tick.ts:130,137`). The tests take a session-level `pg_advisory_lock` around migrations (`test/support/scratch-database.ts:33`).

## 2. dbmate over the 12 files

**File format.** `pkg/dbmate/migration.go` parses each file:

- The file must contain a line matching `^--\s*migrate:up` (`upRegExp`, line 62). Otherwise `ErrParseMissingUp`: "dbmate requires each migration to define an up block" (line 73).
- Each section must also contain `-- migrate:down` (`parseMigrationSection`, lines 107–112). Otherwise `ErrParseMissingDown`. The down block may be empty, which is what `dbmate new` writes (README, "Creating Migrations").
- Before the first `migrate:up`, only blank lines and `--` comment lines may appear. Anything else gives `ErrParseUnexpectedStmt` (`statementsPrecedeMigrateBlocks`, lines 187–198). The leading comment headers of 0010 and 0011 are therefore allowed.
- Options follow the directive on the same line. The only one is `transaction:false`, and transactions are on by default (lines 55–59, README "Options").
- Several `migrate:up`/`migrate:down` pairs per file are allowed. Since v2.37.0 the version is recorded once, after the last section (release notes, #824; `db.go:429–446`).

**The Drizzle files as they stand.** None of `0000`–`0011` contains `-- migrate:up` or `-- migrate:down`, so dbmate rejects every one of them. To run them, each file needs at least the two directive lines. Drizzle's `--> statement-breakpoint` markers, including the ones glued to the end of a statement (`…;--> statement-breakpoint`), start with `--`. They are SQL comments to Postgres and plain text to dbmate's parser.

**How a body runs.**
- `Migrate` calls `tx.Exec(migrationSection.Up)` with no arguments (`db.go:436`).
- dbmate's Postgres driver is `lib/pq` (`go.mod`: `github.com/lib/pq v1.12.3`; `pkg/driver/postgres/postgres.go:24,203`). `lib/pq`'s `conn.Exec` sends an argument-less query through `simpleExec`, the simple-query protocol (`lib/pq@v1.12.3/conn.go:954–960`).
- Postgres runs several `;`-separated statements in one simple query, `DO $$…$$` included. **A file body therefore runs as written, with no splitting.** ⚠️ I did not run the 12 files through dbmate.

**Transactions.** Each section runs inside `doTransaction` unless it is marked `transaction:false` (`db.go:450–456`). The `schema_migrations` insert happens inside the same transaction (`db.go:441–446`). Drizzle, by contrast, runs all pending files in one transaction (prior note §1). `0010`'s `ALTER TYPE … ADD VALUE` inside a transaction is allowed on PG ≥ 12, but the new labels can't be used until commit, and 0010 doesn't use them (prior note §3).

**Versions and tracking.**
- A file is a migration if its name matches `^(\d+).*\.sql$`, and the leading digits are its version (`db.go:36`, `findMigrations` lines 544–553). `0004_request.sql` is version `0004`.
- Files run in filename order (lines 563–567).
- The README says only the version is recorded, "so you can safely rename a migration file".
- The table is `create table if not exists <schema>.schema_migrations (version varchar primary key)` (`postgres.go:351–386`). Its name can be changed with `--migrations-table` / `DBMATE_MIGRATIONS_TABLE`.
- Without `--strict`, a missing lower version is applied out of order. `--strict` fails instead (`db.go:396–412`).

**Concurrent runs.**
- v2.37.0 adds `--migration-lock` (`DBMATE_MIGRATION_LOCK`), with a 5-minute default timeout. It holds `pg_advisory_lock(48372615)` on one dedicated connection for the whole run and fails if that connection is lost (`postgres.go:545–640`; README; release notes #848).
- The README notes that it needs a direct or session-pooling connection, and that it does not cover the race between concurrent `dbmate up` calls that create the database.
- The tests' own migration lock key is `6_425_001` (`scratch-database.ts`), so the two keys don't collide.
- Without the flag there is no lock, and two runners race on `CREATE ROLE` exactly as they do under Drizzle today.

**`dbmate dump` / `schema.sql`.**
- `DumpSchema` runs `pg_dump --format=plain --encoding=UTF8 --schema-only --no-privileges --no-owner`. Since pg_dump 15.14 / 16.10 / 17.6 it also passes `--restrict-key=dbmate` to keep the output deterministic. It then appends `INSERT INTO schema_migrations (version) VALUES …` (`postgres.go:255–307`). Extra `pg_dump` flags can be appended (`dbmate dump -- --flag`), but the defaults above can't be removed.
- The consequences for this schema:
  - **No `GRANT`/`REVOKE` lines** (`--no-privileges`) and no `CREATE ROLE`, because `pg_dump` never emits roles. The grant structure the e2e suites assert (`test/audit-spine.e2e-spec.ts:391`, `test/scheduler-tick.e2e-spec.ts:766,787`) is not in `schema.sql`.
  - `dbmate load` replays `schema.sql` with one `Exec` (`db.go:274`). A database built that way has the tables, enums, CHECKs and sequence, but no grants and possibly no role.
  - `request_reference_seq`, the CHECKs, the enums and the indexes *are* in the dump, because they are schema objects.
- `up`, `migrate` and `rollback` rewrite the dump automatically (`AutoDumpSchema: true`) unless `--no-dump-schema` is given. If `pg_dump` is missing, the dump is silently skipped. `pg_dump` must be at least the server's version (README "Exporting Schema File"). The official Docker image installs `postgresql-client` (`Dockerfile:13,34`).

**Baseline off `drizzle.__drizzle_migrations`.**
- dbmate treats a version as applied if and only if a `schema_migrations` row holds it (`findMigrations`, `db.go:519–556`). Nothing in dbmate reads Drizzle's table.
- Drizzle records `(id serial, hash text, created_at bigint)`, where `created_at` is the journal's `when` and the hash is of the file contents (prior note §1; `meta/_journal.json`). It doesn't record the tag.
- Adopting a Drizzle-migrated database would therefore mean inserting `('0000')…('0011')` into `schema_migrations`, matching on journal `idx`, `when` or file hash. dbmate has no command for this.
- **ADR 0024's context, which ADR 0025 doesn't change on this point, says that every Port database starts empty and that "no database migrated by Drizzle is ever adopted."** On that premise no baseline step is needed.

**Rollback.** `dbmate rollback`/`down` runs the down section of the newest applied version (`db.go:574–660`). An empty down block "succeeds" and deletes the version row while leaving the schema in place. ⚠️ This is inferred from the code: an empty `Exec` followed by `DeleteMigration`.

## 3. persistent with dbmate owning the schema

**Not migrating.** Entities come from `share [mkPersist sqlSettings] [persistLowerCase|…|]`. `mkMigrate "name"` / `migrateModels` is a separate generator, and nothing runs a migration unless the code calls `runMigration*` (`persistent 2.18.1.0: Database/Persist/Sql/Migration.hs`; `Database/Persist/TH/Internal.hs:3056–3063, 3274–3298`). Two more switches:
- the `!no-migrate` entity attribute takes an entity out of `migrateModels` (`Database/Persist/Quasi.hs:355–364`);
- `MigrationOnly` marks a column that exists in SQL but not in the Haskell record (`Quasi.hs:366–383`).

**Reading a diff without running it.**
- `getMigration :: Migration -> ReaderT SqlBackend m [Sql]`, `showMigration` and `printMigration` return or print the statements a migration *would* run (`Migration.hs:127–147`). `parseMigration` returns them with an unsafe flag per statement (`CautiousMigration`).
- `persistent-postgresql` also exposes `migrateEntitiesStructured`, which returns `Either [Text] [AlterDB]` as data rather than SQL text (`Database/Persist/Postgresql/Internal/Migration.hs:50–69`, `@since 2.14.1.0`), and `mockMigration` for a database-less print (`Postgresql.hs:838–841`).
- Running one of these against a dbmate-migrated database gives a drift signal: an empty list means persistent sees no difference. **What persistent can see** is set by `collectSchemaState` (`Internal/Migration.hs`):

| Compared | How | Source |
|---|---|---|
| Columns | `information_schema.columns`: name, nullability, `COALESCE(domain_name, udt_name)` as the type (plus `(n)` when `character_maximum_length` is set), `column_default`, `generation_expression` | `Internal/Migration.hs:241–316` |
| Column type | `getType` maps only `int4 int8 varchar text date bool timestamptz float4 float8 bytea time numeric`. Anything else becomes `SqlOther "<udt_name>"`. The comparison is case-insensitive over `showSqlType` and treats `int8`/`bigint` as aliases | `:333–346`, `sqlTypeEq :766–779` |
| Defaults | Exact text equality after stripping a trailing `::text` / `::character varying`. A `nextval…` default on the old column is never reported | `:322–331`, `findAlters :1156–1162`; the Quasi docs warn about canonicalisation (`Quasi.hs:95–116`) |
| Unique constraints | `information_schema.key_column_usage ⨝ table_constraints` excluding PRIMARY KEY and FOREIGN KEY, compared **by name**. A DB unique constraint the entity lacks is reported as `DropConstraint` unless its name starts with `__manual_` | `:403–423`, `getAltersU :740–757` |
| Foreign keys | Read from `pg_constraint`, compared by constraint name, target table and cascade | `equivalentRef :781–794`, `parseCascade :516–531` |
| Extra DB columns | A DB column with no entity field is reported as `Drop` | `getAltersC :731–733` |
| **Not compared** | CHECK constraints (they have no `key_column_usage` rows), non-unique indexes, grants, roles, standalone sequences, **enum labels** (`pg_enum` is never read), the PK's identity beyond its column | absence in `collectSchemaState` |

**Matching this schema's names and types in entity declarations** (all syntax from `Quasi.hs`):

- **FK names**: persistent's default is `<table>_<column>_fkey` (`Sql/Internal.hs:222–224`; `Internal/Migration.hs:802–812`). Drizzle's are `…_<col>_<reftable>_<refcol>_fk`, so each reference needs `constraint=<drizzle name>` (`Quasi.hs:417–429`).
- **FK cascade**: the DB reports `ON DELETE/UPDATE no action` as `Just NoAction` (`parseCascade "a"`). An undeclared cascade is `Nothing`, and `eqCascade Nothing (Just NoAction)` is `False`; only `Nothing`/`Restrict` are treated as equal (`:788–793`). So every FK field needs `OnDeleteNoAction OnUpdateNoAction` to compare equal. Those attributes parse from `"On" <> "Delete"|"Update" <> <constructor>` (`Quasi/Internal.hs:1273–1284`), and `NoAction` exists since persistent 2.18.0.0 (ChangeLog).
- **Unique names**: `UniqueX col sql=<name>` sets the constraint name (`Quasi.hs:251–258`). The four Drizzle names would go there.
- **Types**: `showSqlType (SqlOther t) = t` (`:1100`), and the DB side is the `udt_name`. ⚠️ From the code I read, this schema's columns would compare equal only with `sqltype=` values that spell the `udt_name`: `int2` for `smallint`, `bpchar(2)` for `char(2)`, `_text` / `_bpchar` for the arrays, `uuid`, `inet`, `jsonb`, and the enum type name such as `request_state`. `Text` (VARCHAR vs `text`, both `SqlString`), `Int32`/`Int64`, `Bool`, `Day` and `UTCTime` (`SqlDayTime` ↔ `timestamptz`) compare equal without one. I did not run `getMigration` against a migrated database.

**Primary keys.**
- **Implicit `id`**: the default is `Int64` with `SERIAL8` (`maySerial`, `:719–721`), which matches the three `bigserial` tables. The `nextval` default is never reported as drift.
- **A custom-typed `id`**: `Id UUID default=gen_random_uuid()`. With a `default=`, `insert` omits the column and Postgres generates it. Without one you must `insertKey` (`Quasi.hs:118–143, 226–243`). The `Id` line also takes `sql=<col>` and `sqltype=` (`Quasi.hs:226–250`).
- **A natural key**: `Primary field [field…]` (`Quasi.hs:145–171`). A single field is allowed, which covers `province_id`, `request_contact.request_id`, `login_throttle.key`, `download_throttle.ip` and `scheduler_heartbeat.id`. For a natural key, `insert` uses `ISRManyKeys`: the key comes from the record's own values, and no `RETURNING` is issued (`persistent-postgresql: Postgresql.hs:544–551`).
- No table here has a composite key.

**DB defaults on non-key columns.** `default=` is DDL only. The Haskell record still has the field, so "you still need to provide a value when performing an `insert`" (`Quasi.hs:84–92`). persistent's insert lists every non-generated field (`insertSql'`, `Postgresql.hs:544–566`). Columns such as `recorded_at DEFAULT now()`, `created_at DEFAULT now()`, `status DEFAULT 'queued'` and `attempts DEFAULT 0` get whatever value the application supplies. The Quasi docs' workaround is a `Maybe` field. Esqueleto's `insertSelect` with `now_` is the other route (§4).

## 4. How the column types map

Wire path, for orientation:
- **Writes.** `postgresql-simple` interpolates parameters into the query text on the client: `execute` is `exec conn =<< formatQuery conn template qs`, and `Escape` becomes a single-quoted literal through `PQescapeStringConn` (`postgresql-simple 0.7.0.1: Database/PostgreSQL/Simple.hs:314–316`; `Internal.hs:591–617`). A `Text` value therefore arrives as an *untyped* literal, which Postgres coerces to the column's type (enum, `inet`, `uuid`, …).
- **Reads.** `persistent-postgresql` decodes by result-column OID through `builtinGetters`. Any OID not in that table falls back to `PersistLiteralEscaped <raw bytes>` (`Internal.hs:131–214`).

| Postgres | Haskell side | Read gives | Notes |
|---|---|---|---|
| `timestamptz` | `UTCTime` (`SqlDayTime`) | `PersistUTCTime` | Built in (`Internal.hs:152`; `Sql/Class.hs:15225`) |
| `date` | `Day` | `PersistDay` | Built in |
| `text` | `Text` | `PersistText` | Built in |
| `char(2)` | `Text` | `PersistText` (`bpchar` OID) | `sqltype=bpchar(2)` for a clean diff (§3) |
| `smallint` / `integer` / `bigint` | `Int16` / `Int32` / `Int64` | `PersistInt64` | `Int16`'s `sqlType` is `SqlInt32`, i.e. `INT4` (`Sql/Class.hs:15201`), so a `smallint` column needs `sqltype=int2` |
| `boolean` | `Bool` | `PersistBool` | Built in |
| `jsonb` | aeson `Value` (orphan instance from `Database.Persist.Postgresql.JSON`) or Esqueleto's `JSONB a` | `PersistByteString` | Both write `PersistLiteralEscaped (encode x)` and declare `SqlOther "JSONB"` (`persistent-postgresql: JSON.hs:330–357`; `esqueleto: PostgreSQL/JSON/Instances.hs:25,88–99`) |
| `uuid` | **no instance in persistent or persistent-postgresql** | `PersistLiteralEscaped` (OID 2950 is not a builtin) | `Sql/Class.hs:15145–15160` gives a UUID instance as the documentation example, with `SqlOther "uuid"`. That example matches on `PersistLiteralEncoded`, a pattern that doesn't exist in 2.18.1.0. The patterns are `PersistLiteral`, `PersistLiteralEscaped` and `PersistDbSpecific` (`PersistValue.hs:130–153`) |
| `inet` | **no instance** | `PersistLiteralEscaped` (OID 869 is not a builtin) | Hand-written `PersistField` with `sqltype=inet`. It is also `download_throttle`'s PK type |
| `text[]`, `char(2)[]` | **no exported instance that writes an array** | `PersistList [PersistText…]` (OIDs 1009, 1014 → `listOf`) | The overlappable `PersistField [a]` writes `PersistList`, which `persistent-postgresql` sends as a **JSON string** (`toField (P (PersistList l)) = toField (listToJSON l)`, `Internal.hs:79`). That doesn't parse as a Postgres array literal ⚠️. `PersistArray` is sent as `ARRAY[…]`, or `'{}'` when empty (`Internal.hs:84`; `postgresql-simple ToField.hs:297–304`). `persistent-postgresql`'s `PostgresArray` newtype writes `PersistArray`, but it isn't exported from `Database.Persist.Postgresql.JSON` and its reader accepts only `PersistArray`, never the `PersistList` a read returns (`JSON.hs:399–404`). The `[a]` instance's `sqlType` is `SqlString` (`Sql/Class.hs:15229`) |
| Postgres enum (e.g. `request_state`) | **hand-written `PersistField`** | `PersistLiteralEscaped "<label>"` (an enum's OID is not a builtin) | `Sql/Class.hs:15163–15178` documents `sqlType _ = SqlOther "rainbow_color"` for a `CREATE TYPE … AS ENUM`. Writing `PersistText "<label>"` or `PersistLiteralEscaped` reaches the server as an untyped literal and coerces to the enum. `derivePersistField` (`TH/Internal.hs:3161–3193`) writes `PersistText (show x)`, the Haskell constructor name rather than the snake_case label, and reads through the `Text` instance, whose `fromPersistValueText (PersistLiteral_ _ _) = Left "Cannot convert PersistLiteral to Text"` (`PersistValue.hs:213`). So it fails on enum columns |
| `citext` | not used here | — | — |

**Sequences.** persistent has no sequence API. `nextval('request_reference_seq')` is a `rawSql "SELECT nextval('request_reference_seq')" []` returning `[Single Int64]`, or an Esqueleto `unsafeSqlFunction`. Neither tool creates or checks the sequence. It lives only in the dbmate SQL and in `schema.sql`.

**Enum labels in two places.** A new label means `ALTER TYPE … ADD VALUE` in a dbmate file *and* a constructor plus its label in the Haskell instance. `getMigration` doesn't read `pg_enum` (§3), so it won't catch a mismatch. Reading a row whose label the instance doesn't know fails in `fromPersistValue`. A test that compares `SELECT enumlabel FROM pg_enum …` with the Haskell constructors is the kind of check that would catch it. That is an option, not something either library provides.

## 5. Esqueleto features for these queries

All from `esqueleto 3.6.0.3`.

| Need | Esqueleto | Source |
|---|---|---|
| `FOR UPDATE` | `locking forUpdate`. The `LockingKind` constructors `ForUpdate`/`ForUpdateSkipLocked` are deprecated in 3.6.0.0 in favour of the smart constructors `forUpdate` and `forUpdateSkipLocked` | `Internal/Internal.hs:1676–1720`, rendered at `:3574–3575` |
| `FOR UPDATE SKIP LOCKED` | `locking forUpdateSkipLocked`, exported from `Database.Esqueleto`, `.Experimental` and `.Legacy` | `Esqueleto.hs:87`; `Experimental.hs:76` |
| `FOR UPDATE OF t [NOWAIT \| SKIP LOCKED]` | `Database.Esqueleto.PostgreSQL.forUpdateOf entities skipLocked\|noWait\|wait`; also `forNoKeyUpdateOf`, `forShare`, `forShareOf`, `forKeyShareOf` | `PostgreSQL.hs:30–40, 615–660`; rendered at `Internal.hs:3592–3599` |
| `INSERT … ON CONFLICT (cols) DO UPDATE … RETURNING *` | `PostgreSQL.upsert` / `upsertBy uniqueKey record (NonEmpty updates)` → `Entity record` | `PostgreSQL.hs:272–296, 397–418` |
| `… DO NOTHING RETURNING *` | `upsertMaybe` / `upsertMaybeBy` with `[]` updates → `Maybe (Entity record)`, which is `Nothing` when the row already existed (since 3.6.0.0) | `PostgreSQL.hs:298–395` (the `DO NOTHING RETURNING ??` rewrite at `:380–387`) |
| `INSERT … SELECT … ON CONFLICT` | `insertSelectWithConflict`, `insertSelectWithConflictCount` | `PostgreSQL.hs:420–505` |
| `INSERT … RETURNING id` | persistent's `insert` (`ISRSingle (sql <> " RETURNING " <> id)`) | `persistent-postgresql: Postgresql.hs:544–551` |
| `UPDATE … RETURNING`, `DELETE … RETURNING` | **None.** `update`/`delete` return `()`, and `updateCount`/`deleteCount` return the `Int64` row count (`Experimental.hs:199–201`). The only `RETURNING` in the Esqueleto source is the upsert's | grep over `src/` |
| Raw SQL | `rawSql` (persistent), `unsafeSqlFunction`, `unsafeSqlValue` | `Experimental.hs:447` |

How upserts pick their conflict target:
- The upsert SQL is `ON CONFLICT (<unique's columns>)`: it names **columns**, not a constraint (`persistent-postgresql: upsertSql'`, `Postgresql.hs:569–590`). So constraint names don't matter at runtime.
- Esqueleto's `upsertBy` needs a `Unique record`, which is a `UniqueX` line in the entity.
- Three of this schema's upserts target the **primary key** (`login_throttle.key`, `download_throttle.ip`, `scheduler_heartbeat.id`). For those the options in the libraries are:
  - persistent's `repsert`/`repsertMany`, whose SQL is `ON CONFLICT (<key fields>) DO UPDATE SET <every field> = EXCLUDED.<field>`, a full-row replace (`repsertManySql`, `Postgresql.hs:884–888`);
  - a `Unique` declared over the PK column. ⚠️ `getMigration` would then expect a separate UNIQUE constraint by that name, because PRIMARY KEY is excluded from its constraint read (§3);
  - `rawSql`.
- `reviewer-accounts.ts:102`'s `ON CONFLICT (username) DO NOTHING RETURNING id` maps onto `upsertMaybeBy (UniqueReviewerUsername u) r []`.

## 6. Raw connection and advisory locks

- **Raw statements in a persistent transaction.** `runSqlPool action pool` takes one connection, wraps the action in `BEGIN`…`COMMIT`, and rolls back on an exception (`persistent: Sql/Run.hs:22–40`). `rawExecute` and `rawSql` inside it run on that connection. `SELECT pg_advisory_xact_lock(?)` and `pg_advisory_xact_lock(hashtextextended(?, 0))` therefore work as a plain `rawSql`/`rawExecute` at the start of the action, and the lock is released at commit. This matches the two Node uses.
- **The underlying connection.** `getSimpleConn :: BackendCompatible SqlBackend backend => backend -> Maybe PG.Connection` (`persistent-postgresql: Postgresql.hs:479–485`, `@since 2.13.0`). It returns the `postgresql-simple` `Connection` stored in the backend's vault by `createBackend` (`:497–503`). Inside a `ReaderT SqlBackend` action, `asks getSimpleConn` therefore gives the same connection, inside the same transaction.
- **Session-scoped locks** (the tick's `pg_try_advisory_lock` / `pg_advisory_unlock`). Lock and unlock must run on one connection.
  - `runSqlPoolNoTransaction action pool Nothing` holds one pooled connection for the whole action and issues no `BEGIN`/`COMMIT` (`Sql/Run.hs:52–61`, `@since 2.12.0.0`). With `postgresql-simple`'s default autocommit, each statement commits on its own.
  - `withPostgresqlConn` / `openSimpleConn` give a dedicated, unpooled `SqlBackend` (`Postgresql.hs:28–67`).
  - ⚠️ I did not check how a transaction *inside* a `runSqlPoolNoTransaction` block is best written (for example `postgresql-simple`'s `withTransaction` on `getSimpleConn`, or `transactionSave`). Nothing in the source forbids either.
- **Lock keys.** The fixed keys (64, 7_215_003, and 6_425_001 in the tests) don't depend on the ORM. dbmate's own migration-lock key is `48372615` (§2).

## 7. Facts the deciding tickets will need

1. dbmate needs `-- migrate:up` and `-- migrate:down` in every file. The Drizzle files have neither, but their bodies run unchanged as one simple-protocol `Exec` each, one transaction per file.
2. `schema.sql` from `dbmate dump` contains no grants and no role, so `dbmate load` doesn't reproduce `dds_app`'s privileges.
3. `--migration-lock` (new in v2.37.0) serialises concurrent `dbmate up` runs. Without it, the `CREATE ROLE` race the tests guard against today remains.
4. No baseline is needed if, as ADR 0024 states, no Drizzle-migrated database is ever adopted. If one were, it would be a hand-written `INSERT INTO schema_migrations`.
5. persistent runs no migration unless asked. `getMigration` / `migrateEntitiesStructured` give a drift diff over columns, types, nullability, default text, unique names and FK names/cascade, and nothing else. A clean diff takes `constraint=`, `sql=`, `sqltype=` and `OnDeleteNoAction OnUpdateNoAction` throughout.
6. Hand-written `PersistField` instances are needed for the 9 enums, `uuid`, `inet` and the two array columns. `derivePersistField` doesn't work for Postgres enums, and the default list instance writes JSON, not an array.
7. Non-key `DEFAULT`s are not applied by persistent inserts. The application supplies those values.
8. Esqueleto has `FOR UPDATE [OF] [SKIP LOCKED|NOWAIT]` and upsert-with-`RETURNING`, but no `UPDATE`/`DELETE … RETURNING`. Upserts on a primary key go through `repsert`, a declared `Unique`, or raw SQL.
9. Transaction-scoped advisory locks are a `rawSql` inside `runSqlPool`. Session-scoped ones fit `runSqlPoolNoTransaction` or a dedicated connection, and `getSimpleConn` exposes the raw `postgresql-simple` connection.
