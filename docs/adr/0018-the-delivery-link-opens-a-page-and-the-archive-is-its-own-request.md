# 18. The Delivery link opens a page, and the archive is its own request

Date: 2026-09-18

## Status

Accepted. **Amends** [ADR 0003](0003-plain-spa-and-a-collection-path-that-bypasses-it.md)'s
collection path, `docs/spec.md` §9.1, §9.2 and §16.2, and SRS FR-19. Leaves ADR
0003's guarantee intact: collection never depends on the Angular bundle. Decided
with the repo owner on 2026-09-18, after the design drew a download page that the
spec did not have.

## Context

§9.1 had `GET /d/<token>` count the Attempt and **stream the archive in the same
response**. The link in the Delivery email *was* the download.

That makes every GET of the emailed URL a presentation of the token, and every
presentation both an Attempt against the cap of 10 and a transfer of the Extract.
Mail-security scanners, link-preview services and some antivirus products open
links in incoming mail automatically, before a person does. Under §9.1 each such
visit **used up an Attempt and received the case-level archive** — into a scanning
service's infrastructure, outside anything this service can audit beyond an IP and
a user agent.

§11.1 records that every recipient is on a public mail provider. What those
providers and their customers' security tools do with links is not something this
service controls or can observe.

## Decision

**`GET /d/<token>` renders a small page. The archive is a second request,
`GET /d/<token>/archive`, made when a person presses the button. Only the second
request is an Attempt.**

- **The page** is rendered by NestJS from a server template, with its strings
  from the same catalogue as the four emails (§16.3). No Angular bundle, no
  client script is required to read it or press its button, so ADR 0003's
  guarantee holds: an Extract stays collectable if a front-end asset fails.
- **A page view is a lookup, not an Attempt.** It writes a `token_lookup` row, so
  the audit still shows who opened the link and when, and failed lookups still
  count towards the per-IP throttle (§9.2). It is **not** mirrored as
  `download_attempted` and does not touch the cap. The Request event catalogue is
  unchanged.
- **`GET /d/<token>/archive`** is where §9.2 now applies: it counts the Attempt at
  presentation, writes both audit rows, and streams with `Accept-Ranges: bytes`.
  A dropped transfer resumes against this URL, and each resume is a presentation,
  exactly as before. That is still why the cap is 10.
- **A dead token redirects to `/link-expired` from either route**, with the same
  one sentence for all four causes (§9.4).
- **`/d/<token>` stays fixed**, because it travels in email. `/d/<token>/archive`
  is reached only from the page, so it could move; keep it stable anyway.

## Consequences

**A scanner that opens the emailed link sees HTML and moves no data.** The Extract
leaves the service only when something requests the archive, and in the normal
case that is a person pressing a button.

**This raises the bar; it does not remove it.** A scanner that renders the page
and follows its links will still reach the archive and still count an Attempt.
Most fetch only the URL they were given; some do more. The audit trail, not the
cap, remains the control (§9.2).

**The counter means downloads now, and the wording says so.** The page, the
Reviewer's file panel and the lookup all say *ดาวน์โหลดแล้ว* rather than *เปิดลิงก์แล้ว*,
and the page tells the Requester that opening it does not count.

**The page shows the reference number, file name, size, time left and attempts
used.** Only to a holder of a live token, who already has all of it in the
Delivery email. A dead or unknown token still reaches the expiry page, which shows
none of it (§9.4).

**Only bytes make an Attempt** (amended on #19, `docs/port-disagreements.md` D3,
D4). A `HEAD` on the archive, or a `416` for a range past the end of the file,
sends none of the Extract, so it is audited as a lookup: it spends no Attempt,
moves no Request to `collected` and clears no lapse alert. Without this, a
link-checker that HEADs the archive link would mark an Extract collected that
nobody received.

**One extra round trip for the Requester.** Accepted: it is one click on a page
that tells them what they are about to download and how long they have.

## Alternatives rejected

**Keep streaming on the emailed URL** (§9.1 as it was). Simplest, and it lets any
automated visitor spend the Requester's Attempts and receive the data.

**Make the archive request a POST.** Scanners rarely POST, so this defends more
strongly. Rejected because browsers do not reliably resume an interrupted download
that began with a POST, and §16.2 relies on range-request resume for สคร.
connections that drop mid-transfer.

**Count the Attempt on the page view as well as the download.** Keeps "every
presentation is an Attempt" literally true, and brings back exactly the problem
this ADR solves.
