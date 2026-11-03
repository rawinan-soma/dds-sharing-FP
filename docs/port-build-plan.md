# Port build plan

The order the Port is built in: slices, each one PR that leaves `main` coherent,
and the blocking edges between them. Input for `to-tickets`; each slice becomes
one ticket.

Decided on "The Port's build order" (#33), map #1. What each slice builds is
decided on the map's other tickets; this file fixes only the order and where
each carried constraint lands.

## Shape

- **Node moves aside, the Port grows in place.** Slice 0 moves the Node code to
  `legacy/`; Rust and Elm are written at their final paths from their first
  commit. Cutover deletes `legacy/`.
- **Two tracks, joined at cutover.** The `api` and `web` tracks block only on
  their own slices and on the shared foundation. The contract
  (`docs/http-api.openapi.yaml`) is the boundary: a Rust slice is proved by its
  integration tests (and runs behind Angular), a web slice by `elm-test` /
  `elm-program-test` (and runs against `legacy/api`). A closing chain joins
  them.
- **Progress is a shrinking `unported` list.** A checked-in list of the
  operations not yet ported, keyed by OpenAPI tag (`ops`, `requester`,
  `delivery`, `reviewer-auth`, `reviewer`); the `reviewer` tag is split by
  operationId. It starts full. Each `api` slice removes its own entries in the
  same PR. The coverage test skips what is listed; the ledger check treats every
  area off the list as ported. The `web` slices fill their ledger rows the same
  way. Cutover deletes the empty list.

`←` names the slices a slice is blocked by.

## Shared

### 0. Move Node aside

`git mv apps/api legacy/api` and `apps/web legacy/web`; fix the paths in
`pnpm-workspace.yaml`, the root `package.json`, CI and the `Dockerfile`. No
other change: the Node suites stay green where they are.

### F. Copy and theme foundation (← 0)

Files only, no Elm and no Rust.

- Rename the catalogue to `messages/messages.{th,en}.json`, without `$schema`;
  check the inlang plugin (still serving `legacy/web`) accepts a file without
  it; update the paths in `docs/spec.md` and ADR 0010.
- Start `theme/theme.css` from the `dds-a` variant on the
  `prototype/daisyui-theme` branch, at the top level beside `messages/`, so
  `apps/web` and `apps/api`'s `build.rs` both read it.

## `api` track

### 1. Cargo workspace and schema (← 0)

- `rust-toolchain.toml`, the virtual Cargo workspace (`apps/api`,
  `apps/api/migration`, `tools/fake-upstream`, `xtask`), pinned Cargo CLIs.
- SeaORM DSL migrations, one per `.sql` file.
- The 12 Drizzle `.sql` files frozen into
  `apps/api/migration/tests/drizzle-sql/`, and the schema-equivalence test.
- Generated entities with their diff check; the province seed generated from
  the CSV; `dds migrate`.
- The CI `api` job, cargo-deny, Dependabot for Cargo.

### 2. Server skeleton (← 1)

- `dds serve` and `dds check`: config, logging, `axum-client-ip`, the error
  shape, `serve` refusing a schema that isn't current.
- `GET /api/health` (`ops`, less `/api/health/scheduler`).
- The OpenAPI response validator, the coverage test and the `unported` list.
- `docs/port-test-ledger.md`, generated from the `legacy/` suites, with its CI
  check.

### 3. Requester submission (← 2)

`requester`: `/api/reference`, `POST /api/requests` — duplicate suppression,
the reference number, audit events. No Probe yet (6a).

### 4. Reviewer sign-in (← 2)

`reviewer-auth` and `dds reviewer …`: sessions over `reviewer_session`, CSRF,
`argon2`, `totp-rs`, the keyed sign-in lock.

### 5. Jobs, tick and mail (← 2, F)

- Redis with `bullmq-official`; the tick holding the advisory lock on a pinned
  connection; `/api/health/scheduler`.
- `lettre` with Mailpit (a service of the `api` CI job from here); the askama
  emails; `build.rs` generating the copy functions from
  `messages/messages.th.json` and the colour consts from `theme/theme.css`; the
  send-retry job.
- The §17.1 catalogue parity and Thai-script gate as a Rust test.

### 6a. Probe (← 3, 5, 7)

Submission enqueues the Probe; `probe_performed` / `probe_failed`.

### 6. Queue and Decision (← 4, 6a)

Queue list and view, approve (enqueues the extraction), reject (sends its
email), amend-note.

### 7. Fake upstream in Rust (← 1)

`tools/fake-upstream/` with the per-token control endpoint, replacing
`legacy/api/test/fake-upstream/`.

### 8. Extract writer (← 1)

The golden fixture generated from `legacy/api`'s writer (before `legacy/` is
deleted), then the `sha2` writer byte-identical under it, the deflate-only
`zip`, the fingerprint, the typed `UpstreamRow`, `dds verify-extract` (file and
stdin). Pure code, no I/O.

### 9. Extraction pipeline (← 5, 7, 8)

- Fetch, filter, project, completeness, retry, resume and stall.
- Garage through `aws-sdk-s3`: Garage and the one-shot `garage-init` replace
  MinIO in the compose files, `MINIO_*` becomes `S3_*` there and in
  `.env.example`, and Garage joins the `api` CI job.
- The delivery and failure emails.

### 10. Delivery page (← 9, F)

`delivery`: `/d/{token}`, the archive, `/link-expired`; the second Tailwind +
daisyUI build over the templates, committed and diff-checked; deletion,
collection lapse, `expired_uncollected`.

### 11. Reviewer follow-ups (← 6, 9)

The rest of `reviewer`: alerts, in-flight, re-run, resend, lookup.

### 12. Remaining host commands (← 9)

`dds traffic-report` and any `dds` subcommand not landed by an earlier slice.

## `web` track

Each slice fills the ledger rows of the Angular `it`s it replaces.

### w1. Elm shell (← F)

- elm-land in `apps/web` as a pnpm island (`.node-version`, `packageManager`);
  Tailwind v4 + daisyUI via `.postcssrc.json`, daisyUI pinned exactly,
  importing `theme/theme.css`.
- travelm-agency `--inline`, compiling only `messages.th.json`.
- `elm-test` + `elm-program-test`: check the generated `.elm-land/src/Main.elm`
  is reachable from the tests' `source-directories`; the
  `Effect` → `SimulatedEffect` mapping; contract-validated JSON fixtures and
  `Fixtures.elm`; elm-review.
- The dev server proxies `/api` and `/d` to `legacy/api`.
- The CI `web` job.

### w2. Requester date and span logic (← w1)

`be-date`, `calendar`, `span-cap`, `format-day` under `elm-test`; the date
field.

### w3. Requester page (← w2)

The form, segmented control, region map, de-identification block, phone text,
submission state, and the submitted page.

### w4. Reviewer shell and sign-in (← w1)

Session, expiry toast, password gate, sign-in, `return-to`.

### w5. Queue and review screen (← w4)

Queue page (`queue-format`), the dossier, the Decision dialog and amend-note.

### w6. Reviewer follow-ups (← w5)

Alerts, in-flight, record, lookup.

## Closing chain

### c1. Full stack (← 10, 11, 12, w3, w6)

- The `e2e/` pnpm island: Playwright with `@axe-core/playwright`, the three
  journeys on Rust + Elm; Garage, `garage-init` and Mailpit as CI services.
- The cargo-chef → `distroless/cc-debian12:nonroot` `Dockerfile`, serving the
  SPA from a directory.
- The CI `image` job, then `e2e` against the built image.

### c2. Cutover (← c1)

Delete `legacy/`, the root `package.json`, `pnpm-workspace.yaml` and
`pnpm-lock.yaml`, `project.inlang/`, the schema-equivalence test, the ledger and
its check, and the empty `unported` list; delete `deploy/pilot/` and
`compose.pilot.yml`. Rulings marked `after cutover` in
`docs/port-disagreements.md` are not part of this slice.

### c3. Deploy (← c2)

- `xtask deploy` with `env`, `check`, `down`, `retire`, keeping the Pilot
  Makefile's guardrails; `check` refuses a missing, `REQ` or `PLT`
  `REFERENCE_PREFIX`.
- The production compose file: the one-shot `migrate` service with the owner
  env file only it receives, `app` depending on its success;
  `REFERENCE_PREFIX=PRT`, `SMTP_FROM=noreply-dds-sharing-port@ddc.mail.go.th`.
- The runbook, with the outside-Gmail arrival test (and its fallback to the
  Pilot's From) before the first real Requester.
- The `port` branch.
