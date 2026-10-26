# elm-land with Tailwind, daisyUI and translated copy

- **Question:** How does an elm-land app take Tailwind CSS + daisyUI (build pipeline, class scanning over `.elm`, custom themes, self-hosted fonts)? What are the established ways to put translated copy into Elm, given that the copy lives in `messages/th.json` / `en.json` (inlang message format)? Can a server-rendered Rust page outside the SPA use the same compiled CSS? How would axum serve elm-land's output with an SPA fallback?
- **Issue:** #3 (map #1). Feeds #12 (the delivery page outside the SPA), #13 (translated copy in the Elm app), #14 (a daisyUI theme for the Port), #16 (toolchain, image and CI for the Port).
- **Date:** 2026-10-26
- **Primary sources:** the elm-land repo (`projects/cli/src/effects.js`, `examples/19-tailwindcss`, `docs/`) and its npm metadata; tailwindcss.com docs and the tailwindcss `CHANGELOG.md`; daisyui.com docs; the travelm-agency and elm-i18next repos and package source; elm-i18next-gen's README; tower-http 0.7.1 docs and source; axum's `examples/static-file-server`. This repo's `messages/*.json`, `project.inlang/settings.json`, `apps/web`, `apps/api/src/i18n/copy-catalogue.ts` and `apps/api/src/delivery/pages.ts`.
- **Run, not just read:** two throwaway builds in a scratch directory. One was elm-land 0.20.1 + tailwindcss/@tailwindcss/postcss 4.3.3 + daisyUI 5.7.50 + @fontsource/ibm-plex-sans-thai 5.3.0. The other ran travelm-agency 3.8.0 over copies of this repo's two catalogue files. Results are marked **(ran)**. ⚠️ marks what was not verified.

This note reports facts. It decides nothing; the open choices are listed per ticket at the end.

## Summary

- **Tailwind v4 + daisyUI 5 work in elm-land through PostCSS, not through the Vite plugin.** elm-land calls Vite with `configFile: false` and a fixed plugin list, so `@tailwindcss/vite` cannot be added. A `.postcssrc.json` naming `@tailwindcss/postcss`, plus `import "./styles.css"` in `src/interop.ts`, builds **(ran)**.
- **Class scanning of `.elm` needs no configuration.** Tailwind v4 scans every non-ignored file as plain text. daisyUI and utility classes written in `class "…"` / `classList` strings were all emitted **(ran)**.
- **fontsource breaks if it is `@import`ed inside the Tailwind stylesheet.** Its relative `url(./files/…)` came out unrewritten and the font files were not emitted. Importing the fontsource CSS from `interop.ts` instead makes Vite copy and hash the fonts **(ran)**.
- **Elm i18n splits three ways.** (a) Compile-time generated modules with typed functions (travelm-agency: it reads our files nearly as-is, **(ran)**). (b) Runtime JSON with string keys (elm-i18next). (c) A hybrid that generates typed accessors over runtime JSON (elm-i18next-gen: its `{{…}}` placeholders don't match our `{…}`).
- **Sharing one CSS file with a server-rendered page is mechanically possible, but the bundle filename is hashed and elm-land writes no manifest.** Today's `/d/<token>` page inlines its styles on purpose (`pages.ts`, ADR 0003's guarantee).
- **elm-land builds a plain static SPA into `dist/`.** tower-http `ServeDir` with an `index.html` fallback serves it. `ServeDir` 0.7.1 sends `ETag`/`Last-Modified` but no `Cache-Control`.

## 1. elm-land's build is Vite, with no user Vite config

`projects/cli/src/effects.js` (elm-land `main`, matching the 0.20.1 npm release):

- `elm-land server` calls Vite's `createServer({ configFile: false, root: process.cwd(), publicDir: 'static', envPrefix: 'ELM_LAND_', server: { …, proxy }, plugins: [ElmVitePlugin(…), ElmLandIndexHtml.plugin()], appType: 'spa' })`.
- `elm-land build` calls Vite's `build({ configFile: false, root: '.elm-land/server', publicDir: 'static', build: { outDir: '../../dist' }, plugins: [ElmVitePlugin({ mode: 'minify' })] })`.
- **`configFile: false` with a hard-coded `plugins` array means a project cannot add Vite plugins or set Vite options** (no `build.manifest`, no `assetsDir`, no custom plugin). The extension points are a PostCSS config file, which Vite still discovers on its own (see §2), `static/`, `elm-land.json`, and `src/interop.(ts|js)`.
- `src/interop.ts` is picked up by the generated `main.js` (`import.meta.glob('../../src/interop.ts')`). It may export `flags` (async, gets `{ env }`) and `onReady({ app, env })`. Any CSS it imports goes through Vite's CSS pipeline. (`templates/_elm-land/server/main.js`)
- `index.html` is generated from `elm-land.json` → `app.html` (`title`, `meta`, `link`, `script`, `attributes`). (`toIndexHtmlFile` in `effects.js`; `docs/reference/elm-land-json.md`)
- `app.proxy` is passed to Vite's dev proxy (`{"/api": "http://localhost:5000"}`). It is **dev-server only**, and JS functions such as `rewrite` are not possible. (`docs/reference/elm-land-json.md`)
- **Pinned versions:** elm-land 0.20.1 depends on `vite: 5.2.8` (exact), `@lydell/elm 0.19.1-14`, `vite-plugin-elm-watch 1.3.3`, `typescript 4.9.3` (`npm view elm-land@0.20.1 dependencies`). The npm package was last published 2024-04-15, and the GitHub repo was last pushed 2025-03-10 (`gh api repos/elm-land/elm-land`). Relevant to #16: the Vite major is fixed by elm-land, not by us.

## 2. Adding Tailwind v4 + daisyUI 5

**elm-land's own example is Tailwind v3.** `examples/19-tailwindcss` has `.postcssrc.json` with `tailwindcss` + `autoprefixer`, `tailwindcss ^3.4.0`, `src/interop.ts` = `import "./styles.css"`, and a `tailwind.config.ts` whose `content` lists `./src/**/*.{js,elm,ts,css,html}` and `./.elm-land/**/*`.

**The v4 equivalent (ran, builds clean):**

```jsonc
// .postcssrc.json (project root, next to elm-land.json)
{ "plugins": { "@tailwindcss/postcss": {} } }
```

```css
/* src/styles.css */
@import "tailwindcss";
@plugin "daisyui" { themes: false; }
@plugin "daisyui/theme" { name: "dds"; default: true; color-scheme: light; /* --color-*, --radius-*, … */ }
@theme { --font-sans: 'IBM Plex Sans Thai', 'IBM Plex Sans', system-ui, sans-serif; }
```

```ts
// src/interop.ts
import "@fontsource/ibm-plex-sans-thai/400.css"; // see §4: here, not in styles.css
import "./styles.css";
```

- Tailwind's PostCSS install is `npm install tailwindcss @tailwindcss/postcss postcss`, with `"@tailwindcss/postcss": {}` as the only plugin (https://tailwindcss.com/docs/installation/using-postcss). daisyUI 5 is `npm i -D daisyui@latest` plus `@plugin "daisyui";` (https://daisyui.com/docs/install/). daisyUI has no Elm or axum guide. It does list "Tailwind CSS Standalone without Node" (below).
- The build root is `.elm-land/server`, yet the project-root `.postcssrc.json` was found: daisyUI's banner printed and its classes were emitted **(ran)**.
- Output: a single `dist/assets/index-<hash>.css` (17.7 KB minified in the probe) beside `index-<hash>.js`. `dist/index.html` links both with absolute `/assets/…` paths **(ran)**.
- ⚠️ Dev-server rebuild when an `.elm` file changes (Tailwind's PostCSS dependency messages under elm-land's Vite) was not exercised. Only `elm-land build` was run.
- **Browser floor:** Tailwind v4 targets Chrome 111, Safari 16.4, Firefox 128 (https://tailwindcss.com/docs/compatibility).
- Versions on npm today: `tailwindcss` / `@tailwindcss/postcss` / `@tailwindcss/cli` 4.3.3, `daisyui` 5.7.50.

### Standalone (no Node) path

daisyUI documents it at https://daisyui.com/docs/install/standalone/. Download the `tailwindcss-<os>-<arch>` binary, and fetch `daisyui.mjs` and `daisyui-theme.mjs` from daisyUI's GitHub release. Then `@plugin "./daisyui.mjs";`, add `@source not "./daisyui{,*}.mjs";`, and run `./tailwindcss -i input.css -o output.css`. elm-land itself still needs Node, so this matters only for a CSS build that runs outside elm-land (§5).

## 3. Class scanning over `.elm` sources

From https://tailwindcss.com/docs/detecting-classes-in-source-files:

- Tailwind treats source files **as plain text** and scans **every file** except: files in `.gitignore`, `node_modules`, binaries, CSS files, and lock files. There is no extension list and no config file in v4.
- The scan starts from the **current working directory**. `@import "tailwindcss" source("../src")` sets the base. `@source "<path>"` adds a path relative to the stylesheet, `@source not "<path>"` excludes one, `source(none)` turns auto-detection off, and `@source inline("…")` safelists (brace-expanded).
- Class names must appear **whole** in the source. String concatenation such as `"bg-" ++ color` is invisible to the scanner.

Consequences observed **(ran)**:

- `class "btn btn-primary font-sans"` and `classList [ ( "alert alert-success", True ), ( "mt-4", False ) ]` in `src/Pages/Home_.elm` were all emitted. `mt-4` was emitted too, although its flag is `False`: plain-text scanning.
- elm-land's scaffolded `.gitignore` lists `/.elm-land` and `/dist` (`examples/19-tailwindcss/.gitignore`), so generated code is not scanned. User code lives in `src/`, so nothing is lost. The v3 example's explicit `./.elm-land/**` entry has no v4 counterpart unless one is added with `@source`.
- Tailwind honours `.gitignore` only inside an initialised git repo (CHANGELOG: "Don't look at .gitignore files outside initialized repos", #15941). In this monorepo the root `.gitignore` applies too.
- If the Rust server's templates must contribute classes (§5), they lie outside the elm-land cwd and need an explicit `@source "<path to templates>"`.

## 4. Custom daisyUI theme and self-hosted fonts

### Theme (https://daisyui.com/docs/themes/)

- Built-ins are chosen by `@plugin "daisyui" { themes: light --default, dark --prefersdark; }`. `themes: false` disables every built-in "and remove[s] all daisyUI colors".
- A custom theme is `@plugin "daisyui/theme" { name: "…"; default: true; prefersdark: false; color-scheme: light; … }` with these variables:
  - `--color-base-100/200/300`, `--color-base-content`
  - `--color-{primary,secondary,accent,neutral,info,success,warning,error}` and each one's `-content`
  - `--radius-selector`, `--radius-field`, `--radius-box`
  - `--size-selector`, `--size-field`, `--border`, `--depth`, `--noise`
- The docs' examples use `oklch(…)`, but hex is accepted. `#3b5bfd` came through unchanged **(ran)**.
- A theme that customises a built-in inherits the variables it leaves out. **A new theme with `themes: false` does not.** The probe's `[data-theme=dds]{…}` block held only the six variables declared, and no `--color-secondary` existed anywhere in the output **(ran)**. So a from-scratch theme has to declare every variable it relies on.
- `data-theme="…"` on any element applies a theme to that subtree, and themes nest.
- Tailwind `@theme` variables generate utilities (`--font-sans` → `font-sans`). `:root` variables do not. `@theme inline` is for variables that reference other variables (https://tailwindcss.com/docs/theme).
- What #14 maps from: today's tokens are plain `:root` custom properties in `apps/web/src/styles.css` (`--primary #3b5bfd`, `--background #e4e6ea`, `--card #fff`, `--success`, `--pending`, `--failed`, four radii, five shadows, `--font-sans`). The file says the design's Tailwind `@theme` is "design-tool scaffolding, not the implementation" (`docs/design/README.md`). daisyUI has slots for some of these tokens but not for others: there are no slots for `pending`/`inert`/`*-wash`, shadows, or a fourth radius.

### Fonts (@fontsource 5.3.0, already a dependency of `apps/web`)

- `@fontsource/ibm-plex-sans-thai/400.css` declares one `@font-face` per subset (cyrillic-ext, thai, latin-ext, latin), each with a `unicode-range`, `font-display: swap` and **relative** `src: url(./files/….woff2)` (package contents, `npm pack`). Angular loads these files today through `angular.json` `styles`.
- **Through elm-land:**
  - `@import "@fontsource/…/400.css"` *inside* the Tailwind stylesheet was inlined by Tailwind with `url(./files/…)` **left as-is**, and no font files were emitted. The font would 404 **(ran)**.
  - `import "@fontsource/…/400.css"` from `src/interop.ts` made Vite rewrite the URLs to `/assets/ibm-plex-sans-thai-thai-400-normal-<hash>.woff2` and copy the files **(ran)**. The smallest subset (cyrillic-ext) produced no file. ⚠️ Probably inlined as a data URI under Vite's default 4 KB asset-inline limit; not confirmed.
  - The Tailwind CHANGELOG's *Unreleased* section has "Ensure `@tailwindcss/postcss` rewrites relative `url(…)`s of stylesheets that are inlined before it runs … in optimized builds" (#20576). ⚠️ Whether that changes the in-stylesheet case above was not checked against an unreleased build.
- **Through the Tailwind CLI (standalone or `@tailwindcss/cli` 4.3.3)**, the same `@import` also leaves `url(./files/…)` unrewritten **(ran)**. The CLI does not copy font files, so they would have to sit at `<output dir>/files/`.

## 5. The same CSS on a server-rendered page outside the SPA

Facts that bear on #12:

- **Today the server pages deliberately share nothing with the SPA.** `apps/api/src/delivery/pages.ts`: "these pages load no bundle, so they cannot share the SPA's stylesheet or its self-hosted fonts, and fall back to the platform's Thai face". The B tokens are inlined in a `<style>`. ADR 0018 requires that "no Angular bundle, no client script is required to read it or press its button" (ADR 0003's guarantee: an Extract stays collectable if a front-end asset fails). A `<link rel="stylesheet">` brings in no script, but the page's *styling*, though not its function, would then depend on a front-end asset.
- **The Vite bundle's CSS name is content-hashed** (`/assets/index-<hash>.css`), and elm-land offers no way to turn on Vite's `build.manifest` (§1). A server that wants that exact file has to find its name some way. Two routes: read it from `dist/index.html`'s `<link rel="stylesheet">`, or glob `dist/assets/index-*.css`, at build time or at startup. Both depend on elm-land's output shape **(ran: output shape as stated)**.
- **A separate, fixed-name CSS build is possible.** Run the Tailwind CLI (or standalone binary) with daisyUI over the same theme source. Write the output into `static/` (elm-land copies `static/` into `dist/` verbatim as Vite's `publicDir`), or anywhere the server serves. A fixed name means caching has to rely on validators (below) or a version query string. For the font URLs, see §4.
- **Either way, classes used only in server templates must be scanned.** Add `@source "<templates dir>"` (relative to the stylesheet), or the server page's classes will be absent (§3).
- daisyUI components are CSS-only class names (`btn`, `alert`, `card`, …). Those used in the probe needed no JS. ⚠️ Interactive daisyUI components (dropdown, modal) were not checked for script-free behaviour.

## 6. Translated copy in Elm

### What our catalogue looks like (this repo)

- `messages/th.json` and `messages/en.json` each have 415 messages plus a `"$schema"` key. **Every value is a flat string**, there are no `.` in keys, and the two files have identical key sets. Placeholders are single-brace `{name}`: 19 distinct names, 54 occurrences in `th.json`. No `<`, no variants or plurals.
- `project.inlang/settings.json`: `baseLocale: "th"`, `locales: ["th"]`. ADR 0010: copy is authored in English, `en.json` is "checked in and permanently maintained", and only Thai is ever served.
- The server side reads the same JSON directly, not through Paraglide. `apps/api/src/i18n/copy-catalogue.ts` replaces `/\{(\w+)\}/g` and feeds the four emails and both server-rendered pages (`/d/<token>`, `/link-expired`). Paraglide has no Elm output.

### Option A: compile-time generated module (travelm-agency)

- npm `travelm-agency` 3.8.0, published 2025-10-27. The repo (now `anmolitor/travelm-agency`; README.adoc) was last pushed 2026-02-09.
- Input files are named `[identifier].[language].[ext]` (`.json`, `.properties`, `.ftl`). JSON placeholders are `{curly}`, `<` starts inline HTML, and keys must match across languages unless `--fallback-language` or `--devMode` is used.
- **Inline mode** (`--inline`) generates `Language`, `init : Language -> I18n`, and one function per key. **Dynamic mode** (default) emits per-language JSON (optionally content-hashed with `--hash`) plus `loadMessages : (Result Http.Error (I18n -> I18n) -> msg) -> I18n -> Cmd msg` (needs `elm/http`). Fluent `.ftl` adds `Intl`-backed NUMBER/DATETIME.
- **Ran over copies of our files**, renamed `messages.th.json` / `messages.en.json`, untouched otherwise:
  - Inline output: `src/Translations.elm`, 130 KB, both languages as `case lang of En -> … ; Th -> …`.
  - Keys become camelCase functions (`requester_collect_attempts_value` → `requesterCollectAttemptsValue : { cap : String, used : String } -> I18n -> String`).
  - **`"$schema"` became an exported `schema : I18n -> String`.** It is harmless, but it is a stray function.
  - Dynamic + `--hash` emitted `messages.th.3348778545.json` (35.5 KB) and `messages.en.3448602975.json` (17.3 KB).
  - It called `npx elm-format` on its output, which fetched `elm-format@0.8.8`.
- Trade-offs:
  - For: typed keys and placeholders, so a missing key or a renamed placeholder is a compile error; no runtime loading in inline mode; reads our existing files.
  - Against: a codegen step in the build (#16), and the file rename or copy.
  - In inline mode the copy is baked into the JS bundle, so changing a string means rebuilding the SPA.
  - With only `messages.th.json` as input, `Language` would have one constructor. ⚠️ Not run that way.

### Option B: runtime JSON, string keys (elm-i18next)

- `ChristophP/elm-i18next` 4.2.1 (2022-09). It exposes `translationsDecoder`, `t : Translations -> String -> String` (a missing key returns the key itself), `tr`/`tf` with `Delims = Curly | Underscore | Custom (String, String)`, `customTr` for non-string (e.g. `Html`) replacements, and fallback languages (`src/I18Next.elm`, README).
- `Custom ("{", "}")` matches our placeholder syntax. The decoder accepts flat string objects, so `$schema` decodes as an ordinary key.
- Translations arrive via flags (in elm-land, `interop.ts` `flags`, then `Shared.init`'s `Result Json.Decode.Error Flags`; `docs/concepts/shared.md`) or via an HTTP fetch.
- Trade-offs:
  - For: no codegen, and copy changes need no Elm rebuild.
  - Against: keys and placeholder names are unchecked strings. A typo renders the key, with no compile error.

### Option C: typed accessors over runtime JSON (elm-i18next-gen)

- `elm-i18next-gen` 1.1.0 (npm, 2022-05; repo last pushed 2023-01) generates one Elm function per key that calls elm-i18next, from one "source" JSON.
- Its README lists "Allow parameter delimiter to be configured (currently hard-coded to `Curly`)" as a todo. Curly is `{{…}}`, which **does not match our `{…}`**.

### Option D: our own generator

The catalogue is 415 flat strings with `{name}` placeholders only. A generator from JSON to an Elm module is a small script. In Rust it could share the parser with the axum side's catalogue loader. No library involved; listed because the input format is this simple.

### Locale in the model

elm-land's `Shared.Model` is the place for app-wide state ("available to every page"), customised with `elm-land customize shared`. An `I18n` value (A) or `Translations` value (B, C) would live there. View functions receive it from `shared` (`docs/concepts/shared.md`). With Thai-only serving and no switcher (ADR 0010), a locale *switch* is not needed. The value in the model is still the handle the view reads copy through.

## 7. Serving elm-land's output from axum

- `elm-land build` writes a static SPA to `dist/`: `index.html`, `assets/index-<hash>.js`, `assets/index-<hash>.css`, hashed font files, and the contents of `static/` copied verbatim. The docs say every unknown path must be "directed to a single file: `dist/index.html`" (`docs/guide/deploying.md`; output shape **(ran)**).
- elm-land routes on the path unless `app.router.useHashRouting` is `true` (`docs/reference/elm-land-json.md`). With path routing, the server needs the fallback.
- axum's `examples/static-file-server/src/main.rs` shows the patterns:
  - `Router::new().route(...).fallback_service(ServeDir::new("assets").not_found_service(ServeFile::new("assets/index.html")))`: SPA from the root, explicit routes first.
  - A variant wraps `index.html` in `SetStatus(…, StatusCode::NOT_FOUND)` so client-routed paths return 404.
- tower-http 0.7.1 `ServeDir` (https://docs.rs/tower-http/latest/tower_http/services/struct.ServeDir.html, source `services/fs/serve_dir/open_file.rs` at tag `tower-http-0.7.1`):
  - `fallback` leaves the fallback's status as-is; `not_found_service` forces 404.
  - `append_index_html_on_directories` defaults to true.
  - `precompressed_gzip` / `precompressed_br` serve `.gz` / `.br` siblings.
  - Non-GET/HEAD requests get 405 unless `call_fallback_on_method_not_allowed`.
  - It sets **`ETag` and `Last-Modified`** and honours `If-None-Match` / `If-Modified-Since` / `Range`.
  - **It sets no `Cache-Control`.** Long-lived caching of `/assets/*` would come from a header layer.
- Routing order facts for one origin: the API routes, `/d/<token>`, `/d/<token>/archive` and `/link-expired` (server-rendered, ADR 0018) have to be matched before the SPA fallback, or the fallback answers them with `index.html`.
- In dev, elm-land's `app.proxy` can forward `/api` (and `/d`) to a local axum on another port (dev only, §1).

## Open questions per ticket (facts only, no picks)

- **#14 (theme):** which daisyUI slots our tokens map to. What to do with the tokens daisyUI has no slot for: `pending`, `inert`, `*-wash`, shadows, the fourth and fifth radius. Whether to start from `themes: false` and declare every variable (a new theme inherits nothing, §4).
- **#13 (copy):** compile-time typed (A, D) or runtime strings (B), or the hybrid (C) with a placeholder mismatch. Inline or loaded. How `$schema` and the `messages.<lang>.json` naming are handled if A is used.
- **#12 (delivery page):** keep inlined styles (today's ADR 0003 stance), link the hashed SPA CSS (needs name discovery), or link a separately built fixed-name CSS (needs `@source` over templates and font placement).
- **#16 (toolchain):** Node is required for elm-land (Vite 5.2.8 pinned). Tailwind runs as a PostCSS plugin inside it. There is an optional i18n codegen step, an optional second CSS build, and `elm-format` being fetched by travelm-agency.
