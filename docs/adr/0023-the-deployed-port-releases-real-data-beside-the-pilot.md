# 23. The deployed Port releases real data beside the Pilot

Date: 2026-10-28

## Status

Accepted. Decided by the repo owner while working the Port map's ticket *The
upstream the Port calls* (#7, map #1). Amends the **Port** entry in
`CONTEXT.md` and the map's Notes, which had the Port serve no real Requesters
and release no real DDS surveillance data.

## Context

The Port re-implements this service on elm-land + daisyUI and axum + SeaORM, in
place in `apps/`, to evaluate that stack and to learn from rebuilding it. It was
charted as data-free: no upstream token, no real Requesters, no releases.

The question was what the Port calls in place of the upstream API. The
offered answer was the fake upstream only, ever. The repo owner wants more: the
Port must be exercised for real once deployed, not only in development.

## Decision

**In development and CI the Port calls only the fake upstream. Once deployed, it
calls the real upstream with the department's token and serves real
Requesters**: its Decisions are real releases and its Extracts real case-level
DDS surveillance data.

- **It does not replace the Node Pilot.** The deployed Port is a second release
  service beside the Pilot (and beside Production, when that exists), never
  merged into either. The Node Pilot stays untouchable.
- **One client, configured, never switched.** The Rust upstream client reads
  `UPSTREAM_BASE_URL` and `UPSTREAM_TOKEN` as the Node one does; the fake stands
  behind that seam, and there is no fake mode in the code.

## Considered options

- **Fake upstream only, ever** — declined: the deployed Port must call the real
  upstream.
- **Real upstream, with the repo owner as the only Requester** — declined in
  favour of real Requesters.
- **The Port replaces the Node code inside the Pilot** — declined: the Port is
  not for replacing the Node Pilot.

## Consequences

**ADR 0022's reasons return for the deployed Port.** Once its Download tokens
open real data, a mail catcher is a second screen showing them (§10.8), so the
deployed Port's mail goes through a real relay; development may still use a
catcher. The upstream token and real Extracts sit wherever the Port is deployed.

**The deployed Port needs its own answers to what the Pilot settled for
itself**: a reference prefix that says which service to look in, the PDPO
notice, whether its record is disposable, and how a Requester knows which of two
services they are using. These are open on the Port map.

**Its upstream traffic is real traffic** and counts in §13.6's report the same
way the Pilot's does.
