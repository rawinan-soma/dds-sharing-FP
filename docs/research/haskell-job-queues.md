# Haskell job queues and schedulers for the Port

- **Question:** What are the maintained Haskell options for durable background jobs and a recurring tick beside Servant? Postgres-backed queues and Redis-backed ones (is anything compatible with the BullMQ Redis layout?). For each: retries and backoff, delayed jobs, concurrency limits, heartbeats, inspection or a dashboard, coexistence with `persistent-postgresql` (shared pool or separate, hasql vs postgresql-simple underneath), maturity. Also: how to hold a session advisory lock on one pinned connection from a persistent pool for the tick.
- **Issue:** rawinan-soma/dds-sharing-FP#36 (child of #35; blocks #41)
- **Date:** 2026-11-03
- **Primary sources:** Hackage (package index, `preferred` versions, `upload-time`, `.cabal` files, changelogs, Haddock pages, release tarballs), Stackage snapshot `cabal.config` files (`lts-24.65`, `nightly-2026-10-28`), GitHub API (stars, last push, contributors, file trees) and repo sources/READMEs, the Faktory wiki, the PostgreSQL 18 manual. Every claim below names the source it came from. Anything a source did not say is marked **not stated**.

This note lists facts. It recommends nothing. #41 makes the decision.

## What the current service relies on

The behaviours a replacement has to reproduce are listed in [`rust-job-queues.md`](rust-job-queues.md#what-the-current-service-relies-on) and still hold. The ones that bear on the Haskell choice, re-read from `apps/api/src` and `docs/spec.md` at `432d1bd`:

- **Postgres is the record; the queue executes** (spec §7.7, §15.3). A startup reconcile re-enqueues `queued`/`running` extraction jobs with no live queue job, and never touches `pending`.
- **Retry timing lives in Postgres, not the queue.** Mail is enqueued with `attempts: 1`, `jobId: mailDeliveryId` (`mail-queue.ts`); the tick decides when a failed send is due again (§15.3).
- **A sent Delivery's queue job is kept for 72 h**, the life of its Download token, because the rendered message is the only place the raw token exists (§10.8). Whatever store holds the job holds the token.
- **Extraction concurrency is 1** (§7.7) and a stall guard fails a job that makes no progress (`stall-guard.ts`, §7.6).
- **The tick:** one pass every 60 s; each pass runs `SELECT pg_try_advisory_lock($1)` on one checked-out `pg` client, runs the pass, then `pg_advisory_unlock` on the **same** client before releasing it (`scheduler/tick.ts`). That is a **session-level** lock on a pinned connection. A pass with a failed job writes no heartbeat; the heartbeat goes stale after 5 minutes (§15.3). The business-hours clock and the collection lapse are derived by queries on each pass, so there is no schedule state to persist (§15.3).
- **The prior Rust decision** (#6, as revised by #16): Redis with `bullmq-official`, keeping the BullMQ layout; bull-board dropped (`docs/port-build-plan.md` slice 5).

## Version baseline: persistent, esqueleto, the drivers

Source: Hackage `preferred` + `upload-time`; Stackage `cabal.config`.

| package | latest on Hackage (upload) | in `lts-24.65` | in `nightly-2026-10-28` |
|---|---|---|---|
| `persistent` | 2.18.1.0 (2026-01-14) | 2.17.1.0 | 2.18.1.0 |
| `persistent-postgresql` | 2.14.3.0 (2026-01-14) | 2.13.7.0 | 2.14.3.0 |
| `esqueleto` | 3.6.0.3 (2026-08-19) | 3.6.0.3 | 3.6.0.3 |
| `resource-pool` | 0.5.1.0 (2026-08-15) | 0.4.0.0 | 0.5.1.0 |
| `postgresql-simple` | 0.7.0.1 (2025-08-02) | 0.7.0.1 | 0.7.0.1 (via persistent-postgresql) |
| `hasql` | 2.0.1.0 (2026-08-12) | 1.9.3.1 | 2.0.1.0 |
| `hasql-pool` | 1.5.1.1 (2026-11-02) | 1.3.0.5 | 1.5.0.1 |
| `hedis` | 0.16.4 (2026-09-08) | 0.15.2 | 0.16.4 |

- **persistent-postgresql sits on postgresql-simple.** Its cabal description says "Based on the postgresql-simple package". 2.14.3.0 depends on `postgresql-simple >=0.6.1 && <0.8` and `postgresql-libpq` (`.cabal`).
- **Its pool is a resource-pool `Pool SqlBackend`.** `type ConnectionPool = Pool SqlBackend` (`Database/Persist/Sql/Types.hs`, persistent 2.18.1.0).
- **The raw connection is reachable.** `getSimpleConn :: BackendCompatible SqlBackend backend => backend -> Maybe PG.Connection`, "@since 2.13.0", returns the underlying postgresql-simple `Connection` (`Database/Persist/Postgresql.hs`, 2.14.3.0). So a postgresql-simple queue API that takes a `Connection` can be called **inside the same transaction** persistent opened. It returns `Maybe`, `Nothing` only when the backend "isn't backed by postgresql-simple".
- **A hasql-based queue cannot share that pool or transaction.** hasql has its own `Connection` type over libpq, and hasql-pool its own pool. There is no conversion from a postgresql-simple `Connection` in either library's API (not found in either's docs). A hasql queue therefore means a second pool and separate transactions. (Inference from the types; no document says it.)
- **hasql 2.0 is a breaking major.** Its changelog: "`Hasql.Connection.acquire` now takes an explicit adapter as its first argument"; the transport is pluggable via `pqi`, with an alpha pure-Haskell backend. 1.10 was also a "major revision". Packages bounded `hasql ^>=1.10` (below) build against neither `lts-24.65` (1.9.3.1) nor the nightly (2.0.1.0).
- **Esqueleto has row-lock clauses but no advisory-lock helper.** `Database.Esqueleto.PostgreSQL` has `forUpdateOf`, `forNoKeyUpdateOf`, `forShareOf`, `forKeyShareOf`, each taking an `OnLockedBehavior` of `skipLocked`, `noWait` or `wait` ("@since 3.5.9.0"). The older `ForUpdateSkipLocked` constructor is deprecated in 3.6.0.0. A grep of the esqueleto, persistent and persistent-postgresql tarballs finds no "advisory".

## Postgres-backed queues

### odd-jobs

- **Maturity** (Hackage, GitHub API):
  - Latest **Hackage** release is **0.2.2, uploaded 2020-07-07**. Four releases in all (0.1.0 to 0.2.2), no metadata revisions.
  - The repo's `CHANGELOG.md` has an unreleased **0.2.3**: "Upgrade resource pool", "Add ability to kill jobs". `package.yaml` on `master` still says `version: 0.2.2`. An issue titled "Release a new version" has been open since 2021-06-30.
  - Last commit on `master` 2025-10-30 ("qualify text imports to support ghc 9.10.2"); repo last push 2026-07-03. 78 stars, 38 open issues and PRs. Contributors: saurabhnanda 83, philderbeast 37, jappeace 23. BSD-3-Clause.
  - Not in `lts-24.65` (Stackage returns 404) and not in the nightly `cabal.config`.
  - Hackage 0.2.2 builds its default pool with `createPool` (`ConfigBuilder.hs` in the tarball). resource-pool 0.5.1.0 still exports `createPool`, marked `DEPRECATED "Use newPool instead"` (`src/Data/Pool.hs`). Master uses `newPool`/`defaultPoolConfig`. Whether 0.2.2 builds against current Servant/aeson/GHC is **not stated**; nobody has published a build report, and this note did not test it.
- **Driver and pool:** postgresql-simple. `cfgDbPool :: Pool Connection` (postgresql-simple `Connection`) in both 0.2.2 and master `Types.hs`. That is a separate pool from persistent's `Pool SqlBackend`. A wrapper pool could hand out `getSimpleConn` results, but the library does not provide one.
- **Enqueue:** `createJob :: ToJSON p => Connection -> TableName -> p -> IO Job` and `scheduleJob ... -> UTCTime -> IO Job` (`OddJobs/Job.hs`). The `Connection` argument "should ideally come out of your application's DB pool, not the `cfgDbPool`". With `getSimpleConn` that can be the connection of a running persistent transaction.
- **Schema:** one table with `id serial primary key`, `run_at`, `status text`, `payload jsonb`, `last_error jsonb`, `attempts`, `locked_at`, `locked_by`, a CHECK constraint `incorrect_locking_info`, and an `AFTER INSERT` trigger that does `pg_notify` (`OddJobs/Migrations.hs`). The library creates it at runtime with `createJobTable :: Connection -> TableName -> IO ()`. The job id is its own serial, so "job id = the app's row id" (as in BullMQ today) is not available; the app id would ride in the payload.
- **Claiming:** LISTEN/NOTIFY for jobs due now, plus a poller every `cfgPollingInterval` (default 5 s) using `UPDATE ... SELECT ... FOR UPDATE` (`Job.hs`, `ConfigBuilder.hs`). Whether the select uses `SKIP LOCKED` is **not stated** in the comments I read.
- **Retries and backoff:** `cfgDefaultMaxAttempts = 10`, `cfgDefaultRetryBackoff = \attempts -> pure $ Seconds $ 2 ^ attempts` (`mkConfig` in `ConfigBuilder.hs`). On failure the job's `run_at` is set to now plus the backoff, with status `Retry`, or `Failed` once attempts reach the max (`retryOrFail`, `Job.hs`). The **backoff is stored in the row**.
- **Delayed jobs:** `scheduleJob` with a `UTCTime`. The docs warn: picked up "with an error of +/- `cfgPollingInterval` seconds".
- **Concurrency:** `ConcurrencyControl` = `UnlimitedConcurrentJobs` | `MaxConcurrentJobs Int` | `ResourceLimits ResourceCfg` | `DynamicConcurrency (IO Bool)` (`Types.hs`). The limit is per runner process: `MaxConcurrentJobs` counts that process's own job threads.
- **Heartbeat / stuck jobs:** no heartbeat. A `Locked` job whose `locked_at` is older than `cfgDefaultJobTimeout` (default **600 s**) is picked up again by the poller. The runner also kills the job thread with a `TimeoutException` after that timeout (`runJobWithTimeout`, `Job.hs`). That is a duration cap, not progress-based stall detection.
- **Inspection:** a built-in admin UI written with **Servant + Lucid** (`OddJobs/Endpoints.hs`, `Web.hs`). Routes include filtered listing, `enqueue`, `run`, `cancel`, `kill` (kill is master-only) and refresh of job types/runners. Authentication is **not stated**. The routes can be served on their own Warp port or mounted in an app.
- **Other:** structured JSON logging, graceful shutdown, a CLI/daemon helper (`OddJobs.Cli`, using `hdaemonize`), immediate or delayed deletion of successful jobs (`cfgImmediateJobDeletion`, `cfgDelayedJobDeletion`).
- **Cron:** none built in.

### hasql-queue

- **Maturity:** 1.2.0.2 uploaded **2020-07-06**; five releases; no revisions (Hackage). Repo last push 2021-07-28, 18 stars, 13 open issues. BSD-3-Clause. Author Jonathan Fischoff. Not in Stackage LTS.
- **Driver:** hasql, with **no version bound** in the cabal file, plus `postgresql-libpq-notify`. `tested-with: GHC ==8.8.1`. Whether it builds against hasql 1.10 or 2.0 is **not stated**.
- **API:** modules `Hasql.Queue.{High,Low}.{AtLeastOnce,AtMostOnce,ExactlyOnce}` and `Hasql.Queue.Migrate` (`.cabal`). `enqueue :: Text -> Connection -> Value a -> [a] -> IO ()`; `withDequeue :: Text -> Connection -> Value a -> Int -> Int -> ([a] -> IO b) -> IO b` (`Low.AtLeastOnce` Haddock).
- **Retries:** `withDequeue` takes a retry count. Payloads attempted more often than that are marked "failed", and `failures` lists them. Backoff, delayed jobs, a concurrency setting, heartbeats and a UI are **not stated**.
- **Transactional enqueue:** the README shows `enqueue` inside a hasql `transaction`. As above, that cannot be a persistent transaction.

### postgresql-simple-queue

- 1.0.1 uploaded **2017-12-21** (Hackage). Repo `jfischoff/postgresql-queue` last push 2020-01-21, 16 stars. postgresql-simple based; the description shows enqueue "as part of larger database transaction". Not studied further.

### pgmq-hs family (`pgmq-hasql`, `pgmq-effectful`, `pgmq-inspect`, `pgmq-migration`, `pgmq-config`, `pgmq-core`)

- **What PGMQ is** (pgmq repo, `pgmq-extension/README.md`): a Postgres message queue in SQL functions (`pgmq.send`, `pgmq.read`, `pgmq.delete`, `pgmq.archive`, `set_vt`). It "can be run on any existing Postgres instance or installed as a Postgres Extension". Delivery is "exactly once ... within a visibility timeout": a read message is invisible for `vt` seconds and reappears if not deleted or archived. Each row carries `read_ct`. `pgmq/pgmq`: 5,370 stars; latest release v1.13.1 (2026-10-20).
- **Maturity** (Hackage, GitHub API):
  - All six Haskell packages are at **0.7.0.0, uploaded 2026-10-26**, 13 releases (fewer for `pgmq-config` and `pgmq-inspect`). MIT.
  - Repo `shinzui/pgmq-hs`: **0 stars**; contributors shinzui 262 commits, one other with 1.
  - Only `pgmq-core` is in `nightly-2026-10-28`; none is in `lts-24.65`.
- **Driver:** hasql. `pgmq-hasql` depends on `hasql ^>=1.10`, `hasql-transaction ^>=1.2` (tests use `hasql-pool ^>=1.4`) (`.cabal`). That is **neither** the LTS hasql (1.9) nor the nightly one (2.0). A separate pool from persistent's.
- **Features** (README): supports PGMQ 1.12/1.13; send with `delay`; read with visibility timeout and batch size; FIFO reads; headers; topic routing; LISTEN/NOTIFY throttles. `pgmq-migration` installs the PGMQ schema "without requiring the pgmq extension", but "does not ship a migration runner of its own". `pgmq-effectful` adds OpenTelemetry tracing.
- **Retries and backoff:** PGMQ has none of its own. A message re-appears when its visibility timeout lapses. Backoff would be the consumer's `set_vt` / `delay` choice.
- **Inspection:** `pgmq-inspect` is a **read-only** embeddable WAI app and standalone executable over HTTP and WebSocket: queue listings, depth metrics, browsing live and archived messages "without leasing them", lookup by id, health probes, configurable CORS (`.cabal` description, README). Authentication is **not stated**.
- **Workers on top:** see shibuya and keiro below.

### shibuya + shibuya-pgmq-adapter; keiro-pgmq

- **shibuya** (`shinzui/shibuya` README): "Supervised queue processing for Haskell, inspired by Broadway". Bounded inboxes, explicit ack decisions, keyed batching, configurable ordering. Current release 0.12.0.0, "pre-1.0". Targets **GHC 9.12** (`base ^>=4.21`) and uses `effectful-core`. 24 stars; last push 2026-10-27. `shibuya-metrics` adds HTTP/JSON, Prometheus, WebSocket metrics and health probes.
- **shibuya-pgmq-adapter** 0.18.0.0 (2026-10-27), 16 releases, repo 0 stars. "Visibility-timeout-based leasing, automatic retry handling, and optional dead-letter queue support", with an in-process API to browse and "atomic redrive" a dead-letter queue (README). Depends on `hasql ^>=1.10`, `hasql-pool ^>=1.4`, `pgmq-hasql ^>=0.7` (`.cabal`). Backoff formula, delayed-job API and per-queue concurrency knobs are **not stated** in the README.
- **keiro-pgmq** 0.23.0.0 (2026-10-31), 23 releases: "A typed background-job queue for Keiro applications on top of PGMQ ... and shibuya". You declare a `Job` "bundling a queue, a payload codec, and a retry/dead-letter policy", and the handler is `p -> Eff es JobOutcome` (`.cabal`). `tested-with: ghc >=9.12 && <9.13`; `hasql >=1.10 && <1.11`; depends on `keiro-core` (the Keiro framework). Repo 9 stars.
- All three have the same author as pgmq-hs (Nadeem Bitar) and are effectively single-maintainer.

### pgqueuer-hs

- 0.0.1.0 uploaded 2026-07-12, the only release. Repo `tusharad/pgqueuer-hs`: 2 stars, last push 2026-07-20. MIT (Hackage, GitHub API).
- A Haskell port of Python's `pgqueuer` that "uses the **exact same schema**", so jobs can cross between Python and Haskell (README).
- Driver: `postgresql-simple >=0.7 && <0.9` (`.cabal`). Whether it can use persistent's connection is **not stated**.
- Features (README): transactional enqueue, `FOR UPDATE SKIP LOCKED`, per-entrypoint and global concurrency limits, LISTEN/NOTIFY dispatch, deferred jobs via `execute_after`, `dedupe_key`, a log of status transitions, and "failed jobs can be retried or held for manual inspection". Backoff, heartbeat and a UI are **not stated**.

### Others found but thin

From a name search of the Hackage package index on 2026-11-03 (`queue|job|worker|cron|schedul|bull|sidekiq|faktory|pgmq|advisory|...`):

- `haskell-pgmq` and `haskell-bee-pgmq` 0.1.0.0 (2025-06-16, one release each). Gargantext; **AGPL-3.0-or-later**; postgresql-simple.
- IHP's job runner lives inside the `ihp` framework. `ihp-job-dashboard` 1.5.0 (2026-03-25) depends on `ihp` and `hasql`. Not separable from IHP as packaged.
- `immortal-queue` 0.2.0.0 (2025-12-12) is a worker-pool abstraction over any push/pop queue; it brings no storage.
- `yesod-job-queue` (2017), `jobs-ui` (2019), `Spock-worker`, `yesod-worker` (2016). Not studied.

### Hand-rolled `SKIP LOCKED` through persistent/esqueleto

- PostgreSQL manual, SELECT, the locking clause: "With `SKIP LOCKED`, any selected rows that cannot be immediately locked are skipped ... can be used to avoid lock contention with multiple consumers accessing a queue-like table." (Quoted in full in [`rust-job-queues.md`](rust-job-queues.md#hand-rolled-for-update-skip-locked).)
- Esqueleto 3.6 writes this as `forUpdateOf row skipLocked` (`Database.Esqueleto.PostgreSQL`). It runs on persistent's own pool and transaction, so no second driver or pool is needed.
- The current service already has a Postgres row per extraction job (`extraction_job`) and per mail delivery (`mail_delivery`), and the tick already drives mail retry timing. What a hand-rolled queue has to build, judging by what odd-jobs ships: a persisted `run_at` backoff, a lock timeout or heartbeat (the current stall guard), LISTEN/NOTIFY or polling, a concurrency cap, and an inspection view. persistent-postgresql exposes the postgresql-simple connection (`getSimpleConn`), so postgresql-simple's `LISTEN` / `getNotification` are reachable from it.

## Redis-backed queues

### BullMQ layout compatibility

- **No Haskell BullMQ client was found.** No Hackage package name contains "bull" except unrelated physics/pushbullet ones (package index, 2026-11-03). A GitHub repository search for `bullmq language:haskell` returns 0 results.
- BullMQ's own repo ships ports in `rust/`, `python/`, `elixir/` and `php/` directories, and none for Haskell (GitHub tree of `taskforcesh/bullmq`).
- **Hand-rolling it would mean driving BullMQ's Lua scripts.** The repo has 64 top-level scripts in `src/commands/*.lua` plus 72 in `src/commands/includes/` (GitHub tree). hedis 0.16.4 has the primitives: `eval`, `evalsha`, `scriptLoad`, `scriptExists` (`Database/Redis/Commands.hs`, `ManualCommands.hs`). How the other ports load and version those scripts was not studied.

### hedis (the Redis client)

- 0.16.4 uploaded 2026-09-08, 86 releases. Repo `informatikr/hedis`: 336 stars, last push 2026-09-08. BSD-3-Clause. In both Stackage snapshots (0.15.2 in LTS, 0.16.4 in nightly).

### hworker

- 0.1.0.1 uploaded 2017-01-01, with one metadata revision. Repo now `positiondev/hworker`: 38 stars, last push 2024-12-30. ISC.
- "A Redis-backed persistent at-least-once queue". A job returns `Success`, `Failure` or `Retry`, and `Retry` puts it back on the queue. A monitor thread re-queues jobs that ran past a timeout (README). Backoff, delays and a UI are **not stated**. Its own Redis layout, not BullMQ's.

### redis-job-queue

- 0.1.0, uploaded 2015-09-21, the only release (Hackage). Not studied.

### faktory (needs the Faktory server, not Redis directly)

- **Client/worker:** `faktory` 1.2.0.0 uploaded 2026-07-23, 28 releases. Freckle Engineering; MIT. Repo 17 stars, last push 2026-10-27. In `lts-24.65` (1.1.3.2) and the nightly (1.2.0.0). 1.2.0.0 requires `resource-pool >= 0.3.0` (changelog).
- "This package contains only the client and worker parts." The server is a separate daemon (README).
- **Server:** `contribsys/faktory`, Go, 6,155 stars, latest v1.10.0 (2026-08-10). "Faktory is AGPL licensed" (LICENSE). "Contains a comprehensive Web UI for management and monitoring" (README).
- **Job options** (`Faktory/JobOptions.hs`): `retry :: Int`, `once`, `queue`, `at :: UTCTime` (and an `in` offset via `joSchedule`), `reserveFor :: Natural`, `custom`.
- **Retries** are server-side (Faktory wiki, *Job Errors*): default 25 retries "over 21 days", wait `15 + count ^ 4 + (rand(30) * (count + 1))` seconds, then the Dead Set, from which jobs "can be manually executed from the Web UI". `"retry": 0` discards on failure. "Any jobs left over by a worker crash" are re-enqueued "after the job reservation times out".
- Coexistence with persistent: none needed (separate server). The service would gain a new process and an AGPL server.

## Recurring-tick options

- **A plain loop.** `forever (pass >> threadDelay 60_000_000)` in a thread started with `async`/`withAsync` next to Warp. Nothing is persisted and there is no cross-instance coordination; the advisory lock provides that, as it does today. (`threadDelay` and `async` are base/async; no further source needed.)
- **`cron`** 0.7.2 (2024-11-09), MichaelXavier. In both Stackage snapshots. "Cron data structure and Attoparsec parser". `System.Cron.Schedule` has `execSchedule` with `addJob action "* * * * *"`, which forks a thread per job (Haddock). That is five fields, so one-minute resolution. In memory, with no lock or catch-up.
- **Queue-native cron:** odd-jobs, hasql-queue and pgmq have none. Faktory's is Enterprise-only: "Faktory Enterprise can enqueue jobs periodically" from a `cron.toml` (Faktory wiki, *Ent Cron*).
- Spec §15.3 already makes "due" a query on every pass, so none of these needs to persist a schedule.

## Holding a session advisory lock on one pinned persistent connection

- **What the lock is** (PostgreSQL 18 manual, 13.3.5 "Advisory Locks"): "Once acquired at session level, an advisory lock is held until explicitly released or the session ends. Unlike standard lock requests, session-level advisory lock requests do not honor transaction semantics: a lock acquired during a transaction that is later rolled back will still be held following the rollback, and likewise an unlock is effective even if the calling transaction fails later." Transaction-level locks "are automatically released at the end of the transaction". `pg_try_advisory_lock(key bigint) → boolean` "will either obtain the lock immediately and return true, or return false without waiting" (9.28 System Administration Functions).
- **Pinning a connection.** `Data.Pool.withResource :: Pool a -> (a -> IO r) -> IO r` (resource-pool 0.5.1.0) takes one `SqlBackend` out of persistent's `ConnectionPool` for the whole callback. Every `runSqlConn action backend` inside that callback runs on that same session.
- **Each `runSqlConn` is its own transaction.** `runSqlConn r conn = with (acquireSqlConn conn) $ runReaderT r`. `acquireSqlConn` begins a transaction and commits on release, or rolls back on exception (`Database/Persist/Sql/Run.hs`, persistent 2.18.1.0). By the manual quote above, a session lock taken in one `runSqlConn` survives that transaction's commit or rollback. So the pass can run several transactions on the pinned backend while holding it.
- **No-transaction runner.** `runSqlPoolNoTransaction` ("@since 2.12.0.0") runs without `BEGIN`/`COMMIT`, but it takes its own connection from the pool on each call, so on its own it does not pin across calls (`Run.hs`).
- **What `withResource` does at the edges** (`src/Data/Pool.hs`, 0.5.1.0): `r <- unmask (act res) \`onException\` destroyResource pool localPool res; putResource localPool res`.
  - On an **exception** the connection is destroyed. persistent's close frees the libpq connection, the session ends, and the lock goes with it.
  - On a **normal return** the connection goes back to the pool **still holding any session lock not explicitly unlocked**. The next borrower of that connection would then own it.
  - The current Node code avoids that with `pg_advisory_unlock` in a `finally` on the same client before `release()` (`tick.ts`). The same pattern applies here: unlock inside the `withResource` callback, in `finally`/`bracket`.
- **SQL from persistent:** `rawSql "SELECT pg_try_advisory_lock(?)" [PersistInt64 k]` returning `[Single Bool]`, and `rawExecute` or `rawSql` for `pg_advisory_unlock` (persistent `Database.Persist.Sql`). Esqueleto has no advisory-lock wrapper (see the version baseline).
- **The transaction-level alternative:** `pg_try_advisory_xact_lock` inside one `runSqlPool` frees itself at commit or rollback, with no pinning or unlock needed. But then the whole pass is one transaction. The current service already uses `pg_advisory_xact_lock` elsewhere (`reviewer-accounts.ts`, `requests.service.ts`), and uses the session form for the tick.
- **Idle reaping does not touch a held connection.** resource-pool's collector threads destroy idle resources in the stash ("ordered by `lastUsed`", `Data/Pool/Internal.hs`). A resource checked out by `withResource` is not in the stash. (Read from the source structure; the docs do not say it in these words.)

## Summary table

"stored" means the retry time is written to the database or server. "per process" means the limit counts only that process's threads.

| option | store / driver | latest (date) | shares persistent's pool/txn | retry/backoff | delayed | concurrency | stuck-job handling | UI | cron |
|---|---|---|---|---|---|---|---|---|---|
| odd-jobs | PG table / postgresql-simple | 0.2.2 (2020-07-07); 0.2.3 unreleased | txn yes via `getSimpleConn`; own `Pool Connection` for the runner | stored, `2^attempts` s, max 10 | `scheduleJob` (±5 s) | `MaxConcurrentJobs`, resources, dynamic (per process) | lock timeout 600 s, no heartbeat | Servant+Lucid admin UI, auth not stated | none |
| hasql-queue | PG / hasql (unbounded) | 1.2.0.2 (2020-07-06) | no | retry count, no backoff stated | not stated | not stated | not stated | none | none |
| pgmq-hasql (+ shibuya adapter / keiro-pgmq) | PGMQ schema / hasql ^>=1.10 | 0.7.0.0 (2026-10-26) | no (separate hasql pool) | visibility timeout; adapter retry + DLQ, backoff not stated | `delay` | shibuya supervision (knobs not read) | visibility timeout, `set_vt` | `pgmq-inspect` read-only, auth not stated | none |
| pgqueuer-hs | pgqueuer schema / postgresql-simple | 0.0.1.0 (2026-07-12) | not stated | retry or hold, detail not stated | `execute_after` | per-entrypoint + global | not stated | none | not stated |
| hand-rolled SKIP LOCKED | existing tables / persistent | — | yes | build it (`run_at`) | build it | build it | build it (stall guard) | build it | the tick |
| BullMQ layout | Redis / hedis + BullMQ Lua | no Haskell client exists | — | would be BullMQ's | would be BullMQ's | would be BullMQ's | would be BullMQ's | none (bull-board dropped) | — |
| hworker | Redis (own layout) / hedis | 0.1.0.1 (2017-01-01) | — | `Retry` status, no backoff stated | not stated | not stated | monitor re-queues after timeout | none | none |
| faktory | Faktory server (Go, AGPL) | 1.2.0.0 (2026-07-23) | — | server: 25 tries, `15+count^4+jitter` | `at` / `in` | worker settings (not read) | reservation timeout (`reserveFor`) | Faktory Web UI | Enterprise-only `cron.toml` |
| loop + `threadDelay` | none | base | — | — | — | — | — | — | the tick |
| `cron` | memory | 0.7.2 (2024-11-09) | — | — | — | — | — | — | five-field `execSchedule` |

## Open points this note did not settle

- Whether odd-jobs 0.2.2 from Hackage (or master) builds against the GHC, Servant and aeson versions the Port will pin. Nothing published says so, and nothing was built here.
- Whether odd-jobs' poller query uses `SKIP LOCKED`. Its doc comment says only `UPDATE` plus `SELECT ... FOR UPDATE`.
- shibuya-pgmq-adapter's retry backoff, delayed-retry and concurrency configuration; the README names them, but the Haddocks were not read.
- How the existing BullMQ ports (Rust, Python, Elixir, PHP) package and version BullMQ's Lua scripts, which a Haskell client would have to track.
- Whether any of the hasql-based packages builds against hasql 2.0; all found pin `^>=1.10` or leave it unbounded.
