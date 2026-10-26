# Rust job queues and schedulers for the Port

- **Question:** What are the maintained Rust options for durable background jobs and a recurring tick under axum: Postgres-backed queues and Redis-backed ones? For each: retries and backoff, delayed jobs, concurrency limits, job inspection or a dashboard comparable to bull-board, SeaORM/sqlx coexistence, maturity.
- **Issue:** rawinan-soma/dds-sharing-FP#2 (child of #1; blocks #6, "Queue and scheduled work in the Port")
- **Date:** 2026-10-26
- **Primary sources:** crates.io API (versions, dates, dependency ranges, owners), GitHub API (stars, last push, contributors), crate READMEs and docs.rs pages, crate source and migrations on GitHub, the PostgreSQL manual. Every claim below names the source it came from. Anything a source did not say is marked **not stated**.

This note lists facts. It recommends nothing. #6 makes the decision.

## What the current service relies on

Read from `apps/api/src` at `64fc877`. These are the behaviours a replacement has to reproduce or consciously drop.

- **Two BullMQ queues, `extraction` and `mail`.** In both, the Postgres row id is the BullMQ job id (`extraction-queue.ts`, `mail-queue.ts`). This is how the reconcile asks whether a Postgres row has a live job.
- **Postgres is the record, Redis does the execution.** `extraction-queue.ts` says "Postgres, not Redis, is the durable record of what ran". `tick.ts` says "Redis executes; Postgres is the truth."
- **Mail has no backoff inside the queue.** `mail-queue.ts` enqueues with `attempts: 1`. A failed job stays in the failed set, and the tick calls `job.retry('failed')` once its 15-minute delay is up (`MAIL_MAX_ATTEMPTS = 5`, `MAIL_RETRY_DELAY_MS = 15 min`). The comment gives the reason: "a backoff living only in Redis is a schedule a Redis loss silently cancels."
- **The rendered message lives in Redis.** A sent Delivery job is kept for the life of its Download token, and the raw token exists nowhere else (`mail-queue.ts`). Losing the job means the mail cannot be rebuilt, only abandoned.
- **Extraction runs at concurrency 1.** `EXTRACTION_CONCURRENCY = 1` in `extraction.config.ts`, and the worker has a stall guard (`stall-guard.ts`).
- **One recurring tick.** It runs every 60 s via `@nestjs/schedule` `setInterval`, with a `startup` pass first. Each pass takes a Postgres advisory lock (`TICK_LOCK_KEY`). The tick is stateless: "due" is a query, so a restart needs no catch-up (`scheduler.module.ts`, `tick.ts`).
- **bull-board runs on its own listener.** It is not on the app port and has no auth of its own (`bull-board.ts`, spec §14.4).

## Version baseline: SeaORM and sqlx

Source: crates.io API.

| crate | latest | sqlx it depends on |
|---|---|---|
| `sea-orm` | 2.0.4 (2026-09-27); 2.0.0 was 2026-07-19 | `^0.9.0` |
| `sea-orm` 1.x line | 1.1.20 (2026-03-31) | `^0.8.4` |
| `sqlx` | 0.9.0 (2026-05-21) | — |

- A queue crate that pins **sqlx 0.8** and runs next to **SeaORM 2.x** (sqlx 0.9) puts two sqlx major versions in the build. Their `PgPool` types are different types, so the two cannot share a pool. (This follows from Cargo semver; no crate document says it.)
- SeaORM exposes its pool as `DatabaseConnection::get_postgres_connection_pool(&self) -> &PgPool`, and builds a connection from one with `impl From<PgPool> for DatabaseConnection` (docs.rs, sea-orm 2.0.4). So a queue that takes a sqlx 0.9 `PgPool` can share SeaORM's pool.
- The `DatabaseConnection` docs page shows **no** accessor from a SeaORM `DatabaseTransaction` to the underlying sqlx transaction or connection. Not confirmed from source. The practical effect: a queue whose transactional enqueue takes `&mut PgConnection` or a sqlx `Executor` cannot obviously join a transaction SeaORM opened. Two ways around it: (a) open the transaction in sqlx and write SeaORM's rows with raw SQL, or (b) enqueue with plain SQL through SeaORM's statement API, which needs the queue's enqueue to be expressible as SQL. Notes on (b) are in each Postgres section below.

## Postgres-backed queues

### apalis + apalis-postgres

- **Maturity** (crates.io, GitHub API):
  - `apalis` 1.0.0 was released 2026-10-12, after rc.1–rc.12 across 2026. The crate dates from 2021.
  - `apalis-postgres` 1.0.0 was released 2026-10-12 as a new crate (created 2025-10-25). It replaces the Postgres part of `apalis-sql`, whose last stable is 0.7.4.
  - Main repo: 1,470 stars, last push 2026-10-25. One maintainer dominates: geofmureithi has 440 commits, and the next human has 27. MIT licence.
  - Sibling repos: `apalis-postgres` has 8 stars and `apalis-redis` has 16.
- **sqlx:** `apalis-postgres` 1.0.0 depends on `sqlx ^0.9`, the same major as SeaORM 2.x. `PostgresStorage::new(&pool)` takes an existing `PgPool` (docs.rs).
- **Schema** (GitHub source, `migrations/`):
  - Everything lives in schema `apalis`. The job table is `apalis.jobs`, with columns `id TEXT` (ULID), `job_type`, `status`, `attempts`, `max_attempts DEFAULT 25`, `run_at`, `lock_at`, `lock_by`, `done_at`, `priority`, `metadata`, and `idempotency_key`.
  - The migrations run `CREATE EXTENSION IF NOT EXISTS pgcrypto`. Migration `20260826101335_metadata_hstore.sql` makes `metadata` an `hstore`, so the database must allow both extensions.
  - Setup is `PostgresStorage::setup(&pool)`, or merge `PostgresStorage::migrations()` into your own sqlx `Migrator`.
- **Claiming:** `apalis.get_jobs()` selects `... FOR UPDATE SKIP LOCKED` and sets `status = 'Queued'`. It picks rows where `status = 'Pending'`, or where `status = 'Failed' AND attempts < max_attempts`, and `run_at < now()` (migration `20260902075949`). Wake-ups come from LISTEN/NOTIFY via `.with_pubsub()` or from polling (`poll_with_strategy`).
- **Retries and backoff:**
  - Retries are a tower `retry::Policy` layer: `RetryPolicy::retries(n)`, `.with_backoff(...)` (`BackoffRetryPolicy`, using tower's `ExponentialBackoffMaker`), `.retry_if(pred)`, and `FromTaskConfigPolicy` for per-task `RetryConfig` (docs.rs `apalis::layers::retry`; source `apalis/src/layers/retry/mod.rs`).
  - The backoff is a future that the worker's service stack awaits. In the Postgres backend, `handle_result.sql` writes `status`, `attempts`, `last_result`, and `done_at`. It does **not** move `run_at`. So a backoff delay is not stored in the database, and a `Failed` row with attempts left can be fetched again on the next poll. (This is my reading of the SQL; the docs do not describe how the two interact.)
- **Delayed jobs:** `TaskBuilder::new(arg).run_after(Duration)` and `.priority(n)` (docs.rs).
- **Concurrency:** `WorkerBuilder::...concurrency(n)` (docs.rs). The `limit`, `timeout`, and rate-limit layers are listed as features; their configuration is **not stated** on the page.
- **Custom job id / idempotency:**
  - There is a unique index on `apalis.jobs(id)`.
  - Migration `20260508093314` added `idempotency_key` with a unique index on `(job_type, idempotency_key)`.
  - The public API for setting either is **not stated** in the docs. The docs.rs page shows only `PgTaskId = TaskId<Ulid>`.
- **Transactional enqueue:** `apalis_postgres::queries::push_tasks(&mut *tx, "queue", items)` inside a sqlx transaction (docs.rs example). The SQL is a plain `INSERT INTO apalis.jobs ... SELECT unnest(...)` (`queries/task/sink.sql`). The payload is `bytea` from apalis's codec, so a hand-written SQL enqueue from SeaORM would have to reproduce that encoding.
- **Inspection:** the `queries` module includes `list_tasks`, `list_queues`, `list_workers`, `fetch_by_id`, and `metrics` (GitHub source tree). Dashboard: see apalis-board below.
- **Other:** heartbeats and re-enqueue of orphaned jobs, graceful shutdown, `prometheus`, `opentelemetry`, and `sentry` features, and `SteppedFlow` workflows (docs.rs).

### apalis-board (dashboard for apalis)

Sources: README and crates.io.

- 1.0.0 released 2026-10-12. 75 stars. Credits bull-board and trigger.dev as references.
- Crates: `apalis-board-api` (API for axum, actix, and salvo behind feature flags), `apalis-board-web` (Leptos UI built with trunk), and `apalis-board-types`.
- You mount it with `ApiBuilder` under a path such as `/api/v1` and serve the UI with `ServeApp`. Live logs come over SSE at `/api/v1/events` through a `TracingBroadcaster`.
- **Backend table:** apalis-postgres, -sqlite, and -mysql are supported. apalis-redis carries a warning marker. apalis-cron is unsupported. The README does not define its markers.
- Views: Tasks, Task, Workers, Queues. Which job actions exist (retry, kill, payload view) is **not stated**. **No authentication** is described, which matches bull-board's current posture of a separate listener with no auth.

### graphile_worker (graphile_worker_rs)

- **Maturity** (crates.io, GitHub API):
  - 0.14.1 released 2026-10-11, with frequent releases in Oct 2026. Created 2024-01.
  - 87 stars, last push 2026-10-23. Effectively one maintainer: leo91000 has 415 commits. MIT licence.
  - It is a Rust rewrite of Node's Graphile Worker, described as "mostly compatible", and can run against the same schema (README).
- **sqlx:** depends on `sqlx ^0.9.0`. `WorkerOptions::pg_pool(...)` takes an application-owned pool. A `tokio-postgres` driver is optional (configuration guide, crate map).
- **Claiming:** `SKIP LOCKED` plus LISTEN/NOTIFY, with a `poll_interval` fallback (default 1 s). Optional local queue for batch fetch (README, configuration guide).
- **Retries and backoff** (scheduling guide):
  - On failure the job's `run_at` is set to `greatest(now(), run_at) + exp(least(attempts, 10)) seconds`. The backoff is **stored in the row**.
  - `max_attempts` defaults to **25**. When `attempts` reaches `max_attempts`, the job is no longer fetched.
- **Delayed jobs:** `run_at` in `JobSpec`. `WorkerUtils::reschedule_jobs` moves existing jobs (docs.rs).
- **Concurrency:** `WorkerOptions::concurrency(n)`, which defaults to the number of logical CPUs. **Serial queues:** jobs that share a `queue_name` "must not run in parallel"; the worker locks the queue when it fetches one (scheduling guide).
- **Job keys / dedupe:** `job_key` with modes `Replace`, `PreserveRunAt`, and `UnsafeDedupe`. Lower `priority` runs first (scheduling guide). `remove_job(job_key)` (docs.rs).
- **Transactional enqueue:** `WorkerUtils::with_executor(E)` returns `WorkerUtilsWithExecutor<E>`, so job operations "take part in an existing transaction"; the caller commits (docs.rs). Its method list is **not stated** on that page. The README says jobs can also be added from SQL via `graphile_worker.add_job`, which is the route a SeaORM-opened transaction could use.
- **Cron built in** (cron guide):
  - Crontab lines look like `* * * * * task ?opts {payload}`. Five fields, so the finest resolution is **one minute**. UTC by default.
  - Options: `id`, `fill` (backfill window), `max`, `queue`, `priority`, `job_key`, and `job_key_mode`.
  - Missed ticks are backfilled only within `fill`, and only for entries the worker already knew.
  - How duplicate cron jobs are prevented across several worker instances is **not stated** in the Rust guide.
- **Inspection UI** (operations guide):
  - `graphile_worker_admin_ui` is an embedded **Axum** server with a Leptos UI. Run it with `graphile-worker admin` (CLI, default `127.0.0.1:5678`) or embed it through `AdminServerConfig`.
  - Views: overview, queues, workers, job list, and single job. Actions: complete, permanently fail, run now, reschedule, remove by key, plus migrate, cleanup, force unlock, and sweep stale workers.
  - **Auth is built in:** Basic, Bearer, custom header, or none. "No-auth mode is rejected on non-loopback listen addresses." There is a read-only mode, CSRF on writes, and a CSP.
- **Migrations:** `graphile-worker migrate` requires PostgreSQL 12+. It rejects databases that carry newer breaking migrations. One migration fails while jobs are locked (operations guide).

### sqlxmq

- **Maturity:**
  - 0.6.0 released 2025-05-25. The previous release was 0.5.0 on 2023-10-15. Last push 2025-05-25.
  - 167 stars, 11 open issues and PRs. One maintainer, Diggsey (38 commits). Apache-2.0/MIT (crates.io, GitHub API).
- **sqlx:** `^0.8`, which is behind SeaORM 2.x's 0.9 (crates.io).
- **Features** (README):
  - Exponential-backoff retries with configurable count and initial backoff.
  - Delayed jobs.
  - `set_concurrency(min, max)`.
  - Channels with opt-in strict ordering and fair delivery.
  - Send and complete jobs inside other transactions.
  - Checkpointing for long jobs, with keep-alive so a running job is not retried mid-run.
  - A `#[job]` macro.
  - NOTIFY-based polling.
- **Schema:** tables and stored procedures prefixed `mq`, shipped as migration files you copy into your own migrations (README).
- **Inspection:** none mentioned.

### underway

- **Maturity:**
  - 0.2.0 released 2025-07-16, the only release after 0.1.2 (2024-11). Last push 2026-10-09.
  - 176 stars, 17 open issues and PRs. maxcountryman has 140 commits (crates.io, GitHub API).
- **sqlx:** `^0.8.2` (crates.io).
- **Features** (README): typed multi-step workflows, retries, cron (`"0 2 * * *[UTC]"`), heartbeats and fencing, existing `PgPool`, and transactional `enqueue_using` / `schedule_using`.
- Backoff detail, delayed jobs, concurrency limits, and a UI are **not stated**.

### fang

- 0.11.0 released 2026-07-02, after 0.11.0-rc1 in 2024-04 and 0.10.4 in 2023-08 (crates.io).
- 724 stars. ayrat555 and pxp9 maintain it.
- Depends on `sqlx ^0.8` and has a `cron` dependency. Its features were not read for this note.

### Others found but thin

crates.io search for "postgres job queue" on 2026-10-26 turned up `postmodern`, `asyncq-postgres`, `runledger-*`, `axeon-jobs`, `stampeded`, and `agenda-rs`. Each has under ~3k total downloads, and several are under 200. `backie` was last released 2023-07. `pgmq`, a Postgres-extension message queue, is on 0.34.0-alpha (2026-09) and depends on sqlx 0.9. None of these was studied further.

### Hand-rolled `FOR UPDATE SKIP LOCKED`

- PostgreSQL manual, SELECT, "The Locking Clause": "With `SKIP LOCKED`, any selected rows that cannot be immediately locked are skipped. Skipping locked rows provides an inconsistent view of the data, so this is not suitable for general purpose work, but can be used to avoid lock contention with multiple consumers accessing a queue-like table." `SKIP LOCKED` is not allowed together with `WITH TIES`.
- This is the same primitive that `apalis.get_jobs()` and graphile_worker use (see their sections).
- The current service already keeps a Postgres row per extraction job (`extraction_job`) and per mail delivery (`mail_delivery`), and already drives mail retry timing from the tick. A hand-rolled queue would claim those rows directly.
- What you would have to build yourself, judging by what the libraries above ship:
  - a lock timeout, heartbeat, or orphan recovery (the current stall guard);
  - LISTEN/NOTIFY or polling;
  - a persisted backoff (`run_at`);
  - a concurrency cap;
  - an inspection view.
- SeaORM can run this directly, either through its query builder with a row lock or through raw SQL. No second sqlx is involved.

## Redis-backed queues

### bullmq-official (the official BullMQ Rust port)

- **Maturity:**
  - Published by **Taskforce.sh**, BullMQ's maintainer. The crates.io owner is `manast` (Manuel Astudillo). The source is `taskforcesh/bullmq/rust` in the main BullMQ repo.
  - The crate name is `bullmq-official`; the library is imported as `bullmq`. Created 2026-07-12. Latest is 1.4.3 (2026-10-23), with releases every few days in Oct 2026.
  - Needs Rust 1.85+ and Redis 6.2+ (crates.io, `rust/Cargo.toml`, `rust/README.md`).
- **Compatibility:** "Fully compatible with Node.js and Elixir BullMQ queues (same Lua scripts and Redis data structures)" (README). The Redis keyspace and job states would stay what they are today, including job id = Postgres row id, `removeOnComplete` / `removeOnFail` with age and count, and `job.retry('failed')`.
- **Features** (`rust/FEATURE_PARITY.md`, updated 2026-06-18):
  - Implemented: delay, priority, attempts, backoff (fixed, exponential, custom), custom `jobId`, removeOnComplete/removeOnFail, deduplication, repeat/cron (`JobScheduler` via `upsert_job_scheduler`), worker concurrency (dynamic too), stalled-job detection, lock renewal, rate limiting, `get_job` / `get_jobs` / state getters, QueueEvents, and Prometheus export.
  - Not implemented: Sentinel, OpenTelemetry, legacy repeatable API, `wait_until_finished`, and sandboxed processors.
- **Inspection:** the crate ships no UI. Because the Redis layout is shared, Node **bull-board** pointed at the same Redis should still read these queues. That is an inference from the compatibility claim, not tested. It would mean keeping a Node process just for the board.
- **sqlx/SeaORM:** not involved, since it is Redis-only (depends on `redis ^1.7.1`).
- Side note: BullMQ's Node source tree now contains a `src/postgres` backend and Postgres test configs (GitHub tree, `vitest.postgres.config.ts`). The Rust crate depends only on `redis`. Postgres-backend status was not investigated further.

### apalis-redis

- 1.0.0 released 2026-10-12 (crates.io). Same apalis worker, layers, and retry `Policy` as above. Depends on `redis ^1.7.1`.
- apalis-board marks Redis with a warning marker; see apalis-board above.

### rusty-sidekiq (`sidekiq-rs`)

- 0.15.2 released 2026-09-28. 167 stars. film42 has 203 commits (crates.io, GitHub API).
- "Compatible with sidekiq.rb for both submitting and processing jobs" (README).
- Periodic jobs come from cron strings kept in a Redis sorted set. `unique_for` gives a "good enough" lock via `SET NX EX`. `num_workers` defaults to the CPU count.
- Retry and backoff details and a UI are **not stated**. The README says "Sidekiq.rb is obviously much more mature than this repo."

### Others

- `bullmq-rs` (bogardt) is an unrelated third-party crate: 2.2.0, ~2.9k downloads.
- `oxanus` is 2.0.0 (2026-06) with 110 recent downloads.

## Recurring-tick options

### `tokio::time::interval`

Source: tokio docs.rs, `MissedTickBehavior`.

- The first tick resolves immediately.
- Behaviour on a missed tick:
  - `Burst` (the default) "ticks as fast as possible until caught up".
  - `Delay` re-spaces later ticks one period from the late tick.
  - `Skip` drops missed ticks and stays on the original grid.
- Nothing is persisted, and there is no cross-instance coordination. The current Node tick is the same shape: a `setInterval` plus an advisory lock taken per pass.

### tokio-cron-scheduler

- 0.15.1 released 2025-10-28. 742 stars, 39 open issues and PRs. mvniekerk has 112 commits. ~7M downloads (crates.io, GitHub API).
- Cron syntax has **six fields, including seconds**, parsed by `croner`. It also has `Job::new_one_shot(Duration)` and `Job::new_repeated(Duration)`. Timezones via `_tz` variants and `chrono-tz` (README).
- In-memory by default. Optional `postgres_storage` uses **tokio-postgres**, not sqlx, and there is a `nats_storage` option (crates.io dependencies, README).
- Distributed single-run and catch-up of missed runs are **not stated**.

### Queue-native cron

- **apalis-cron** 1.0.0 (2026-10-12):
  - `CronScheduler` is a backend that produces tasks from a schedule, using the `cron` crate.
  - `with_timezone(...)` sets the timezone.
  - `pipe_to(storage)` persists ticks into a storage backend, with stated purposes "Distribute cronjobs between multiple servers" and "Prevent task skipping in the case of a restart".
  - Backfill is **not stated**. apalis-board marks cron unsupported (docs.rs, apalis-board README).
- **graphile_worker crontab:** five-field (minute) resolution, `fill` backfill, `job_key` dedupe. See its section above.
- **bullmq-official `JobScheduler`:** `upsert_job_scheduler` for cron or interval repeatable jobs, stored in Redis (FEATURE_PARITY.md).
- **underway:** `schedule_using` with a cron expression (README).

## Summary table

"stored" means the backoff delay is written to the database or Redis. "in-process" means the worker waits it out in memory.

| option | store | latest (date) | sqlx | retry/backoff | delayed | concurrency | UI | cron |
|---|---|---|---|---|---|---|---|---|
| apalis + apalis-postgres | PG (`apalis` schema; pgcrypto, hstore) | 1.0.0 (2026-10-12) | 0.9 | tower `RetryPolicy` + backoff, in-process | `run_after` | `.concurrency(n)` | apalis-board (axum, no auth stated) | apalis-cron |
| graphile_worker | PG (`graphile_worker` schema) | 0.14.1 (2026-10-11) | 0.9 | stored `run_at`, `exp(min(attempts,10))` s, default 25 | `run_at` | `concurrency(n)` + serial `queue_name` | admin UI (axum, auth built in) | crontab, minute resolution |
| sqlxmq | PG (`mq_*` + procs) | 0.6.0 (2025-05-25) | 0.8 | exponential, configurable | yes | `set_concurrency(min,max)` | none | not stated |
| underway | PG | 0.2.0 (2025-07-16) | 0.8 | retries (detail not stated) | not stated | not stated | none | yes |
| hand-rolled SKIP LOCKED | PG (existing tables) | — | via SeaORM | build it | build it | build it | build it | the tick |
| bullmq-official | Redis (BullMQ layout) | 1.4.3 (2026-10-23) | — | fixed/exp/custom, stored | `delay` | worker concurrency, global concurrency, rate limit | Node bull-board (inferred) | `JobScheduler` |
| apalis-redis | Redis | 1.0.0 (2026-10-12) | — | as apalis | as apalis | as apalis | apalis-board (warning marker) | apalis-cron |
| rusty-sidekiq | Redis (Sidekiq layout) | 0.15.2 (2026-09-28) | — | not stated | not stated | `num_workers` | not stated | periodic |
| tokio interval | none | tokio | — | — | — | — | — | `MissedTickBehavior` |
| tokio-cron-scheduler | memory / tokio-postgres / NATS | 0.15.1 (2025-10-28) | — (tokio-postgres) | — | one-shot | — | — | six-field cron |

## Open points this note did not settle

- How `apalis-postgres` 1.0 lets a caller set a task's `id` or `idempotency_key`. The columns exist; the builder API was not found in the docs.
- Whether a SeaORM 2.x `DatabaseTransaction` can hand out its sqlx connection. If it cannot, the transactional-enqueue paths above need a raw-SQL or sqlx-first workaround.
- Whether Node bull-board actually works unmodified against queues written by `bullmq-official`.
- How graphile_worker_rs prevents duplicate cron jobs when several worker instances run.
