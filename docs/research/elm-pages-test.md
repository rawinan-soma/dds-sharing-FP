# `elm-pages test` against the Port's web suites

- **Question:** Can `elm-pages test` (`Test.PagesProgram`, `Test.BackendTask`) replace `elm-test` + `elm-program-test` for what the Angular suites cover today? What can and can't be simulated, by behaviour; can responses come from the contract-validated JSON fixtures; can routes that call the browser's own HTTP be driven; what do `--coverage` and `/_tests` do; do `@axe-core/playwright` journeys behave differently on a hydrating page?
- **Issue:** #52, on map #50. The earlier decision is #15. Follows `docs/research/elm-pages-full-stack.md` §7 on branch `research/elm-pages-full-stack`.
- **Date:** 2026-11-05
- **Primary sources:** the elm-pages repo at commit `a1451f36e` ("Prepare 3.5.1 release"): `src/Test/PagesProgram.elm`, `src/Test/PagesProgram/{SimulatedEffect,SimulatedSub}.elm`, `src/Test/BackendTask.elm`, `src/Test/BackendTask/Internal.elm`, `generator/src/commands/{test,shared}.js`, `generator/src/{coverage,dev-server}.js`, `generator/template/app/Effect.elm`, its docs. `avh4/elm-program-test` 4.0.1 from the Elm package cache. In this repo: the 22 specs under `apps/web/src`, `docs/http-api.openapi.yaml`.
- **Run, not just read:** a throwaway app in a scratch directory outside this repo, on Node 22.23.3: elm-pages 3.5.1, Elm package 12.3.0, Lamdera 0.19.1-1.4.0, Vite 7.3.7, elm-test 0.19.1-revision16, elm-coverage 0.4.1, elm-program-test 4.0.1, Playwright 1.57.0, `@axe-core/playwright` 4.14.0, Chromium headless shell build 1200 (143). The browser runs used a production `elm-pages build`, our own small Node server calling `render()`, and a stub standing in for axum. Results are marked **(ran)**. **not stated** means no source says so. ⚠️ marks what was not verified. Source paths are relative to the elm-pages repo unless they start with `docs/` or `apps/`.

This note reports facts. It recommends nothing; the open points are at the end.

## Summary

- **Most behaviours can be driven on real routes.** Typing, blur, clicks, `pressKey`, `selectOption`, `check`, disabled buttons, attributes, form actions, redirects from `data` and `action`, and return-to all ran: 40 of 40 tests in about 0.5 s **(ran)**.
- **Four things can't be simulated on a route.** They are a non-2xx response to `BackendTask.Http`, the browser's own `elm/http` as a pending request, a route's `subscriptions` or passing time, and programmatic navigation from `update`. Moving focus can't be asserted at all **(ran)**.
- **The fixtures fit, with a status limit.** A `Fixtures.elm` string becomes the required `Encode.Value` with one line **(ran)**. `Test.PagesProgram` answers `BackendTask.Http` only with status 200 (`Internal.elm:2837-2846`), and the contract lists 62 4xx/5xx responses against 25 2xx. `Test.BackendTask.simulateHttp` takes any status, but runs on a `BackendTask` value and not on a route **(ran)**.
- **Browser HTTP can only be answered at once.** `SimulatedEffect` has no HTTP constructor. A per-test table through `PagesProgram.start` delivered 201, 409, 503 and unreachable **(ran)**, but the request isn't recorded, and one test can't see the in-flight state and then the reply.
- **`--coverage` works** and writes `coverage/lcov.info`, once `elm-instrument` from `elm-coverage` is on the `PATH` **(ran)**. **`/_tests` is a dev-server viewer** of route tests, not a CI runner **(ran)**.
- **Hydration replaces the DOM.** axe's results were the same before, during and after it **(ran)**. Before hydration, a click is lost and typed text is wiped; a native form POST still works **(ran)**.

## 1. By behaviour

The specs hold 267 `it` blocks. "Route" below means `TestApp.start` on a real route module. All rows are **(ran)** on the trial's routes, which are written to the same shape as the specs and are not ports of them.

| Behaviour (specs) | Hard case tried | Route test | What it could not do |
|---|---|---|---|
| Date field, span cap, calendar logic (`be-date`, `span-cap`, `calendar`, `request-form`: 40 `it`) | Leap-year 366 refused inline, 365 accepted; 29 Feb 2568 refused; blur rewrites the value at rest | Passes. The pure rules also ran as plain `Test` values | The calendar's popover was not built ⚠️. A disabled day is the disabled-button case below |
| Region map (`request-page`) | 13 `role=radio` cells, click, `aria-checked`, ArrowRight moves selection and tab stop; province by `selectOption` | Passes | Whether focus moved (§5). `clickButton` matches text content, and the source has no `aria-label` handling, so the trial used `clickButtonWith` |
| Submission states (`request-page`, `submitted-page`) | Sending state with edit disabled; 201, 409 with fields kept, 503, unreachable | Passes through the table in §3 | Request body and "posts nothing" can't be asserted on browser HTTP. In-flight and reply are separate tests |
| Session expiry (`reviewer-session`, `session-toast`, `session-interceptor`, `queue`) | Warning on arrival inside the last five minutes, dismiss; `data` 401 redirect, status in-band (§2) | Arrival cases pass with `withRequestTime` | The ceiling arriving while the page is open: a `Time.every` never fires (§4). `pushUrl` on a browser 401 doesn't move the URL |
| `return-to` (`return-to`, `sign-in`, `reviewer.routes`) | Six hostile values fall back; sign-in form posts, follows the redirect, lands on return-to | Passes, including `ensureBrowserUrl` | Nothing found for this group |
| Decision dialog (`dossier`) | Opens naming the requester, `aria-modal`; Escape closes; busy with cancel disabled; 200 and 409 | Passes | Focus on open and on return |
| Focus and keyboard (9 `activeElement` assertions in 5 specs) | `Event.focus` opens a tip, Escape closes it; tab arrows wrap | The handlers and resulting attributes pass | `document.activeElement` has no equivalent (§5) |

Also from the specs, not tried:

- `record.page.spec.ts` carries a record in `history.state`. `ensureBrowserHistory` lists URLs only (`PagesProgram.elm:3442`). ⚠️ Not tried.
- `http.expectNone(() => true)` appears 10 times, for example "never polls". For browser HTTP there is no list of calls to assert on.

Test mechanics **(ran)**:

- **Discovery matches the annotation by regex:** `PagesProgram.Test` or `Test.PagesProgram.Test` (`shared.js:1199`). With `import Test.PagesProgram as P` and `suite : P.Test`, the run reported `Found 0 ProgramTest values` and went on without that suite, with no warning.
- `elm-explorations/test` must be a direct dependency, or the run stops with a clear message.
- Exit code: 2 on failure, 0 on success.
- A pending `BackendTask` fails the next view assertion and names the URL. Clicking a disabled button fails. `pressKey` on an element with no handler passes silently, as documented, with no browser default action.
- Assert `tabindex` with `Attr.tabindex`. `attribute "tabindex" "0"` fails, because the view holds the property `tabIndex`.

## 2. `BackendTask.Http` and the contract's fixtures

`data` and `action` calling axum from the page server is the shape these tools cover best.

- **Fixtures.** #15's shape, one `String` per JSON file, works: `Decode.decodeString Decode.value raw` gives what `simulateHttpGet`, `simulateHttpPost` and `simulateHttpPostWith` take **(ran)**. The fixtures in the trial were hand-written in the contract's shape. ⚠️ No fixture was validated against `docs/http-api.openapi.yaml` here; #15's Rust check doesn't depend on the test runner.
- **Status.** `Test.PagesProgram` exposes `simulateHttpGet`, `simulateHttpPost`, `simulateHttpPostWith` and `simulateHttpError`. The first three always give 200 with no headers (`httpSuccessResponse`, `Internal.elm:2837`). `simulateHttpError` takes only `NetworkError` or `Timeout` (`BackendTask.elm:249`). `PagesProgram` has no `simulateHttp`.
- **A 200 with an error body doesn't stand in for a 401:** the `data` didn't redirect **(ran)**.
- **Methods.** The contract has 12 GET, 10 POST and 1 HEAD operation. The two success simulators cover GET and POST; `Test.BackendTask.simulateHttp` takes any method.
- **Headers.** The `ensureHttp*` steps check URL and POST body. No step asserts request headers, so forwarding the `Cookie` to axum can't be checked on a route. No response headers can be supplied, so neither can axum's `Set-Cookie`.

Two ways to test a non-2xx path, both **(ran)**:

- **Factor the call out and use `Test.BackendTask`:** `load cookie id |> BT.fromBackendTask |> BT.simulateHttp {…} { statusCode = 401, … } |> BT.expectSuccessWith`. 401 and 409 passed. This tests the function, not the route: what the view does next isn't reached.
- **Bring the status in-band through `BackendTask.Custom`:** own JS does the fetch and returns `{status, …}`. `simulateCustom "api"` with 401 then drove the route's redirect, and `ensureBrowserUrl` saw `/reviewer/sign-in?returnTo=%2Freviewer%2Fprobe`. This replaces `BackendTask.Http` in the production code path.

**The redirect's data resolves twice.** After a sign-in action redirects, the test had to answer the dossier's GET twice. The real app did the same: `POST /reviewer/sign-in/content.dat/`, then `GET /reviewer/dossier/r1/content.dat?returnTo=%2F…` and `GET /reviewer/dossier/r1/content.dat`, with two GETs reaching the stub **(ran)**. Whether that is intended: **not stated**.

## 3. Routes that call the browser's own HTTP

`Effect` values reach the test through `testPerform`, whose `SimulatedEffect` has only `None`, `Batch`, `DispatchMsg`, `SetField` and `SubmitFetcher` (`SimulatedEffect.elm:54-59`). elm-program-test's has HTTP, tasks and sleep.

- A browser call mapped to `none` is invisible: `ensureHttpPost` says "No requests are currently pending", and `simulateHttpPost` says "The app is not currently waiting for any HTTP responses" **(ran)**.
- **What worked: a table per test.** `Test.PagesProgram.start` is exposed and takes the mapper, so `PagesProgram.start (sim table) Main.config path setup` can answer `Effect.Api r` with `dispatchMsg (r.toMsg result)`. Any status and any string body, so fixtures fit **(ran)**.
  - The reply arrives in the same step as the click. An in-flight state is visible only in a test with no entry for that URL.
  - Nothing records the call, so URL, body and "no call" can't be asserted.
- **What didn't work: delivering a reply later.**
  - The docs name `simulateMsg` (`SimulatedEffect.elm:24,65`, `generator/template/app/Effect.elm`), but it isn't defined or exposed in `Test.PagesProgram` at this commit.
  - A test-only port through `withSimulatedSubscriptions` needs a `Main.Msg`, and the generated `Main` exposes `Msg` without constructors. It fails to compile: "does not expose a `MsgRequest` variant" **(ran)**.
- **`fromConfig`** takes a plain `{data, init, update, view}` whose effects are `BackendTask`s. ⚠️ Not run. It doesn't start a real route.

**`elm-program-test` still runs beside it.** A plain `Test` using avh4's `ProgramTest` on a plain TEA module ran in the same `elm-pages test`, under Lamdera: a 409 through `simulateHttpResponse`, and `advanceTime` **(ran)**. ⚠️ It wasn't tried against a real route's `update`, which takes an `App` value. How a test would build one: **not stated**.

## 4. Time and subscriptions

- **A route's `subscriptions` isn't used by the tests.** `Test.PagesProgram` evaluates only the function given to `withSimulatedSubscriptions`, and `SimulatedSub` has only `port_` (`SimulatedSub.elm:39-42`). The trial's `Time.every` branch shows 0 hits in the coverage report **(ran)**.
- **No step advances time.** `withRequestTime` sets the request's time, which `data` reads, so a case decided on arrival works **(ran)**.
- The specs that advance a clock are `reviewer-session` and `queue`: 14 lines use fake timers. They cover the warning, the ceiling, "does nothing on idle time", stale counts and "never polls".
- **elm-program-test's clock is narrower than its name suggests.** `advanceTime` fires `Process.sleep` only. `SimulatedEffect.Time` exposes only `now` (`Time.every` was a compile error **(ran)**), so #15's plan would also have needed a sleep-based tick in the simulated effects.
- **In a real browser it works:** warning shown, then at the ceiling the URL became `/reviewer/sign-in?returnTo=%2Freviewer%2Fqueue`, and signing in returned to `/reviewer/queue` **(ran)**.

## 5. Focus and navigation

- **Focus.** No test module reads the active element: the only `Browser.Dom` use under `src/Test` is the viewer's own (`Viewer.elm`). `Effect.Focus` has nothing to map to. Elm-program-test has no `Browser.Dom` simulation either, so this gap isn't new.
- **In a real browser** all the focus moves worked: to the heading on arrival, to confirm when the dialog opened, back to the opener on Escape, and to the heading after a client-side landing **(ran)**.
- **Navigation that works in a route test:** form action redirect, `data` redirect, `clickLink`, `navigateTo`, `ensureBrowserHistory` **(ran)**.
- **Navigation that doesn't:** `Browser.Navigation.pushUrl` from `update`. After a 401 on a browser call, `ensureBrowserUrl` still read the dossier **(ran)**. `SimulatedEffect` has no navigation, and the template's `testPerform` maps `FetchRouteData` and `Submit` to `none`.

## 6. `--coverage` and `/_tests`

**`elm-pages test --coverage` (ran).**

- Without `elm-instrument` it stops: "elm-instrument not found. Install elm-coverage". `elm-coverage` installed locally with its `.bin` on the `PATH` was enough; no global install.
- It prints a per-module table and writes `coverage/lcov.info`. 13 modules from `app/` and `src/` were listed, tests excluded. 62.5% at the time, 36 tests.
- Filters: `--coverage-include`, `--coverage-exclude`, `--coverage-include-module`, `--coverage-exclude-module`. No threshold option in the sources searched (`coverage.js`, `cli.js`): a pass exited 0.
- It measures Elm only, not `custom-backend-task` JS. `Effect` read 29.3%, because `perform` never runs under test; the same applies to anything the tests can't reach, such as a `Tick` branch.
- The CHANGELOG first lists `--coverage` for `elm-pages run` in 3.3.4; the `test` flag is in the command's help.

**`/_tests` (ran).**

- Served by `elm-pages dev` (`dev-server.js:304`). A production server has no such route: our server returned 404. It's not part of `elm-pages test`.
- First load 6.6 s. It listed "2 modules · 23 tests" at that point, each with its steps and status. Previews render in iframes at `/_tests-preview?vp=1280`, through the Vite config.
- It shows `Test.PagesProgram` values only: the plain `Test`s and the elm-program-test ones were absent. Tests using the custom `PagesProgram.start` table appeared and passed.
- ⚠️ Only the overview was loaded. The step timeline and network and model inspectors described at `PagesProgram.elm:105-115` were not driven.

## 7. Playwright and axe on a hydrating page

Hydrating means the Elm bundle was held back or slowed with `context.route`. The trial's only axe violations were its own nested `<main>`, which is why the same three rules appear throughout.

**axe (ran).**

| State | Result |
|---|---|
| Server HTML, bundle held, on 3 routes | `landmark-main-is-top-level`, `landmark-no-duplicate-main`, `landmark-unique` |
| Same pages hydrated | The same three |
| axe started as the bundle was released | The same three, no error, no page error |
| Dialog open | The same three |
| After a client-side landing | The same three |

- The server HTML already holds the page: 13 radios with JS disabled **(ran)**.
- `<html lang="en">` from the trial's defaults, on Thai text, was not flagged by axe **(ran)**.
- ⚠️ The trial has no real CSS, so colour contrast against the theme says nothing here.

**Hydration replaces the nodes (ran).** A JS property set on the button, `h1` and `main` was gone afterwards, and the old button had `isConnected === false`. `Platform.elm:1511` uses `Browser.application`. The docs say "a meaningful and interactive app before hydration" (`14-architecture.md:31`). For anything needing `update`, the trial didn't bear that out:

| Before hydration | Result |
|---|---|
| Click a button whose handler is Elm | Nothing, and not replayed afterwards |
| Type in a plain Elm input, focused | Value `""` and focus lost after hydration |
| Submit a `Pages.Form`, bundle held | Native POST: landed on return-to |
| Submit it with JS disabled | The same |

**Timing in a journey (ran).**

| Wait, then click | Bundle | Clicks lost |
|---|---|---|
| `goto` with `commit` | Not slowed, local | 0 of 10 |
| `goto` with `commit` | +500 ms | 5 of 5 |
| `goto` with `domcontentloaded` | +500 ms | 0 of 5 |
| `commit`, then wait for `init`'s focus to land | +500 ms | 0 of 5 |

- The page's `<script defer>` and `type="module"` hold back `domcontentloaded`: with the bundle held indefinitely, `goto` with that wait timed out at 30 s **(ran)**. Playwright's default wait is `load`.
- **No hydration marker found.** Body and `html` attributes and children matched before and after. The announcer text changed from `""` to `Navigated to <title>` **(ran)**; that it is meant as a signal is **not stated**.
- **The announcer was wrong after a link.** After clicking a link from the queue to the dossier, `#elm-pages-announcer` read `Navigated to Page not found`, while `document.title` and the `h1` were right and `content.dat` was 200. 2 of 2 runs. A form redirect gave the right title **(ran)**. ⚠️ Cause not isolated; nothing was filed.

## Open points

Facts missing or left unverified:

1. Whether `Test.PagesProgram` will gain a status-coded simulation, a time step, or `simulateMsg` (which its docs already name): **not stated**.
2. Building an `App` value to drive a real route's `update` from elm-program-test: **not stated**, not tried.
3. Not run: `fromConfig`, the `/_tests` step inspectors, fetchers and optimistic UI, `history.state`, the calendar popover, the password gate, real theme CSS with axe, real axum, the three #15 journeys with Mailpit and MinIO, browsers other than Chromium.
4. The double `content.dat` after a redirect, and the announcer text after a link: observed, causes not isolated.

Depending on who calls axum, for the ticket that settles it, with no pick here:

5. The trial tried both shapes. `BackendTask.Http` from the page server is visible to route tests but only as 200. Browser `elm/http` takes any status but only as an immediate reply with no record of the call.
6. What covers the gaps listed in §3, §4 and §5: `Test.BackendTask` on factored functions, in-band status, elm-program-test on plain modules, or Playwright.

## Sources

| Source | Used for |
|---|---|
| `github.com/dillonkearns/elm-pages` @ `a1451f36e`: `src/Test/PagesProgram.elm`, `src/Test/PagesProgram/{SimulatedEffect,SimulatedSub,CookieJar,Session,Viewer}.elm`, `src/Test/BackendTask.elm`, `src/Test/BackendTask/{Internal,Time}.elm`, `src/Pages/Internal/Platform.elm`, `generator/src/commands/{test,shared}.js`, `generator/src/{coverage,dev-server,generate-template-module-connector}.js`, `generator/template/app/Effect.elm`, `adapter/netlify.js`, `CHANGELOG-NPM.md`, `examples/docs/content/docs/14-architecture.md`, `examples/smoothies/` | API, behaviour |
| `avh4/elm-program-test` 4.0.1: `src/ProgramTest.elm`, `src/SimulatedEffect/{Time,Sub,Process}.elm`, `elm.json` | The baseline in #15 |
| This repo: `apps/web/src/app/**/*.spec.ts`, `apps/web/src/app/requester/{region-map.component,submission-state}.ts`, `docs/http-api.openapi.yaml` | Behaviours, response counts |
| #15, #50, #52; `docs/research/elm-pages-full-stack.md` on `research/elm-pages-full-stack` | Prior decision and facts |
| Throwaway trial: the versions above. 40 tests, 6 expected failures, a coverage run, `elm-pages dev`, and Playwright probes against a production build. Not committed | Everything marked **(ran)** |
