# elm-pages server-rendering in front of axum

- **Question:** Does a server-rendered elm-pages app work in front of an axum server, and where does it break? Covers `data` and `action` calling axum through `BackendTask.Http`; forwarding request cookies and passing two `Set-Cookie` headers back; the contract's double-submit CSRF across that hop; a route calling `/api` directly from the browser after hydration; dynamic routes such as `/reviewer/:id`; what the user sees when axum is down, when Node is down and when hydration fails; added latency; whether `.elm-pages/http-cache` and default fetch logging can capture response bodies on this path; and the production image and its dependencies on a read-only distroless Node.
- **Issue:** #51, on map #50.
- **Date:** 2026-11-05
- **Starts from:** `docs/research/elm-pages-full-stack.md` on branch `research/elm-pages-full-stack`, which tested elm-pages with no separate backend. Its facts about elm-pages are not repeated here.
- **Primary sources:** the elm-pages repo at commit `a1451f36e` (the npm 3.5.1 cut). In this repo: `docs/http-api.openapi.yaml`, `docs/spec.md` (§7.1, §10.5, §14.5, §16.1, §16.2), ADR 0003, `docs/port-disagreements.md` (D6), ticket #10.
- **Run, not just read:** a throwaway trial in a scratch directory outside this repo, on Node 22.23.3 and a 4-vCPU Linux host: elm-pages 3.5.1, Elm package 12.3.0, Lamdera 0.19.1-1.4.0, Vite 7.3.7, make-fetch-happen 15.0.6, with our own Node adapter. In front of it, a stub on axum 0.8.9 that sets `reviewer_session` and `reviewer_csrf` and enforces double-submit CSRF. Browser scenarios ran in Chromium through playwright-core 1.57.0. Results are marked **(ran)**. **not stated** means no source says so. ⚠️ marks what was not verified. Source paths are relative to the elm-pages repo unless they start with `docs/`.

This note reports facts. It recommends nothing; the open points are at the end.

## Summary

- **The pairing works end to end.** Sign-in, a queue page, `/reviewer/:id` and an approve action all ran through `BackendTask.Http` to axum, with and without JavaScript, and in a read-only distroless container **(ran)**.
- **Two `Set-Cookie` headers reach `BackendTask.Http` as one comma-joined string.** `Metadata.headers` is a `Dict`, built with `Object.fromEntries(response.headers.entries())` (`request-cache.js`). Passing them back needs a split that survives the comma in `Expires`. A regex split worked, and so did Node's own `fetch` through `BackendTask.Custom` with `getSetCookie()` **(ran)**.
- **Double-submit CSRF holds across the hop only if the token comes from the browser's request.** A hidden form field forwarded as `X-CSRF-Token` passed, and a wrong one got axum's 403. An `action` that copied the cookie into the header on the server approved a POST carrying no token at all **(ran)**.
- **A browser calling `/api` directly after hydration works.** It needs one origin for pages and `/api`, and without a port Elm reaches the token only through page data, so a re-issued token makes an open tab go stale: 403 **(ran)**.
- **Axum down: the outcome depends on the route's code.** A mapped error gave our own page in 4 ms, or after the configured timeout if axum hangs. With no `timeoutInMs`, a page waited until the client gave up. A hydrated `action` whose follow-up `data` fails replaced the page with the bare text `Model mismatch` **(ran)**.
- **Node down: `/d/<token>` and `/api` kept working, when something other than Node routes them.** An open tab's link click then changed the URL, kept the old content and showed no error **(ran)**.
- **Hydration failing is not one case.** With the script blocked or throwing, the server-rendered page stays and forms still post. With damaged inline page data, the page goes blank **(ran)**.
- **Latency:** a page with one axum call took 2.5 ms at p50 against 0.1 ms for axum alone, sequentially on loopback, with a stub that does no database work. At 10 concurrent clients that was 8.9 ms and about 1,100 req/s **(ran)**.
- **`.elm-pages/http-cache` can capture response bodies; default logging captured URLs only.** If that directory exists and is writable, axum's JSON bodies land on disk even with no `Cache-Control`. Cookies and tokens did not. With `ForceCache`, one Reviewer was served another's body. A fatal decode error put a whole response body into the error page, and into no log **(ran)**.
- **Production image:** 222 MB, 18 MB above the distroless base. It ran under `--read-only --cap-drop ALL`, wrote nothing to its filesystem and used about 50 MiB after 500 pages. `npm audit` on the runtime set reports 4 moderate advisories **(ran)**.
- **The bundle installs its own SIGTERM handler that calls `process.exit(143)`**, which cut an in-flight request in the container **(ran)**.

## 1. The trial's shape

```
browser ── edge or axum ─┬─ /api, /d, /health ──────────► axum (cookies, CSRF, sessions)
                         └─ everything else ─► Node ─────► axum, over BackendTask.Http
```

Map #50 gives `/d/<token>` and `/api` to axum, so something other than Node has to route them. Two front ends ran **(ran)**:

- **Axum as the front door:** the stub serves `/api` and `/d`, and proxies the rest to Node.
- **A separate edge:** a small `node:http` router on its own port, standing in for path rules at a tunnel. ⚠️ No real Cloudflare Tunnel or Compose was run.

The stub follows the contract's cookies: `reviewer_session` with `Expires`, `HttpOnly`, `SameSite=Lax`; `reviewer_csrf` without `HttpOnly`, issued by `GET /api/reviewer/session` when absent and re-issued on sign-in. `Secure` was off, because the trial had no TLS. CSRF runs before the session check. Its rows carried marker strings, standing in for Requester contact fields.

⚠️ Everything here is the stub's behaviour, not the Port's: no database, no real session sliding, no `axum-client-ip`.

## 2. `data` and `action` calling axum

**(ran)** Both work. Each route forwards `cookie` and `x-forwarded-for` by hand from `Server.Request`, and axum's status comes back through `BackendTask.toResult`:

| What | Result |
|---|---|
| `data` over `GET /api/reviewer/queue` | Page rendered with the Reviewer's name and Thai text |
| axum 401 in `data` | `Response.temporaryRedirect` to sign-in: 302 |
| axum 404 in `data` | `Response.errorPage ErrorPage.NotFound`: 404 |
| `action` over `POST /api/reviewer/sign-in` | Redirect carrying both cookies, with or without JavaScript |
| `action` over `POST …/approve` | Page re-rendered with `approved` |
| axum's 403 passed on by an `action` | `withStatusCode 403 |> Response.mapError never` gave 403 on a no-JS POST. A hydrated POST to `content.dat/` still returned 200, with the message in the page |

Behaviour a port would meet:

- **An `action` response runs `data` as well**, so a re-rendered approve made two axum calls, the POST and then the GET of the same Request **(ran)**. A redirect did not run the old route's `data`: the sign-in log shows only the POST **(ran)**.
- **A hydrated form posts to `<route>/content.dat/`**, with a trailing slash (`frozen-views-client.js:58`). The redirect travels inside a 200 response, and the client then fetched `/reviewer/content.dat` **(ran)**.
- **What axum receives** beyond what the route sets **(ran)**: `elm-pages-internal: ExpectJson`, `user-agent: request`, `accept: */*`, `accept-encoding: gzip,deflate`, `connection: keep-alive`. Nothing is forwarded unless the route says so.
- **Redirects from axum are followed silently.** A 303 from axum came back to Elm as the target's body **(ran)**. `BackendTask.Http` has no redirect option (no match in `src/BackendTask/Http.elm`). Whether a `Set-Cookie` on the 303 can be read is ⚠️ not tested. The contract's only 3xx responses are under `/d`, which Node does not call.
- **Timeouts arrive as `NetworkError`.** `timeoutInMs` fired on time, but Elm got `NetworkError`, not `Timeout`, because the error's code is `FETCH_ERROR` and `request-cache.js` only matches `ETIMEDOUT` and `ERR_SOCKET_TIMEOUT` **(ran)**.
- **Hanging is the default.** With no `timeoutInMs`, a request against a hung axum was still open when the client gave up at 12 s **(ran)**. ⚠️ How long Node would hold it was not measured.
- **`BackendTask.Http` cannot abort**, as the earlier note found. Node's `fetch` through `Custom` took `AbortSignal.timeout` **(ran for a normal call; the abort itself was not exercised)**.

## 3. Cookies across the hop

**Request cookies.** `Request.header "cookie"` gives the raw header, and axum received `reviewer_session=…; reviewer_csrf=…` unchanged **(ran)**.

**Response cookies.** `Metadata.headers` is `Dict String String`. Two `Set-Cookie` headers from axum arrived in Elm as this **(ran)**:

```
reviewer_session=abc; Path=/; Expires=Thu, 05 Nov 2026 23:00:00 GMT; HttpOnly; SameSite=Lax, reviewer_csrf=def; Path=/; SameSite=Lax
```

A split at any comma would break `Expires`. Two ways through both ran:

- **A regex in Elm:** split on `,\s*(?=[!#$%&'*+\-.^_`|~0-9A-Za-z]+=)`, a comma followed by a cookie name and `=`. It would also split inside a value containing such a sequence; the contract's token values are base64url, which has no comma.
- **`BackendTask.Custom` with Node's `fetch`:** `res.headers.getSetCookie()` returns an array. This also bypasses make-fetch-happen and its cache. The Elm side is then a `Decoder` checked at run time only, as the earlier note describes.

**Passing them on.** `Response.withHeader "Set-Cookie"` twice produced two separate headers, on an HTML render and a redirect. In order, attributes intact, on a no-JS POST **(ran)**. On a hydrated POST the browser ended up with `reviewer_session` as `HttpOnly`, `SameSite=Lax` with an expiry, and `reviewer_csrf` readable from `document.cookie` **(ran)**. The adapter has to write the array as separate headers; `res.setHeader` with an array did.

**Sessions.** ⚠️ Not tested, because the stub does not slide: the contract says any request passing the session check slides the idle window, and §10.5 says only user-initiated requests may. Every `data` call makes such a request. The client's prefetch applies only to links with `elm-pages:prefetch` (`elm-pages.js:109`), and a hover before a click made no extra `content.dat` request here **(ran)**.

## 4. Double-submit CSRF

The contract wants header `X-CSRF-Token` equal to the `reviewer_csrf` cookie. A plain form cannot set a header, so on this path Node turns a form field into the header.

| Request, all with valid cookies | Reached axum | Result (ran) |
|---|---|---|
| Hidden `csrf` field equal to the cookie | Yes, `csrfOk: true` | Approved, no-JS and hydrated |
| Wrong `csrf` field | Yes | axum 403 `{"error":"csrf"}`, shown in the page |
| No `csrf` field | No | The form's own validation stopped it |
| **`action` copying the cookie into the header on the server, POST with no token and `Origin: https://evil.example`** | Yes, `csrfOk: true` | **Approved** |
| Direct `POST /api/…/approve`, right header | Yes | 200 |
| Direct `POST /api/…/approve`, no header | Yes | 403 |

The fourth row is the failure. Node holds the cookie, so it can always satisfy axum's comparison, and the check then proves nothing about the browser. ⚠️ `curl` sent the cookie there; a browser's `SameSite=Lax` would withhold it on a cross-site POST, leaving that as the only layer. Spec §10.5 asks for both layers. elm-pages itself gives no check, as the earlier note found.

**Where the token sits.** `data` read it from `Request.cookie` or from the fresh `Set-Cookie` and put it in `Data`. It then appears in the HTML, as a hidden input, and in `content.dat` **(ran)**. The contract's cookie is not `HttpOnly`, so this exposes nothing new to the page's own scripts. Elm has no browser-side cookie access without a port.

**Re-issue.** The contract re-issues the token on every sign-in. After a second sign-in in another tab, an open tab's approve got 403 through its hidden field and through a direct `/api` call using its `Data` token. The tab showed the 403 message and stayed where it was **(ran)**. ⚠️ Reading `document.cookie` at click time through a port was not tried.

**A dead session** (signed out elsewhere): a hydrated link and a hydrated action both ended on `/reviewer/sign-in`, following our redirect **(ran)**. The stale `reviewer_csrf` survives sign-out, as D5 says **(ran)**.

## 5. The browser calling `/api` after hydration

**(ran)** A stateful route issued `Http.get "/api/reviewer/session"` from `init` and `Http.request` with `X-CSRF-Token` from a button, via `Effect.fromCmd`:

- Server HTML held the model's initial text, `server-rendered`, so `init`'s command did not run on the server. After hydration the paragraph held axum's JSON.
- Right token: `200 {"status":"approved",…}`. Wrong token: `403 {"error":"csrf"}`, readable through `expectStringResponse`.
- The requests went straight to `/api` with no `content.dat` and no Node hop, and carried the `HttpOnly` session cookie by themselves.
- With hydration dead, the paragraph stayed at `server-rendered` and the buttons did nothing. Anything built this way has no no-JS path.
- With Node down, this tab's call still got 200. With axum down, the tab got the edge's 502.

Same origin is what makes this work without CORS. The trial got it from the front end in §1; Node itself proxied nothing. ⚠️ Without a front end, the adapter would have to proxy `/api` itself, which puts Node on that path; this was not built.

## 6. Dynamic routes

Route files and what they became **(ran)**:

| File | Path |
|---|---|
| `Route/Reviewer.elm` | `/reviewer` |
| `Route/Reviewer/SignIn.elm` | `/reviewer/sign-in`, kebab-cased, and it beats `:id` |
| `Route/Reviewer/Id_.elm` | `/reviewer/:id` |
| `Route/Reviewer/Index.elm` | `/reviewer/index`, **not** `/reviewer`. Beside `Id_.elm` it fails the build with `REDUNDANT PATTERN` |

Odd values for `:id`, with a live session **(ran)**. The 404s from non-UUIDs are axum's, mapped by our `data`:

| Path | Result | What axum got |
|---|---|---|
| `/reviewer/{uuid}` and `/reviewer/{uuid}/` | 200 | `/api/reviewer/queue/{uuid}` |
| `/reviewer/{uuid}/content.dat` | 200, bytes | same |
| `/reviewer/not-a-uuid` | 404 | the same text |
| `/reviewer/with.dot` | 404, framework's own page | **nothing** (the dot rule from the earlier note) |
| `/reviewer/..%2F..%2Ftest%2Flast` | 404, framework's own page | **nothing** (has a dot) |
| `/reviewer/%2E%2E%2F%2E%2E%2Ftest%2Flast` | 404 | Still encoded, so it stayed one segment: `queue/%2E%2E%2F…`. axum's `{id}` decoded it to `../../test/last`, and its check refused it |
| `/reviewer/x%2Fy` | 404 | `queue/x%2Fy`, one segment |
| `/reviewer/ทดสอบ` | 404 | `queue/%e0%b8%97…`, encoded by the HTTP client |
| `/reviewer/a%3Fb%3D1` | 404 | `queue/a%3Fb%3D1`, no query string created |
| `/reviewer/SIGN-IN` | 404 | `queue/SIGN-IN`. Literal matching is case-sensitive |
| `/reviewer/content.dat` | 200, bytes | `/api/reviewer/queue`. That is `/reviewer`'s data, so `content.dat` cannot be an `:id` |
| `/reviewer/{uuid}/extra` | 404 | nothing |

So a route parameter arrives percent-encoded as the browser sent it, and interpolating it into a URL kept it to one path segment in every case tried. ⚠️ `Url.percentEncode` on top would double-encode; not run.

**What `Data` keeps.** The approve route decoded the Requester's email and an internal note into `Data`, and its view never read them. Neither marker appeared in the HTML or in `content.dat`; the name, which the view reads, did **(ran)**. This is the static analysis the earlier note describes, for which a security guarantee is **not stated**.

## 7. What the user sees

### Axum down, Node up

Measured against Node directly, and in a browser through the edge **(ran)**. "Process gone" refuses at once. "Hung" is `SIGSTOP`: connections are accepted and never answered.

| Situation | Status, time | What the user sees |
|---|---|---|
| `data` mapping the error to a message, process gone | 200, 4 ms | Our text: `ระบบยังไม่พร้อมใช้งาน: NetworkError` |
| Same, hung, `timeoutInMs` 5000 | 200, 5.0 s | Same text |
| `data` with no timeout, hung | none after 8 s, client gave up | A loading page |
| `Response.errorPage`, full load | 500, 5 ms | The `ErrorPage` view |
| Same, hydrated link click | `content.dat` 200 | The `ErrorPage` view, URL updated |
| `allowFatal`, full load | 500 | The `ErrorPage` view with the error's text (§9) |
| Sign-in page | 200 | The form, with an empty token. ⚠️ Submitting it was not run |
| No-JS approve POST | 500 | The `ErrorPage` view: the `action`'s message is lost when the follow-up `data` fails |
| **Hydrated approve POST** | `content.dat/` 200 | **Blank page with the text `Model mismatch`**, title the same (`codegen/GenerateMain.elm:478`). A reload gave the `ErrorPage` view |
| A page calling nothing | 200, 2 ms | Normal |
| `/d/<token>` via the edge | 502 | The edge's page. `/d` belongs to axum, so this is that path's own outage |

If axum is the front door, axum down means the whole site is down, since it owns the proxy. ⚠️ Inferred from the topology; only the Node-down side was exercised on that front end.

### Node down, axum up

**(ran)**

| Situation | Axum as front door | Separate edge |
|---|---|---|
| Fresh page | 502 and the stub's own Thai page, in 1 ms | 502 and the edge's page |
| `/d/<token>`, `/d/<token>/archive`, `/api/health` | 200 | `/d/<token>` 200 (others not run) |
| Open tab, link click | not run | **URL changes, old content stays, no error shown** |
| Open tab, hydrated form POST | not run | `POST` 502, then a full load, then the edge's page |
| Open tab, direct `/api` call | not run | 200 |

ADR 0003's guarantee holds in both, on the stub's `/d` routes: an Extract remains collectable with Node dead. That follows from Node not being on the path. ⚠️ A real Cloudflare Tunnel error page was not seen.

**The stuck link click.** `Platform.elm:641` says any `content.dat` error means a full page load. In the trial the tab kept the old content under the new URL for the 6 s observed. The cause seen in source: `frozen-views-client.js` returns `null` on a non-OK response, and `elm-pages.js:48-53` then sends nothing to Elm. With Node back, the next link click worked **(ran)**.

### Hydration fails

Through a browser, intercepting the script or the page **(ran)**:

| Failure | What the user sees | Forms | Direct `/api` |
|---|---|---|---|
| JavaScript off | Server page | Sign-in and approve complete by native POST | No |
| Script blocked | Server page | Same | No, buttons do nothing |
| Script throws at start (`Elm is not defined`) | Server page | Same | No |
| Inline page data replaced, not base64 or empty | **Blank page**, title `Page Data Error`, no form | None | None |

So the script failing leaves a working server-rendered page, and bad data removes it. The bad `__ELM_PAGES_BYTES_DATA__` was produced by tampering. ⚠️ Whether it can arise otherwise, such as a proxy or antivirus rewriting HTML, was not tried.

### A deploy with a tab open

The approve route's `Data` gained a field, Node was rebuilt and restarted, and the old script file was gone from `dist/` **(ran)**:

- **Link click:** `content.dat` 200 in the new format. URL changed, old content stayed, no error, no reload. The same stuck state as above, although here `response.ok` was true. ⚠️ Cause not isolated.
- **Form POST:** the action **ran on the server** (one `/approve` call in the log), then the tab did a full load onto the new version. The user sees the page with no `approved` message.
- **Direct `/api` call:** unaffected.

On the Port's approve route the form POST above would be a Decision that completed with no confirmation on screen (inference). ⚠️ No real audit trail was involved; the stub only counts the call.

### Screen readers

After each client-side navigation, including one following a redirect and after Back, the `aria-live` announcer read `Navigated to Page not found`, while `document.title` was right (`คิวคำขอ`, `คำขอ`) **(ran)**. A same-page action announced the right title **(ran)**. `Platform.elm` sets it from `mainView config updatedModel |> .title`, and `GenerateMain.elm:416` holds the `Page not found` title. ⚠️ Cause not isolated, and not checked against another app. Full page loads announced correctly.

## 8. Added latency

Single load-generating client on the same host, keep-alive, loopback, no TLS, 2,000 requests per cell after 300 to warm up, Node logging to a file, the stub doing no database work **(ran)**. Times in ms:

| Target | Seq p50 | Seq p95 | Seq p99 | 10-conc p50 | 10-conc p95 | 10-conc p99 | 10-conc req/s |
|---|---|---|---|---|---|---|---|
| axum direct, `GET /api/reviewer/queue` | 0.11 | 0.18 | 0.25 | 0.79 | 2.04 | 4.16 | 10,050 |
| axum direct, `GET /api/reviewer/queue/{id}` | 0.07 | 0.12 | 0.20 | 0.46 | 0.91 | 1.06 | 19,048 |
| Node, `/plain` HTML, no axum call | 1.61 | 1.85 | 2.77 | 2.80 | 5.39 | 8.53 | 3,215 |
| Node, `/reviewer` HTML, 1 axum call | 2.54 | 3.66 | 5.06 | 8.94 | 11.92 | 14.93 | 1,081 |
| Node, `/reviewer/{id}` HTML, 1 axum call | 2.51 | 3.61 | 3.74 | 8.95 | 10.50 | 11.93 | 1,103 |
| Node, `/reviewer/{id}/content.dat` | 2.48 | 3.60 | 3.67 | 8.39 | 9.98 | 10.98 | 1,174 |
| Node, `/viacustom` HTML, Node `fetch`, 1 axum call | 2.07 | 2.59 | 4.04 | 6.69 | 10.75 | 16.27 | 1,388 |
| axum front door to Node, `/reviewer` HTML | 2.65 | 3.87 | 4.03 | 8.42 | 9.97 | 10.87 | 1,176 |

- Boot to first answer: 271 ms. First real page 19.6 ms, second 6.0 ms.
- An `action` that re-renders adds a second axum call (§2).
- Node RSS 160 MB after about 26,000 requests, axum RSS 6 MB.
- ⚠️ Pages were small and the stub answers instantly, so these bound the hop's overhead and say nothing about real pages, a real database or a real network. Node ran as one process.

## 9. Where response bodies can land (§7.1, §14.5)

On this path the bodies are axum's JSON, the Reviewer screens' contact details and dossiers. Upstream rows do not pass through Node if the backend stays in Rust as map #50 says, so §14.5's ban on case data in logs is the rule most directly in reach (inference).

### `.elm-pages/http-cache`

`request-cache.js` uses `./.elm-pages/http-cache` relative to the working directory, if that is readable and writable, and otherwise forces `no-store` (`safeFetch`, `canAccess`). Nothing in `generator/src` creates it, and it did not appear after `elm-pages build` **(ran)**. ⚠️ `elm-pages dev` was not run.

| Directory, request | Written to disk (ran) |
|---|---|
| Absent, even with `max-age=600` from axum | Nothing; no directory created |
| Present and writable, `GET` with **no `Cache-Control`** | **The body**, under `content-v2/` |
| Present, `Cache-Control: no-store` from axum | Nothing |
| Present, `private, max-age=600` | The body |
| Present, `cacheStrategy = Just IgnoreCache` | Nothing |
| Present, `chmod 500` | Nothing |
| Present, sign-in `POST` | Body not found on disk |
| Container, writable bind mount | Body: 4 files, 2 with the marker |
| Container, `:ro` bind mount | Nothing; page served |
| Container, read-only root, no directory | Nothing; `docker diff` empty |

**Cookies did not land.** The index entry held the URL, `reqHeaders: {}` and response headers without `set-cookie`. Searches for the session token, the CSRF token and a test cookie found no file in any of these runs **(ran)**.

**Reading the cache back.** The cache key is the URL alone, as the index shows:

- Default strategy: every request went back to axum, even inside `max-age=600`, so the disk copy was never served **(ran)**. ⚠️ Why was not isolated.
- **`ForceCache`: the second Reviewer got the first Reviewer's body**, the same `serial` and `who` **(ran)**. Forwarded cookies are not part of the key.

### Default logging

Each call prints `fetch <url> <id>: <ms>` to stdout, including path segments and the query string, about 90 bytes a line **(ran)**:

```
fetch http://127.0.0.1:4000/api/reviewer/queue/11111111-2222-4333-8444-555555555555 mw3819ksu2ud1n: 1.377ms
```

- No bodies and no cookies appeared in any log line **(ran)**. So what can reach a log on this path is what sits in the URL.
- `BackendTask.quiet` silenced the line; so did a quiet `Custom` call **(ran)**.
- A timeout prints a `console.trace` with the URL and a stack, no body **(ran)**. It is unconditional in source (`request-cache.js`). ⚠️ Not run under `quiet`.
- ⚠️ A logging adapter was not built. The trial's prints whatever `render()` rejects with, and it never fired on these routes.

### Error pages

**A fatal error on a route module does not reject `render()`.** It returned a 500 through the app's `ErrorPage`, with nothing in Node's log **(ran)**. The earlier note saw `render()` reject on an `ApiRoute`; route modules differ.

- The scaffold's `app/ErrorPage.elm` prints the message, and that text goes to the browser:
  - A decoder missing a field: **the whole JSON body, in both the HTML and `content.dat`**. Marker in log: 0 **(ran)**.
  - A wrong type: the offending value only **(ran)**.
  - `BadStatus`: status alone, and the body is not included (`errorToString`) **(ran)**. Terminal colour codes are in the page text.
- So no operator sees these failures unless we log them, and a Reviewer can see the body **(ran)**. `app/ErrorPage.elm` is our file, and `toResult` before `allowFatal` keeps the text in our hands (the `/reviewer` route, **ran**). ⚠️ A silent `ErrorPage` was not built.

## 10. Production image

Build on `node:22-bookworm-slim` plus `ca-certificates`, then runtime on `gcr.io/distroless/nodejs22-debian12:nonroot` (Node 22.22.0) **(ran)**:

| | |
|---|---|
| Image | 222 MB. Base 204 MB, `node_modules` 14.7 MB, `dist` 254 KB, `server-out` 250 KB |
| Docker build | 86 s |
| Run flags | `--read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges --network host` |
| Worked | Sign-in with two cookies, queue, approve, `content.dat`, static script, `BackendTask.Http`, `Custom` |
| `docker diff` | Empty |
| Memory | 49.5 MiB after 500 pages |
| No compiler | Runtime holds no Lamdera or Elm |

**Runtime dependencies.** `elm-pages-cli.mjs` imports these beyond Node's built-ins, the same ten as the earlier note: `@sindresorhus/merge-streams` 4.0.0, `cli-cursor` 5.0.0, `cookie-signature` 1.2.2, `cross-spawn` 7.0.6, `globby` 16.2.4, `gray-matter` 4.0.3, `kleur` 4.1.5, `make-fetch-happen` 15.0.6, `micromatch` 4.0.8, `which` 6.0.1. Installed alone: about 90 packages, 14 MB.

**`npm audit --omit=dev` on that set:** 4 moderate, 0 high **(ran)**. `glob` 9.0.0–14.0.1 (GHSA-3g3m-2fjr-6pw4) and `minimatch` 7.0.0–11.0.0 (GHSA-g3hv-75wg-m249), through `cacache` and `make-fetch-happen`. npm's offered fix is `make-fetch-happen@12.0.0`, a breaking downgrade. This answers the earlier note's open item; a bare `elm-pages` install had 12. All ten are imported by the bundle whichever of them a route uses, so `BackendTask.Custom`-only routes still need them at boot (inference).

**Shutdown.** `render.js:2497` calls `process.on("SIGTERM", …process.exit(143))`, and importing the bundle installs it, for a Port server too (it also handles `SIGINT` and `exit`) **(ran; source)**:

| Server's handling | In-flight request | Exit |
|---|---|---|
| `server.close()` on SIGTERM, added after the import, container | Client got 502 at 0.32 s, `docker stop` took 52 ms | 143 |
| Same, on the host | Connection dropped at 0.30 s | 143 |
| `process.removeAllListeners("SIGTERM")` first, then `server.close()` | Finished, 1.02 s | 0 |

That the third is safe for the bundle's own cleanup, which kills child processes, is ⚠️ not checked; this trial spawns none.

## 11. D6: the client address across the hop

The hop changes what axum's peer address means **(ran)**:

- **`ConnectInfo`** on any Node-made call is Node's address (`127.0.0.1` here), whichever browser caused it. ⚠️ So `CLIENT_IP_SOURCE=ConnectInfo`, which #10 names for development and CI, would key every sign-in reaching axum through Node on one address (inference).
- **`RightmostXForwardedFor`:** Node forwards the header verbatim and appends nothing. A client's spoofed `6.6.6.6`, with the front appending its peer, reached axum as `6.6.6.6, 127.0.0.1`, and the rightmost value is what the front recorded. A route that does not forward it leaves nothing for axum to read (inference).
- **Canonical form:** axum bound to `[::]` saw `::ffff:127.0.0.1`, and `to_canonical()` gave `127.0.0.1`. A mapped value in the header stays a string, `::ffff:203.0.113.9`, until axum parses it. Both would go through D6's one rule. ⚠️ `axum-client-ip` itself was not used; the stub read the header by hand.

## Against the requirements

| Requirement | On this path |
|---|---|
| §7.1 raw upstream never lands | Not on this path if Rust owns upstream (inference). The `http-cache` capture of axum's bodies is §9 |
| §10.5 cookie attributes | Passed through unchanged **(ran)**, `Secure` not exercised |
| §10.5 CSRF, double submit | Holds when the token comes from the browser. A server-side copy defeats it **(ran)** |
| §10.5 only user-initiated requests extend | ⚠️ Not exercised; depends on `data` calling axum (§3) |
| §14.5 no case data in logs | Default log lines hold URLs only **(ran)**. Error pages are the leak, to the browser (§9) |
| §16.1 no SSR, no second runtime | Map #50 says these are to be amended; ADR 0003 and §16.1 still say so |
| §16.2 base URL from configuration | The adapter builds `rawUrl` from `PUBLIC_BASE`, not `Host` **(ran)** |
| §16.2 `/d/<token>` fixed, server-rendered | axum-owned; Node not on the path **(ran)** |
| ADR 0003, Extract collectable if front end fails | Holds with Node dead, given a front end other than Node **(ran)** |

## Open points

Facts missing or left unverified:

1. Why a stale tab's `content.dat` 200 left the old content in place, and whether `Platform.elm`'s full-load path is ever reached for a client navigation: not isolated.
2. Why the default cache strategy never served a fresh entry: not isolated.
3. Why the announcer says `Page not found`, and whether it happens in every app: not isolated, nothing filed.
4. Whether `Model mismatch` on a failed follow-up `data` is intended: **not stated**. Nothing filed.
5. Whether the bundle's SIGTERM handler is meant to apply to a long-lived server: **not stated**.
6. Not run: real Cloudflare Tunnel, Docker Compose, TLS and `Secure` cookies, real sessions and sliding, `axum-client-ip`, `elm-pages dev`, gzip from axum, a `Set-Cookie` on a 3xx from axum, multipart bodies through the hop, a Node `fetch` abort, a long soak, a real database, the Node-proxies-`/api` shape.

Decisions for the owner, with no picks here:

7. Which front end routes `/api` and `/d`: axum, or an edge's path rules. They fail differently (§7).
8. Screens through `data` and `action`, or direct `/api` after hydration, or a mix. The first keeps forms working without JavaScript; the second has no Node hop and no no-JS path (§4, §5).
9. `BackendTask.Http` with a `Set-Cookie` split, or `BackendTask.Custom` with Node's `fetch`. They differ on cache, abort, logging and compile-time checking (§2, §3, §9).
10. How to answer a failed follow-up `data`, a stuck link click and an unconfirmed Decision after a deploy (§7).

## Sources

| Source | Used for |
|---|---|
| `github.com/dillonkearns/elm-pages` @ `a1451f36e`: `generator/src/request-cache.js`, `generator/src/render.js`, `generator/static-code/elm-pages.js`, `generator/static-code/frozen-views-client.js`, `codegen/GenerateMain.elm`, `src/BackendTask/Http.elm`, `src/Server/{Request,Response}.elm`, `src/Pages/Internal/Platform.elm`, `generator/template/`, `examples/end-to-end/app/Route/Login.elm` | Behaviour, API |
| Same repo, `examples/docs/content/docs/15-adapters.md` | Adapter shape |
| `docs/research/elm-pages-full-stack.md` on `research/elm-pages-full-stack` | Everything already known |
| `npm audit` advisories GHSA-3g3m-2fjr-6pw4, GHSA-g3hv-75wg-m249 | Runtime set |
| This repo: `docs/http-api.openapi.yaml`, `docs/spec.md` (§7.1, §10.5, §14.5, §16.1, §16.2), `docs/port-disagreements.md` (D5, D6), ADR 0003, issues #10, #50, #51 | Requirements |
| Throwaway trial: elm-pages 3.5.1, Elm package 12.3.0, Lamdera 0.19.1-1.4.0, Vite 7.3.7, make-fetch-happen 15.0.6, axum 0.8.9, reqwest 0.12.28, playwright-core 1.57.0 on Chromium, `gcr.io/distroless/nodejs22-debian12:nonroot`. Not committed | Everything marked **(ran)** |
