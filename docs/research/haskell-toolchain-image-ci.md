# Haskell toolchain, static image and CI for the Port

- **Question:** What does a current Haskell toolchain, image and CI look like for one Servant service? cabal vs stack vs Nix (and `ghcup`); pinning GHC and the package index; a multi-package layout; build caching in GitHub Actions; the container image (static musl binary vs dynamic on distroless), size and build time; formatting and linting; dependency audit and updates; how dbmate ships beside the binary.
- **Issue:** rawinan-soma/dds-sharing-FP#40 (map #35; blocks #47). Prior Rust answer: #16.
- **Date:** 2026-11-03
- **Primary sources:** GHCup metadata (`haskell/ghcup-metadata`, `ghcup-0.1.0.yaml`) and the GHCup user guide; the cabal user guide (readthedocs, "stable") and cabal-install source at `cabal-install-v3.18.2.0`; Stack docs; Stackage snapshot API; Hackage package pages and `.cabal` files; `haskell-actions/*` READMEs and release API; the `haskell/security-advisories` repo; `MangoIV/cabal-audit`; GitHub's Dependabot ecosystem table; Renovate's manager and datasource pages; the OSV schema; Docker Hub and gcr.io registry APIs; `GoogleContainerTools/distroless` READMEs; Alpine package contents search; Debian package page for `libpq5`; `amacneil/dbmate` README, Makefile, Dockerfile, CI and release API; Docker's GitHub Actions cache docs; the `actions/runner-images` Ubuntu 24.04 readme. Every claim names its source. Anything a source did not say is marked **not stated**.

This note lists facts. It recommends nothing. #47 (and the grilling tickets on #35) make the decisions.

## Summary

- **No Dependabot ecosystem exists for cabal, Stack or Hackage.** The Haskell options for update PRs are Renovate (`haskell-cabal` manager, reads `build-depends` in `.cabal` files only) or nothing. For advisories there is the Haskell Security Advisory DB (HSEC, ~40 advisories, exported to OSV), read by `cabal-audit` (on Hackage, 0.3.0.0).
- **GHC versions today:** GHCup tags 9.10.3 *Recommended* and 9.14.1 *Latest*; 9.12.4 is the newest 9.12. Stackage LTS 24.65 is GHC 9.10.3, nightly is GHC 9.12.4. `postgresql-simple` 0.7.0.1 (pulled in by `persistent-postgresql`) declares `base <4.22`, which excludes GHC 9.14 without `allow-newer`.
- **Pinning in cabal is three files' worth of fields:** `with-compiler` (or the GHC on `PATH`), `index-state` in `cabal.project`, and a `cabal.project.freeze` that `cabal freeze` writes with every version, every flag and the `index-state`.
- **Static musl:** GHC publishes official Alpine bindists for 9.10.3, 9.12.4 and 9.14.1, and `benz0li/ghc-musl` publishes ready Docker images (including GMP-free `int-native` tags). Alpine `main` has `libpq.a` (`libpq-dev`) and `libssl.a` (`openssl-libs-static`). cabal's `executable-static: True` passes `-static -optl=-static`.
- **Dynamic on distroless:** no distroless image contains `libpq` or `libgmp`; Debian's `libpq5` pulls `libgssapi-krb5-2`, `libldap-2.5-0` and `libssl3`.
- **dbmate** ships as a single statically linked Linux binary per arch (v2.37.0, ~34 MB) and as an Alpine-based image on Docker Hub and GHCR; `dbmate dump` needs `pg_dump` on `PATH`, other commands do not.
- **Build time and final binary size for a Servant + persistent service are not stated by any primary source read.** Only base-layer sizes are measured below.

## 1. Toolchain managers: GHCup, cabal, Stack, Nix

### Versions (2026-11-03)

| tool | version | source |
|---|---|---|
| GHC *Recommended* (GHCup) | 9.10.3 (`base-4.20.1.0`) | `ghcup-0.1.0.yaml` `viTags` |
| GHC *Latest* (GHCup) | 9.14.1 (`base-4.22.0.0`, released 2025-12-18) | same |
| GHC 9.12 line | 9.12.4 (`base-4.21.2.0`, released 2026-03-26) | same |
| cabal-install | 3.16.1.0 *Recommended*, 3.18.2.0 *Latest* (GitHub release 2026-09-30) | same; `haskell/cabal` release API |
| Stack | 3.11.1 (*Latest*, *Recommended*; release 2026-06-13) | same; `commercialhaskell/stack` release API |
| GHCup | 0.2.6.2 | same |
| HLS | 2.14.0.0 *Recommended*, 2.15.0.0 *Latest* | same |
| GitHub `ubuntu-24.04` runner preinstalls | GHC 9.14.1, Cabal 3.18.2.0, GHCup 0.2.6.2, Stack 3.11.1 | `actions/runner-images` `Ubuntu2404-Readme.md` |

- **GHCup moved to Codeberg.** `github.com/haskell/ghcup-hs` is archived with the description "MOVED TO CODEBERG"; the GHCup guide links `codeberg.org/haskell/ghcup-hs` and `codeberg.org/haskell/ghcup-metadata`. The GitHub `haskell/ghcup-metadata` repo is not archived and was pushed 2026-10-21.
- **GHCup in CI:** the guide documents `BOOTSTRAP_HASKELL_NONINTERACTIVE=1` and `BOOTSTRAP_HASKELL_MINIMAL=1` for a non-interactive install, and links the action `haskell/ghcup-setup` (v1.2.3, 2026-04-03), whose README shows `ghcup install ghc --set <version>`. `ghcup install ghc` with no version installs the *recommended* one (GHCup guide).

### cabal

- **Multi-package project:** `cabal.project` must list a non-empty `packages:` (files, directories or globs such as `*/*.cabal`); without a `cabal.project`, cabal uses `packages: ./*.cabal` (cabal user guide §2.2).
- **One package, several components:** "A package can have no more than one unnamed library", any number of executables and test suites, and named sublibraries; sublibraries are private by default, and `visibility` exists since cabal-version 3.0 (package description §1.2.3–1.2.5). Shared build info goes in **common stanzas** with `import:`, since cabal-version 2.2 (§1.2.12).
- **`build-tool-depends: pkg:exe >= x`** (since Cabal 2.0) declares a build-time executable such as `hspec-discover` (§1.2.7).
- **`index-state`** fixes the Hackage index the solver sees: `HEAD` (default), a Unix timestamp or an ISO 8601 UTC time, per repository since 3.4; the guide says it is "particularly useful in combination with freeze-files" (project file §2.6).
- **`cabal freeze`** "records all of the versions and flags that are picked by the solver" into `cabal.project.freeze`, in `cabal.project` syntax (`constraints:`). The cabal-install 3.18.2.0 source (`CmdFreeze.hs`, `projectFreezeConfig`) also writes `projectConfigIndexState`, so the freeze file carries the `index-state` too. The freeze file sits between `cabal.project` and `cabal.project.local` in precedence.
- **`cabal outdated --v2-freeze-file --exit-code`** reports dependencies newer than the freeze file and exits non-zero (commands doc).
- **Static linking:** project option `executable-static` (default `False`) "links all dependent libraries into executables statically, including libc", by passing `-static` and `-optl=-static` (project file §2.7.4).

### Stack

- A project pins a Stackage snapshot (`snapshot:`/`resolver:` in `stack.yaml`). Today: **LTS 24.65 = GHC 9.10.3; nightly-2026-10-28 = GHC 9.12.4** (stackage.org snapshot pages and `snapshots.json`).
- `stack.yaml.lock` holds "completed package locations for extra-deps" and the snapshot; Stack rewrites it when the config changes; "These files can be stored in source control" (Stack docs, lock files).
- What LTS 24.65 holds for this stack (its `cabal.config`): `servant-server 0.20.3.0`, `esqueleto 3.6.0.3`, `persistent 2.17.1.0`, `persistent-postgresql 2.13.7.0`, `postgresql-libpq 0.11.0.0`, `warp 3.4.9`, `hlint 3.10`, `aeson 2.2.5.1`, `scientific 0.3.10.0`. No `fourmolu` or `ormolu` line appeared in LTS 24.65 (nightly has `fourmolu 0.19.0.1`).
- Hackage latest, for comparison: `persistent 2.18.1.0`, `persistent-postgresql 2.14.3.0`, `warp 3.4.20`, `servant-server 0.20.3.0`, `esqueleto 3.6.0.3` (Hackage `/preferred`).

### Nix

- Dependabot has a `nix` ecosystem (version updates only, for flake inputs; Dependabot table). Nothing else about a Nix-based build was researched; **not stated** here.

### GHC compatibility of the stack's libraries (latest Hackage revisions)

| package | `base` bound | GHC it excludes |
|---|---|---|
| `servant`, `servant-server` 0.20.3.0 | `>=4.16.4.0 && <4.23`; tested-with up to 9.12.1 | none of 9.10–9.14 |
| `esqueleto` 3.6.0.3, `persistent` 2.18.1.0, `persistent-postgresql` 2.14.3.0, `warp` 3.4.20 | `<5` | none |
| `postgresql-simple` 0.7.0.1 (only revision, 2025-08-02) | `>=4.12.0.0 && <4.22` | **GHC 9.14** (base 4.22) |
| `postgresql-libpq` 0.11.0.0 r2 | `<4.23`; tested-with up to 9.14.1 | none |

`persistent-postgresql` 2.14.3.0 depends on `postgresql-simple >=0.6.1 && <0.8` and `postgresql-libpq >=0.9.4.2 && <0.12` (its `.cabal`).

## 2. GitHub Actions: setup and caching

- **`haskell-actions/setup`** v2.12.1 (2026-09-22). Inputs `ghc-version`, `cabal-version`, `enable-stack`, `stack-version`, `cabal-update`; outputs include the resolved `ghc-version`/`cabal-version` and the **`cabal-store`** path. It uses the runner's preinstalled GHC/cabal when the version matches, otherwise GHCup. A short version (`9.12`) resolves to the newest point release in its `versions.json`, which lists up to GHC 9.14.1, cabal 3.18.1.0 and Stack 3.11.1. cabal older than 3.10 is refused because of HSEC-2023-0015 (README).
- **Its model caching workflow** (`docs/examples.md`): `cabal configure --enable-tests …` then `cabal build all --dry-run` to produce `dist-newstyle/cache/plan.json`; `actions/cache/restore` on `steps.setup.outputs.cabal-store` keyed by OS + GHC + cabal version + `hashFiles('**/plan.json')`, with a prefix `restore-keys`; `cabal build all --only-dependencies` on a miss; `actions/cache/save` right after (so failed builds still save dependencies); then `cabal build all`, `cabal test all`, `cabal check`. It caches the **store**, not `dist-newstyle`. Caching `dist-newstyle` is **not stated** in that example.
- **Generator:** `haskell-CI/haskell-ci` (454 stars, pushed 2026-10-10) generates CI workflows from the `.cabal` `tested-with` field. Not researched further.
- **Docker builds in Actions:** Docker's docs say "BuildKit doesn't preserve cache mounts in the GitHub Actions cache by default"; the named workaround is `reproducible-containers/buildkit-cache-dance`. The layer cache uses `cache-from: type=gha` / `cache-to: type=gha,mode=max`. So a `RUN --mount=type=cache,target=/root/.cabal/store` in the Dockerfile is not reused between CI runs without that workaround; a dependency-only layer (copy `.cabal`/`cabal.project*`/freeze, `cabal build --only-dependencies`, then copy sources) is cached by the gha layer cache. (The second sentence is my inference from the docs, the cargo-chef analogue in #16.)

## 3. Formatting and linting

| tool | latest | parser | binaries | Action |
|---|---|---|---|---|
| `ormolu` | 0.9.0.0 (2026-10-01) | — | linux x86_64, darwin, windows zips | `haskell-actions/run-ormolu` v21 — **archived** (2026-08-04), though the ormolu README still calls it "the recommended way" |
| `fourmolu` | 0.21.0.0 (2026-10-04) | `ghc-lib-parser >=9.14 && <9.15` | linux x86_64 **and arm64**, darwin zips | `haskell-actions/run-fourmolu` v13 (2026-06-24), not archived; its README says to pin an exact `version` because new fourmolu releases "are often not backwards compatible" |
| `hlint` | 3.10 (2025-02-02; still latest on Hackage and GitHub) | `ghc-lib-parser == 9.12.*` | linux x86_64, osx, windows | `haskell-actions/hlint-setup` + `hlint-run` (v2.4.10, 2024-06-20; repos pushed 2026-11-02). `hlint-setup` downloads the GitHub release binary; its `version` input defaults to `3.5`. `hlint-run` takes `path` and `fail-on: warning`. |

- Both formatters read `default-extensions` from the `.cabal` file (ormolu README; run-ormolu `respect-cabal-files`).
- hlint 3.10 parses with the GHC 9.12 parser (its `.cabal`), and its `tested-with` is GHC 9.8–9.12. Whether it accepts GHC 9.14-only syntax is **not stated**.

## 4. Dependency audit and update bots

### Dependabot

- The ecosystem table lists no cabal, Stack, Haskell or Hackage entry (between *Bun* and *Cargo*, *Hex* and *Julia*, *poetry* and *pre-commit*). It does list `docker`, `github-actions`, `npm` (pnpm), `nix` and **`elm`** (`elm-package`, Elm 0.19) (GitHub Docs, "Supported ecosystems and repositories"). So Dependabot could cover `apps/web`'s `elm.json` and the Node islands, but not the Haskell packages.

### Renovate

- Manager **`haskell-cabal`**: matches `/\.cabal$/`; extracts `build-depends`; "The dependencies of all components are mushed together in one big list"; does not handle `pkgconfig-depends` or `build-tool-depends`; datasource `hackage`, default versioning `pvp`. `cabal.project`, `index-state` and freeze files are **not mentioned** (Renovate docs).
- `haskell-actions/setup` documents a Renovate recipe: a `commitBody` that writes a `New-Versions:` git trailer, and a CI step that appends those as `constraints:` to `cabal.project` on PRs. It also points to Renovate's `osvVulnerabilityAlerts` for HSEC advisories and to `ignoreDeps` for GHC boot libraries (`docs/examples.md`). Renovate's own Hackage page does not mention vulnerability alerts.

### The Haskell Security Advisory DB (`haskell/security-advisories`)

- Advisories are Markdown with TOML front matter; IDs are `HSEC-YYYY-NNNN`; affected packages are a Hackage `package` or a `ghc-component`; affected ranges are `introduced`/`fixed` (README). Public domain; tooling in `code/` (`hsec-tools`, `hsec-sync`, `osv`) is BSD-3.
- The repo has branches `generated/osv-export` and `generated/gh-pages`. The OSV schema lists ecosystems **`Hackage`** and **`GHC`** and the prefix **`HSEC`** ("Haskell Security Advisory Database").
- Size: 40 advisory files under `advisories/hackage` and `advisories/ghc` (git tree, 2026-11-03), last advisory HSEC-2026-0012 on 2026-10-21. Relevant to this stack: `aeson` (HSEC-2023-0001), `base` (HSEC-2023-0007, HSEC-2024-0006), `process` (HSEC-2024-0003, Windows), `cabal-install` (HSEC-2023-0015, HSEC-2025-0005), `crypton`/`crypton-x509-store`, `jwt` (HSEC-2026-0010), and **`scientific`** (HSEC-2026-0011, fixed 0.3.9.0; HSEC-2026-0012, introduced 0.3.9.0, fixed 0.3.10.0). LTS 24.65 and nightly both pin `scientific 0.3.10.0`.

### `cabal-audit` (`MangoIV/cabal-audit`)

- Scans the cabal **build plan** against the advisory DB (clones `haskell/security-advisories` by default, or `--file-path`/`--repository`), prints fix versions, `--json`, and **`--fail-on-warning`** "Exits with an error code if any advisories are found in the build plan" (README).
- Install: Hackage `cabal-audit` 0.3.0.0, `nix run github:mangoiv/cabal-audit`, or a static binary from the `nightly` GitHub release (dated 2024-05-06). 31 stars; last commit 2026-10-07 ("Bump security-advisories pin").
- For GitHub code scanning, the README points to `blackheaven/haskell-security-action` (9 stars, used as `@master`), which runs cabal-audit and uploads SARIF; it needs GHC on `PATH`.
- Stack projects: the README speaks only of cabal plans; Stack support is **not stated**.

## 5. The container image

### Builder images (Docker Hub API, amd64 compressed size)

| image | size | notes |
|---|---|---|
| `haskell:9.12.4-slim-bookworm` | 781 MB | official image; tags exist for 9.14.1, 9.12.4, 9.10.3 on bookworm, 9.8/9.6 on bullseye |
| `benz0li/ghc-musl:9.12.4` / `9.12.4-int-native` | 852 / 851 MB | Alpine, amd64/arm64/riscv64; Stack included since 2023 |
| `benz0li/ghc-musl:9.14.1-int-native` | 923 MB | |

### Static (musl / Alpine)

- **Official GHC Alpine bindists** exist for 9.10.3 and 9.12.4 (`x86_64-alpine3_12`, `x86_64-alpine3_20`, `aarch64-alpine3_18`) and 9.14.1 (`x86_64-alpine3_22`, `aarch64-alpine3_22`) on `downloads.haskell.org` (GHCup metadata), so GHCup can install GHC inside an Alpine builder.
- **`benz0li/ghc-musl`** describes itself as "*Unofficial* and *untested* binary distributions of GHC on Alpine Linux", used to build the static release binaries of Pandoc, Stack and Juvix. Built with Hadrian and the LLVM backend (`perf+split_sections+llvm`).
  - **GMP licence:** regular images link GMP (LGPL), so a *statically linked* binary "must be provided with source code or object files". `int-native` images use GHC's native bignum and avoid that (9.6.7, 9.8.4, 9.10.1, 9.12.2 and later). For 9.10.2/9.12.2+, `int-native` images link with `ld.lld` by default (README).
- **libpq statically:** `postgresql-libpq` 0.11 splits its C configuration into `postgresql-libpq-configure` (default, `pg_config`) and `postgresql-libpq-pkgconfig` (flag `use-pkg-config`) (its `.cabal`). In Alpine 3.22 `main`, `libpq.a` is in **`libpq-dev`** and `libssl.a`/`libcrypto` statics in **`openssl-libs-static`** (pkgs.alpinelinux.org contents search). Which extra `-optl` flags a static link of libpq needs on Alpine (e.g. `-lssl -lcrypto`) is **not stated** by these sources.
- A static binary needs only `gcr.io/distroless/static-debian12:nonroot` (0.72 MB compressed; ca-certificates, a passwd entry, `/tmp`, tzdata) or `scratch` (distroless `base/README.md`; gcr.io manifest).

### Dynamic (glibc) on distroless

- `gcr.io/distroless/base-debian12` (8.2 MB) has glibc and libssl; `cc-debian12` (9.2 MB) adds libgcc1 (distroless READMEs; gcr.io manifests). **Neither contains libpq or libgmp.**
- Debian bookworm `libpq5` (15.19) depends on `libc6`, `libgssapi-krb5-2`, `libldap-2.5-0` and `libssl3` (packages.debian.org). A dynamic binary on distroless would need those shared objects (and their own dependencies) copied from the builder; how many is **not stated** here.

### Size and build time

- **Not stated** by any source read: no primary source gives a final binary size or cold/warm build time for a Servant + persistent service. The base layers above are measured; the binary and the dependency layer would have to be measured on this repo.

## 6. dbmate beside the binary

- **Release v2.37.0** (2026-10-18; releases roughly monthly: 2.35.0 on 2026-08-07, 2.36.0 on 2026-09-19). Assets: `dbmate-linux-{386,amd64,arm,arm64}`, `dbmate-macos-{amd64,arm64}`, `dbmate-windows-amd64.exe`; `linux-amd64` is 34.5 MB. **No checksum file** is attached to the release (release API).
- **Statically linked on Linux:** the Makefile sets `CGO_ENABLED = 1` (for SQLite) and on Linux builds with `-tags netgo,osusergo,…` and `-ldflags '-s -extldflags "-static"'` ("statically link binaries to support alpine linux"); CI runs `make build` and publishes with `softprops/action-gh-release`. The README: "single self-contained binary".
- **Images:** `ghcr.io/amacneil/dbmate` (README) and `amacneil/dbmate:2.37.0` on Docker Hub (34.5 MB amd64). The runtime stage is `alpine:3.24.2` with `postgresql-client`, mariadb client, sqlite and tzdata; `ENTRYPOINT ["/usr/local/bin/dbmate"]` (Dockerfile).
- **Behaviour relevant to the image:** URL from `DATABASE_URL` (or `--url`/`-e VAR`); `--migrations-dir` (default `./db/migrations`), `--migrations-table` (default `schema_migrations`); `--no-dump-schema`; `--wait`/`--wait-timeout`; `--migration-lock` (PostgreSQL only) so concurrent runs wait for each other; `--strict` fails on out-of-order migrations; `dbmate status --exit-code` (README).
- **`pg_dump`:** only `dump` (and the automatic `schema.sql` refresh after `up`/`migrate`/`rollback`) needs `pg_dump` on `PATH`; without it dbmate "will silently skip the schema dump step". The client must be ≥ the server version (README). The README recommends committing `db/schema.sql`.
- **Embedding:** dbmate can be a Go library with `embed` for migrations; there is no Haskell binding (README). So in a Haskell image dbmate is a second executable next to the server, invoked as a process or as a separate step.

## Open points this note did not settle

- Whether the `index-state`/freeze files are bumped by hand, by a scheduled `cabal outdated`/`cabal freeze` job, or by Renovate (which reads only `build-depends`).
- The exact static-link flags for `postgresql-libpq` on Alpine and whether the result needs `int-native` (GMP licensing) — needs a trial build.
- Real numbers for binary size, cold build and cached build on this repo.
- Whether hlint 3.10 (GHC 9.12 parser) and the chosen GHC agree, if the Port picks 9.14 (which `postgresql-simple` currently excludes anyway).

## Sources

| Source | What it gave |
|---|---|
| `raw.githubusercontent.com/haskell/ghcup-metadata/master/ghcup-0.1.0.yaml` | Recommended/Latest tags, release days, `base` versions, Alpine bindist URLs |
| haskell.org/ghcup/guide; `gh api repos/haskell/{ghcup-hs,ghcup-setup}` | Codeberg move, non-interactive install, ghcup-setup action |
| cabal.readthedocs.io (project file, package description, commands) | `packages`, `index-state`, freeze, `outdated`, `executable-static`, common stanzas, sublibraries, `build-tool-depends` |
| `haskell/cabal` `cabal-install/src/Distribution/Client/CmdFreeze.hs` @ `cabal-install-v3.18.2.0` | Freeze file writes `index-state` |
| docs.haskellstack.org lock files; stackage.org `snapshots.json`, `lts-24.65`, `nightly-2026-10-28` (+ `cabal.config`) | Stack lock file; snapshot GHCs and package versions |
| hackage.haskell.org `/preferred` and `.cabal` files | Latest versions, `base` bounds, `postgresql-libpq` flags, hlint/fourmolu parser versions |
| `gh api repos/haskell-actions/{setup,run-fourmolu,run-ormolu,hlint-setup,hlint-run}` (README, `docs/examples.md`, releases, archived flag) | Actions, inputs, caching example, Renovate recipe |
| `gh api repos/{tweag/ormolu,fourmolu/fourmolu,ndmitchell/hlint}` releases | Versions and binary assets |
| docs.github.com Dependabot supported ecosystems | No Haskell ecosystem; `elm`, `nix`, `docker`, `github-actions` |
| docs.renovatebot.com `haskell-cabal` manager, `hackage` datasource | Renovate's Haskell coverage |
| `gh api repos/haskell/security-advisories` (README, tree, branches, HSEC-2026-0011/0012) | Advisory format, count, OSV export, scientific advisories |
| ossf.github.io/osv-schema | `Hackage`, `GHC` ecosystems, `HSEC` prefix |
| `gh api repos/MangoIV/cabal-audit`, `repos/blackheaven/haskell-security-action`; Hackage `cabal-audit` | cabal-audit usage, install, SARIF action |
| `gh api repos/actions/runner-images` `Ubuntu2404-Readme.md` | Preinstalled Haskell tools |
| docs.docker.com build/ci/github-actions/cache | gha cache backend; cache mounts not preserved |
| Docker Hub tags API (`library/haskell`, `benz0li/ghc-musl`, `amacneil/dbmate`); gcr.io v2 manifests (`distroless/*:nonroot`) | Image sizes |
| `gh api repos/benz0li/ghc-musl` README | ghc-musl images, GMP/int-native, linker |
| `gh api repos/GoogleContainerTools/distroless` `base/README.md`, `cc/README.md` | Distroless contents |
| pkgs.alpinelinux.org contents (v3.22, x86_64) for `libpq.a`, `libssl.a` | Alpine static libs |
| packages.debian.org/bookworm/libpq5 | libpq5 dependencies |
| `gh api repos/amacneil/dbmate` (README, Makefile, Dockerfile, `.github/workflows/ci.yml`, releases) | dbmate assets, static build, image, flags, `pg_dump` |
