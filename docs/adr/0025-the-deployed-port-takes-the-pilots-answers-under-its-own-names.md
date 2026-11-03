# 25. The deployed Port takes the Pilot's answers under its own names

Date: 2026-11-03

## Status

Accepted. Decided by the repo owner while working the Port map's ticket *The
deployed Port as a second release service* (#27, map #1). Answers what
[ADR 0023](0023-the-deployed-port-releases-real-data-beside-the-pilot.md) left
open; adds `docs/spec.md` §18.16.

## Context

The deployed Port is a third release service for real Requesters, beside the
Pilot and (when it exists) Production. ADR 0022 settled the Pilot's own answers:
a reference prefix, the PDPO notice, a disposable record, one Reviewer at
launch. The Pilot shows a Requester nothing that names it; only its host name
and the `PLT-` on a reference number tell it from another service.

## Decision

**The deployed Port takes the Pilot's answers, under names of its own.**

- **Reference prefix `PRT`.** `REFERENCE_PREFIX` keeps its `^[A-Z]{2,8}$` rule
  and its `REQ` default; the deploy's checks refuse a missing value, `REQ` or
  `PLT`, so a forgotten setting cannot reissue another service's numbers.
- **Telling the services apart is the host name and the prefix only.** No
  banner, no configurable service name, no new copy: the Port's screens and
  emails are the Pilot's word for word.
- **From address `noreply-dds-sharing-port@ddc.mail.go.th`**, through the
  Pilot's relay and login (§11.2 needs only the `ddc.mail.go.th` domain). Before
  the first real Requester the deploy sends one message to an outside Gmail
  inbox and confirms it *arrives*; if the relay drops it, the Port falls back to
  the Pilot's `noreply-dds-sharing@…` rather than block the launch.
- **The PDPO is told** of the deployed Port the same way as of the Pilot, with
  no checked-in artefact (§18.1).
- **Its record is disposable.** No backup, no export.
- **One Reviewer at launch**, the repo owner, in the Port's own `reviewer` table.
  The two-Reviewer minimum is knowingly unmet; recovery is `dds reviewer …` over
  SSH on the Port's host.

## Considered options

- **Reusing `PLT`** — declined: two counters stamping the same numbers defeats
  §12.5. Reusing `REQ` would take Production's numbers before it exists.
- **A line on every page naming the service, or a service name from
  configuration** — declined: new copy in both catalogues, a disagreement-log
  entry, and (for configuration) a runtime string in a catalogue compiled to
  constants. A gap both services share is closed for both or for neither.
- **The Pilot's From address** — kept only as the fallback; the inbox should say
  which service wrote.
- **A nightly `pg_dump` off the host, or a Decisions-only export** — declined: a
  second copy of contact details that ADR 0019 keeps forever, against §18.11's
  one disk; backup is Production's to design first.
- **A deploy gate requiring a second Reviewer** — declined: stricter than the
  service the Port translates, for a rule the Pilot breaks knowingly.

## Consequences

**A Requester filling in the form cannot tell the Pilot from the Port except by
the URL.** Once they have a number, `PLT-` or `PRT-` says which service holds
it; the From address says it in the inbox.

**ADR 0022's costs hold again, on a second host**: case data on personally owned
hardware for up to 72 hours, Cloudflare terminating TLS, the upstream token and
relay credentials on the host, and releases that may leave no record anywhere.

**The From address is unproven** until the arrival check passes: the relay is
known to accept a From that is not the login's mailbox, not yet any local part.
