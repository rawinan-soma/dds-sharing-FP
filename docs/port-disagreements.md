# Port disagreements

Where the Node Pilot and the Port's authority (`docs/spec.md`, `CONTEXT.md`,
`docs/adr/`) part ways, and what the repo owner ruled. The Port builds to the
ruling; [`http-api.openapi.yaml`](http-api.openapi.yaml) carries each ruling that
changes the wire as an `x-disagreement` marker on the operation.

Decided on "The spec-vs-code disagreement log" (#19), map #1.

## What goes in

Two kinds of entry, tagged:

- **`contradicts`**: the authority says one thing and the Pilot does another.
- **`unspecified`**: the authority is silent and the Pilot made a choice that a
  Requester, a Reviewer, upstream, an operator or a rule can observe.

Internal choices nobody can observe (libraries, code shape) stay out. So do
code-quality notes: this is not a backlog.

## When it is ruled

- The porter logs an entry when they find it and keeps building **to the
  Pilot's behaviour**, which is always wire-safe.
- The owner rules on every entry a ticket raised **before that ticket's PR
  merges**. A ruling that changes behaviour is a commit on the same PR:
  contract, test and code together. Nothing merges with an open entry it
  raised.
- Each ruling says where it lands:
  - **`with the Port`**: the fix ships in the Port's first build.
  - **`after cutover`**: the fix would break the Angular app against the Rust
    server, or the Elm app against the Node API. Until both halves are ported
    the Port reproduces the Pilot, and the contract marks the entry pending.
    These entries open the post-cutover API redesign.

## Entry shape

Every code reference is `path:line` **at commit `a788f33`**, the Pilot as the
contract describes it. The Port replaces `apps/` in place, so an unpinned path
stops pointing at the Pilot.

```markdown
### D<n>: <title>

- **Tag:** contradicts | unspecified
- **Found by:** #<ticket>
- **Authority says:** <quote with §, term or ADR> | *silent*, and the nearest thing it does say
- **Pilot does:** <behaviour>, `<path>:<line>` @ a788f33
- **Observable by:** Requester | Reviewer | upstream | operator | <rule>
- **Options:** A. … B. … (a porter's preference goes here, marked as theirs)
- **Ruling:** <option>, <YYYY-MM-DD>: <one line of why>. **Lands:** with the Port | after cutover
- **Consequences:** <contract, spec, ADR, test changes>
```

An entry with an empty **Ruling** is open.

---

## Entries

### D1: Three error-body families

- **Tag:** unspecified
- **Found by:** #5
- **Authority says:** *silent* on error bodies for our own routes. Spec §5.5
  documents upstream's two shapes, not ours.
- **Pilot does:**
  1. `/api/requests` answers `{code, errors[]?, message?}`
     (`apps/api/src/requests/requests.controller.ts:33`).
  2. `/api/reviewer/**` answers `{error, …}`.
  3. Framework failures (malformed JSON, `ParseIntPipe` on `eventId`, an
     unknown `/api/*` route) answer Nest's default `{statusCode, message, error}`.

  The Angular app reads `errors[]` on submit
  (`apps/web/src/app/requester/requester-api.ts:45`) and `error` on reviewer
  calls (`apps/web/src/app/reviewer/queue-api.ts:388`, `:416`,
  `session-interceptor.ts:28`, `reviewer-api.ts:80`). Nothing reads family 3;
  it falls through to a generic `failed`.
- **Observable by:** Requester, Reviewer (through the screens' error handling).
- **Options:** A. Keep all three, the Port imitating Nest's default body.
  B. Family 3 answers in its route's own family, with the Pilot's statuses;
  families 1 and 2 are unified after cutover, shape chosen then. C. Copy all
  three until cutover.
- **Ruling:** B, 2026-10-26. Imitating a framework the Port drops, for a body
  nobody reads, is waste; choosing the unified shape now would be designing the
  post-cutover API one entry at a time.
  **Lands:** family 3 with the Port; unifying families 1 and 2 after cutover.
- **Consequences:** contract `info.description` "Errors". With the Port:
  malformed JSON on `/api/requests` → `400 {code: "invalid_request", errors:
  [{field: "body", code: "invalid", message}]}`; malformed JSON or a non-integer
  `eventId` under `/api/reviewer` → `400 {error: "bad_request"}`; an unknown
  `/api/*` route → `404 {error: "not_found"}`. The unified shape is the first
  item of the map's post-cutover API redesign.

### D2: `invalid_note` is 400 on reject, 422 on amend-note

- **Tag:** unspecified
- **Found by:** #5
- **Authority says:** *silent* on the status. Both routes run the same check.
- **Pilot does:** reject throws `400 {error: "invalid_note"}`
  (`apps/api/src/reviewer/review-queue.controller.ts:78`); amend-note throws
  `422` for the same `noteIsValid` failure
  (`apps/api/src/reviewer/decisions.controller.ts:49`). Elsewhere the Pilot uses
  400 for bad input and 422 only for `password_policy`, a well-formed value a
  rule refuses. The Angular app checks `400 && invalid_note` on reject
  (`apps/web/src/app/reviewer/queue-api.ts:420`); no screen calls amend-note.
- **Observable by:** a Reviewer's client.
- **Options:** A. Keep both. B. Amend-note answers 400. C. Both answer 422
  (breaks Angular's reject path).
- **Ruling:** B, 2026-10-26. One check, one status, matching the Pilot's own
  400/422 split; nothing depends on the outlier. **Lands:** with the Port.
- **Consequences:** contract amend-note: `invalid_note` moves from 422 to 400.

### D3: The archive never answers 416

- **Tag:** unspecified (first read as `contradicts`)
- **Found by:** #5
- **Authority says:** "The download endpoint supports range requests
  (`Accept-Ranges: bytes`, honouring `Range`) … a สคร. connection dropping at
  80% resumes rather than restarting" (spec §16.2). The Pilot meets that
  purpose, and RFC 9110 lets a server ignore `Range`; the spec is silent on a
  range that cannot be served.
- **Pilot does:** serves the first well-formed range with a 206. Anything else
  (malformed, a later range of a list, a start at or past the end) is ignored
  and gets the whole file with a 200, which counts as a successful Attempt
  (`apps/api/src/delivery/range.ts:14`, `:32`).
- **Observable by:** a Requester's download client; the Attempt count. A
  repeated `curl -C -` on a complete file sends `bytes=<size>-`, gets the whole
  file again and spends an Attempt.
- **Options:** A. Keep. B. A well-formed range starting at or past the end of
  the file answers `416` with `Content-Range: bytes */<size>`; malformed and
  multi-range headers keep today's 200. C. Full RFC treatment including
  `multipart/byteranges`.
- **Ruling:** B, 2026-10-26. It fixes the one real case and applies D4's rule:
  a presentation counts only when it offers bytes. A 416 is audited and is not an
  Attempt. **Lands:** with the Port (neither app sends `Range`).
- **Consequences:** contract archive route gains a 416; `CONTEXT.md` Attempt,
  spec §9.2 and ADR 0018 narrow the Attempt to presentations answered with
  bytes.

### D4: HEAD on the archive runs the Attempt path

- **Tag:** unspecified
- **Found by:** #5 (impact confirmed on #19)
- **Authority says:** *silent* on HEAD. An Attempt is "one presentation of a
  Download token" (`CONTEXT.md`), and ADR 0018 split the page from the archive
  so that "a scanner that opens the emailed link sees HTML and moves no data".
- **Pilot does:** Express routes HEAD to the GET handler
  (`apps/api/src/delivery/delivery.controller.ts:55`), so a HEAD on a live token
  counts a successful Attempt, moves the Request `approved → collected`, clears
  any collection-lapse alert and writes `download_attempted`
  (`apps/api/src/delivery/token-lookups.ts:50`), with no bytes sent.
- **Observable by:** Reviewer (a false `collected`, a lapse alert that never
  fires), Requester (a spent Attempt), the audit record.
- **Options:** A. Keep: a HEAD is a presentation. B. HEAD answers the same
  headers, and the same 302 or 429 for a dead or blocked token; it writes a
  `token_lookup` row but is not an Attempt, moves no state and writes no
  `download_attempted`. C. 405.
- **Ruling:** B, 2026-10-26. *Collected* must mean bytes were offered. B keeps
  the audit trail and is kind to download managers that HEAD before resuming.
  **Lands:** with the Port (neither app sends HEAD).
- **Consequences:** contract archive route gains a `head` operation; the
  Attempt definition narrows as in D3. **The running Pilot has the same hole**;
  it is outside map #1 and is filed as its own triage issue, #24.

### D5: Sign-out does not clear `reviewer_csrf`

- **Tag:** unspecified
- **Found by:** #5
- **Authority says:** "double-submit token on every state-changing `/reviewer`
  post" (spec §10.5); *silent* on the token's lifetime.
- **Pilot does:** sign-out clears only `reviewer_session`
  (`apps/api/src/reviewer/reviewer.controller.ts:138`). The CSRF cookie is
  reissued on every sign-in (`:120`) and issued by `GET /session` when absent
  (`:79`). Sign-in is itself CSRF-guarded (`:64`), and the Angular app keeps
  `ready` set after sign-out (`apps/web/src/app/reviewer/reviewer-session.ts:144`),
  so clearing the cookie would make its next sign-in fail `403 csrf` until a
  reload.
- **Observable by:** a Reviewer's browser.
- **Options:** A. Keep, and specify it. B. Clear it, after cutover, with the
  Elm app fetching `/session` before every sign-in. C. Leave it unspecified.
- **Ruling:** A, 2026-10-26. A CSRF token without a session authorizes nothing
  and is rotated at sign-in, so clearing it buys no security and costs a
  coupling. **Lands:** nothing changes.
- **Consequences:** spec §10.5 states the cookie's lifetime.

### D6: Reviewer routes keep the IPv4-mapped prefix

- **Tag:** unspecified
- **Found by:** #10
- **Authority says:** *silent* on the form an IP address is recorded in. The
  audit record keeps it (§12.3) as `inet`, where `::ffff:1.2.3.4` and `1.2.3.4`
  are different values.
- **Pilot does:** the Requester and delivery routes strip `::ffff:`
  (`apps/api/src/requests/requests.controller.ts:41`,
  `apps/api/src/delivery/delivery.controller.ts:9`, whose comment says every
  IP-keyed spot does). The Reviewer routes do not: `originOf` passes `req.ip`
  through (`apps/api/src/reviewer/reviewer.controller.ts:41`) into the per-IP
  throttle key and the `ip` of a `login_failed` event
  (`apps/api/src/reviewer/reviewer-auth.ts:144`). The prefix appears only when
  the socket peer is the client address (`TRUST_PROXY=false`, as in
  development); the deployed Pilot reads the proxy's `X-Forwarded-For`.
- **Observable by:** operator (a search of the record for one address misses
  the Reviewer events, or the Requester ones).
- **Options:** A. One rule: every route records the canonical form. B.
  Reproduce the Pilot.
- **Ruling:** A, 2026-10-28. The Pilot's own comment states the rule; the
  Reviewer controller is the exception it missed. **Lands:** with the Port.
- **Consequences:** the Port's client IP extractor returns the canonical
  address for every route; one test per route family feeds it a mapped
  address.
