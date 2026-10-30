# PROTOTYPE, throw me away: a daisyUI theme for the Port

Answers wayfinder ticket #14 ("A daisyUI theme for the Port"). Lives only on
the `prototype/daisyui-theme` branch; `main` keeps the decision, not this code.

```sh
cd apps/web/prototype-daisyui-theme
npm install --no-package-lock
npm start            # http://localhost:4300/?variant=A|B|C&screen=request|queue
```

← / → or the floating bar switch variants. "Server build →" opens a stand-in
for the askama `/d/<token>` page, styled only by `dist/server.css`.

## What it shows

- `theme.css` is the one theme source. `app.css` (the Elm app's build) and
  `server.css` (the templates-only build) both `@import` it. Both build with
  Tailwind 4.3.3 + daisyUI 5.7.50; the server build tree-shakes to the
  template's classes (33 KB vs 109 KB).
- Variants (the markup is identical; only the theme differs):
  - **A**: slots + system shape + ~85 lines of per-component overrides. Looks like today.
  - **B**: slots + system radii, sizes and flat depth; daisyUI draws the components as they ship.
  - **C**: colour slots only, daisyUI's light-theme shape.

## Findings

1. daisyUI 5 emits its components into Tailwind's **`utilities`** layer. An
   override in `@layer components` silently loses; it must go in `utilities`.
2. daisyUI's input edge is `base-content` at 20%, about 1.5:1. That fails
   WCAG 1.4.11 (system.md's `input` token exists for exactly this). B and C fail; A fixes it.
3. One `--size-field` drives fields **and** buttons: 44px fields and 40px
   buttons can't both come from the theme.
4. No slot for: `pending-bright`, the washes, `inert` (put in `info`), the 20px
   card radius, shadows. They live as Tailwind `@theme` tokens in the same file.
5. `-soft` washes are an 8% mix, not the system's wash values (pending comes out greyish).

## The four emails

Emails can't use daisyUI or CSS variables. Every colour in `theme.css` is a hex
literal, so the API crate's `build.rs` (which already generates the copy
functions) reads `--color-*: #rrggbb;` lines from `theme.css` and generates a
`theme::colour` module of `&'static str` consts that the askama email templates
inline. This replaces `apps/api/src/design/b-tokens.ts` and its spec test: a
renamed or deleted token is a compile error in the email templates.
