# Issue tracker: GitHub

Issues and PRs for this repo live on GitHub, `rawinan-soma/dds-sharing-FP`. Use the `gh` CLI for all operations. The build's spec is `docs/spec.md`.

## History

The tracker started empty. A `#NN` in commits, ADRs, `docs/` or code comments that predates this tracker's first issue is a historical label, not a link to an issue here; read the surrounding text for its meaning.

## Conventions

- Branches are named `ticket-<NN>`, after the GitHub issue number.
- The first line of an issue body is `Blocked by: #NN, #NN` when it has open blockers. Omit it when none.
- The body is `## What to build` prose followed by an `## Acceptance criteria` checklist (`- [ ]` lines). `pr-clearance` grades against the checklist.
- The triage role is a label (see `triage-labels.md`); the category is `enhancement` or `bug`.
- A PR closes its issue with `Closes #NN`.

## When a skill says "publish to the issue tracker"

Create a GitHub issue: `gh issue create --title "..." --body-file <file> --label <role>`.

## When a skill says "fetch the relevant ticket"

`gh issue view <NN> --comments`.

## When a skill says "apply a label"

`gh issue edit <NN> --add-label <role>`, removing the previous state role with `--remove-label`.

## Wayfinding operations

Used by `/wayfinder`. The **map** is a GitHub issue labelled `wayfinder:map`; each **child** is an issue labelled `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`) whose body starts `Map: #NN`.

- **Blocking**: a `Blocked by: #NN, #NN` first line. A child is unblocked when every issue it lists is closed.
- **Frontier**: open, unblocked, unassigned children of the map; lowest number wins.
- **Claim**: assign yourself (`gh issue edit <NN> --add-assignee @me`) before any work.
- **Resolve**: comment the answer under an `## Answer` heading, close the issue, then add a gist and link to the map issue's Decisions-so-far.
