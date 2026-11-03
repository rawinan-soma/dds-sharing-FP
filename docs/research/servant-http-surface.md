# Servant libraries for sessions, CSRF and contract checks

- **Question:** Which Haskell libraries cover the Port's HTTP surface under Servant? Our own cookie sessions (custom `AuthHandler` against `servant-auth-server`), a CSRF check, the client IP behind a proxy, argon2id verification compatible with node-argon2 hashes and settings, TOTP for the maths only, an in-memory keyed lock, checking every response against the existing OpenAPI 3.1 file and counting operation coverage, and in-process tests on throwaway Postgres databases.
- **Issue:** rawinan-soma/dds-sharing-FP#38 (map #35; blocks #43, #46). The Rust answers it reopens are on #10 (Reviewer sign-in) and #15 (how the Port proves it keeps the rules).
- **Date:** 2026-11-03
- **Primary sources:** Hackage package pages, metadata-revision pages and the JSON `preferred` endpoint (latest versions, upload dates, uploaders). The source tarballs from `hackage.haskell.org/package/<name>-<ver>/<name>-<ver>.tar.gz`, read directly. Citations look like `openapi-hs 5.0.0: src/Data/OpenApi/Internal.hs:126` and refer to that tarball. The GitHub REST API for stars, last push and archive status. The PostgreSQL manual. This repo at `432d1bd`, and node-argon2 0.45.1 as installed in the Pilot's `node_modules`. ⚠️ marks anything I read but did not run. No GHC toolchain was available, so **no Haskell code was compiled or run** for this note.

This note reports facts and makes no decision. #43 and #46 make the decisions.

## Answer in brief

- **Sessions.** `servant-auth-server` 0.4.10.0 cookie auth puts a **signed JWT** in the cookie. The Port's session is an opaque token whose SHA-256 is a row in `reviewer_session`, so that model does not fit. servant-server's `AuthProtect` / `AuthHandler` (`Servant.Server.Experimental.Auth`, still labelled experimental) takes the raw WAI `Request` and returns whatever type you choose.
- **CSRF.** `servant-auth-server` checks XSRF only inside its cookie authentication. That check cannot cover an unauthenticated sign-in, and D5 requires CSRF there. `wai-csrf` 0.1 is a small WAI middleware whose token format (32 random bytes, base64url without padding) is the same as the Pilot's. It has one release, 0 GitHub stars and one reverse dependency.
- **Client IP.** wai-extra 3.1.18 `Network.Wai.Middleware.RealIp` (`realIpTrusted`) rewrites `remoteHost` from `X-Forwarded-For`, but only when the immediate peer is trusted. Servant's `RemoteHost` combinator hands the result to a handler. iproute has no function that unmaps an IPv4-mapped IPv6 address.
- **argon2.** node-argon2 0.45.1 writes `$argon2id$v=19$m=65536,p=4,t=3$<16-byte salt>$<32-byte hash>`. The parameter order is **m,p,t**. The `argon2` package's `verifyEncoded` uses the reference C decoder, which requires **m,t,p**. Per its source, it rejects Pilot hashes with a decoding failure (⚠️ not run). `password` 3.1.1.0 (`Data.Password.Argon2.checkPassword`) parses the three parameters in any order and recomputes with crypton. It caps crypton at `< 1.2`, and crypton is now at 2.2.6.
- **TOTP.** crypton 2.2.6 `Crypto.OTP` (`totp`, `totpVerify`, `mkTOTPParams`) accepts any key bytes. `one-time-password` 3.0.1.0 types its SHA-1 key as a libsodium 32-byte HMAC-SHA-256 key, which existing 20-byte secrets cannot be loaded into as they are.
- **Keyed lock.** No maintained dedicated package turned up. The parts are in `base` (`MVar`) and `stm` / `stm-containers` 1.2.2.
- **Contract (OpenAPI 3.1).** `openapi3` 3.2.5, and therefore `servant-openapi3` 2.1.0.0, **refuse to parse `openapi: 3.1.0`**: they accept 3.0.0 to 3.0.3. `openapi-hs` 5.0.0 is a 2026 fork that models 3.1 / JSON Schema 2020-12 and validates a JSON `Value` against a `Schema`. It has 2 GitHub stars and one maintainer, and it needs GHC 9.12 or 9.14. `jsonschema` 0.3.0.1 (DPella) is a standalone 2020-12 validator over plain aeson `Value`s.
- **Tests.** wai-extra's `Network.Wai.Test` and `hspec-wai` 0.12.1 run the `Application` in-process and expose status, headers and body. `tmp-postgres` was last released in 2019 (tested with GHC 8.6.5). PostgreSQL's `CREATE DATABASE … TEMPLATE` gives a per-test database on an existing server, which is what #15's Rust answer did.

## What the current service relies on

Read from `apps/api/src` at `432d1bd`. These are what a replacement has to reproduce.

- **Password hashing.** `hashPassword` is `argon2.hash(password, { type: argon2.argon2id })`, with no other options (`reviewer/password-policy.ts:61`). `package.json` asks for `argon2 ^0.45.1`. node-argon2 0.45.1's `defaults` are `hashLength: 32, memoryCost: 1 << 16, parallelism: 4, timeCost: 3, type: argon2id, version: 0x13`, and the salt is `generateSalt(16)` (`argon2.cjs:29-36`, `:89`). It serialises the parameters as `{ m, p, t }` (`argon2.cjs`, in `hash`). A hash generated with this install:
  ```
  $argon2id$v=19$m=65536,p=4,t=3$B9FhhCjkHN8Dz4LQyTb7xQ$fWoLj222VKx7rNWKqErB3nudow8bVW8eUAhMFxPNchA
  ```
  The salt and hash are standard-alphabet base64 without padding: 22 characters for 16 bytes, 43 for 32. `verifyPassword` turns any thrown error into `false` (`password-policy.ts:64-72`). The dummy hash for unknown accounts is a real `hashPassword('Dummy-Password-1!')`, computed once (`reviewer-auth.ts:46-47`).
- **TOTP.** Uses `otpauth ^9.5.2`. SHA-1, 6 digits and a 30 s step are fixed on purpose: Google Authenticator ignores the URI's `algorithm`/`digits`. The secret is `new Secret({ size: 20 })`, 160 bits in base32 (`totp.ts:24-31`). Verification checks steps −2…+1 with no short-circuit, compares in constant time, refuses replays at or below `lastUsedStep`, accepts within ±1, and reports −2 as drift (`totp.ts:55-85`).
- **Session cookie.** `reviewer_session` holds an opaque `randomBytes(32).toString('base64url')`. The database stores only its SHA-256 hex (`reviewer-sessions.ts:28-50`). Idle timeout 1 h, ceiling 6 h, at most 3 sessions with the oldest evicted (`reviewer-sessions.ts:8-13`). Cookie attributes are `Path=/`, `SameSite=Lax`, `HttpOnly`, and `Secure` when `secureCookies` (`cookies.ts:27-45`).
- **CSRF.** Double-submit. The `reviewer_csrf` cookie is `randomBytes(32).toString('base64url')` and not `HttpOnly` (`cookies.ts:49-53`). Every method other than GET/HEAD/OPTIONS must send header `x-csrf-token` equal to the cookie, compared in constant time, or get `403 {"error":"csrf"}` (`csrf.guard.ts`, `cookies.ts:55-59`).
- **One sign-in at a time per account.** A per-key promise chain in an in-process `Map`. It relies on the app being one process (`reviewer-auth.ts:53-70`).
- **Client IP.** Express `trust proxy` comes from config (`main.ts:23`, `config/env.schema.ts:111`). Some routes strip `::ffff:` (`delivery/delivery.controller.ts:7-9`). Ruling D6 (on #10) makes every route record the canonical address.
- **The contract.** `docs/http-api.openapi.yaml` declares `openapi: 3.1.0`. It has 22 paths and 23 operations, and **no `operationId`s**, so coverage has to be counted by method + path. Its schemas use 2020-12/3.1-only forms: `const` (27 occurrences) and type arrays such as `type: [string, 'null']` (8). It also uses `oneOf` 11 times, `allOf` 3, `enum` 24, `additionalProperties` 1, and `format` `date`/`date-time`/`uuid`. There is no `pattern` and no `nullable`. `$ref`s point at `#/components/schemas/`, `#/components/responses/` and `#/components/parameters/`. Responses declare headers (`Location`, `Content-Range` and others) in 12 places, and some bodies are `text/html` or `application/zip`.
- **Test databases.** `apps/api/test/support/scratch-database.ts` runs `CREATE DATABASE` per suite on an existing server, migrates under `pg_advisory_lock`, and connects as a login that is only a member of `dds_app`.

## Versions and maintenance

Source: Hackage package pages and the `preferred` JSON endpoint; GitHub REST API, read 2026-11-03.

| package | latest | first uploaded (latest) | last revision | repo: stars / last push |
|---|---|---|---|---|
| `servant`, `servant-server`, `servant-client` | 0.20.3.0 | 2025-06-04 (hecate) | r7 / r5 / r4, 2026-08-21/22 | haskell-servant/servant: 1,968 / 2026-10-30 |
| `servant-auth-server` | 0.4.10.0 | 2026-06-20 (janus) | r1 2026-08-21 | same repo |
| `servant-openapi3` | 2.1.0.0 | 2026-08-21 (janus) | r1 2026-08-22 | now in the servant repo (changelog 2.1.0.0) |
| `openapi3` | 3.2.5 | 2026-04-22 (maksbotan) | — | biocad/openapi3: 49 / 2026-04-22, 67 open issues |
| `openapi-hs` | 5.0.0 | 2026-07-21 (shinzui) | — | shinzui/openapi-hs: 2 / 2026-10-22 |
| `servant-openapi-hs` | 5.1.0 | 2026-07-22 (shinzui) | — | — |
| `jsonschema` (DPella) | 0.3.0.1 | 2026-07-09 (tritlo) | — | dpella/jsonschema: 3 / 2026-07-09; MPL-2.0 |
| `wai` | 3.2.5 | 2026-08-07 | — | — |
| `wai-extra` | 3.1.18 | 2025-09-13 | — | — |
| `warp` | 3.4.20 | 2026-11-03 | — | — |
| `cookie` | 0.5.1 | 2025-03-31 | — | — |
| `wai-csrf` | 0.1 | 2025-09-25 (RenzoCarbonara) | — | k0001/hs-wai-csrf: 0 / 2026-05-09; Apache-2.0 |
| `argon2` | 1.3.0.1 | 2018-04-07 (hvr) | r4 2023-04-02 | haskell-hvr/argon2: 18 / 2024-06-24 |
| `crypton` | 2.2.6 | 2026-11-03 (KazuYamamoto) | — | kazu-yamamoto/crypton: 51 / 2026-11-03 |
| `password` | 3.1.1.0 | 2026-05-03 (nideco) | none | cdepillabout/password: 61 / 2026-07-02 |
| `one-time-password` | 3.0.1.0 | 2025-05-20 (hecate) | — | haskell-cryptography/one-time-password: 7 / 2026-09-29 |
| `stm-containers` | 1.2.2 | 2026-01-04 | — | — |
| `hspec-wai` | 0.12.1 | 2026-07-15 | — | hspec/hspec-wai: 68 / 2026-07-15 |
| `tmp-postgres` | 1.34.1.0 | 2019-12-29 | r1 2024-05-06 | jfischoff/tmp-postgres: 57 / 2024-08-16, 43 open issues |
| `ephemeral-pg` | 0.3.1.0 | 2026-09-16 (shinzui) | — | shinzui/ephemeral-pg: 0 / 2026-09-16 |
| `tmp-proc-postgres` | 0.8.0.0 | 2026-10-18 | — | — |
| `testcontainers` | 0.5.4.0 | 2026-07-06 | — | — |

`cryptonite` and `hjsonschema` are marked **deprecated** on Hackage. servant-server 0.20.3.0 lists `tested-with: GHC ==9.2.8 … ==9.12.1` (`servant-server.cabal:29`).

## Sessions: `servant-auth-server` against a custom `AuthHandler`

### servant-auth-server 0.4.10.0

- `cookieAuthCheck` reads the cookie named `sessionCookieName` and passes its value to `verifyJWT` (`src/Servant/Auth/Server/Internal/Cookie.hs`, `cookieAuthCheck`). `makeSessionCookie` builds the cookie value with `makeJWT` (same file). So **the session cookie is a signed JWT carrying the user value**. It is not an opaque key into a table.
- `defaultCookieSettings`: `sessionCookieName = "JWT-Cookie"`, `cookieSameSite = SameSiteLax`, `cookieIsSecure = Secure`, `cookieXsrfSetting = Just def` (`Internal/ConfigTypes.hs:88-95`). These are fields and can be changed.
- The library has no server-side session store. Idle timeout, ceiling, the cap of three, eviction and deactivation are not concepts in it (no such fields in `ConfigTypes.hs`).

### servant-server's generalised auth

- `type family AuthServerData a`; `newtype AuthHandler r usr`; `mkAuthHandler :: (r -> Handler usr) -> AuthHandler r usr`. The `HasServer (AuthProtect tag :> api)` instance takes an `AuthHandler Request (AuthServerData (AuthProtect tag))` from the `Context` and runs it per request (`servant-server 0.20.3.0: src/Servant/Server/Experimental/Auth.hs:43-72`).
- Every declaration in the module carries `NOTE: THIS API IS EXPERIMENTAL AND SUBJECT TO CHANGE`. The module has kept that label through to 0.20.3.0.
- The handler gets the whole WAI `Request`, including the `Cookie` header and `remoteHost`, runs in `Handler`, and can `throwError` a 401/403 with any body. The cookie can be parsed with `cookie` 0.5.1 `Web.Cookie.parseCookies`, and set with `SetCookie` (`sameSiteLax`, `setCookieHttpOnly`, …) through Servant's `Header "Set-Cookie" SetCookie` / `addHeader`.

## CSRF

- **servant-auth-server's XSRF.** It runs inside `cookieAuthCheck`: it compares cookie `xsrfCookieName` with header `xsrfHeaderName` using `constEq`, and skips GET only when `xsrfExcludeGet` is set (`Internal/Cookie.hs`, `xsrfCheckRequired` and `xsrfCookieAuthCheck`). The defaults are `XSRF-TOKEN` / `X-XSRF-TOKEN`, `xsrfExcludeGet = False` (`ConfigTypes.hs:117-120`). Because the check sits inside cookie authentication, a route without that auth gets no check, and sign-in is such a route. `makeXsrfCookie` writes `BS64.encode <$> getEntropy 32`, which is standard padded base64, not base64url (`Internal/Cookie.hs`).
- **wai-csrf 0.1** (`lib/Wai/CSRF.hs`):
  - `Token` is exactly 32 bytes. Its `Eq` is `constEq`, and it is encoded `Base64URLUnpadded` (`:39-70`). That is the Pilot's `randomBytes(32).toString('base64url')` format.
  - `Config { cookieName, headerName, reject }`. `defaultConfig` uses `CSRF-TOKEN` / `X-CSRF-TOKEN`. It lets GET, HEAD, OPTIONS and TRACE through and answers anything else with `403` and body `"CSRF"` unless the cookie and header tokens are equal (`:130-179`). `reject :: Request -> Maybe (Token, Bool) -> Maybe Response` can be replaced, for example to return the contract's `{"error":"csrf"}` body.
  - `middleware :: Config -> (Maybe Token -> Application) -> Application`. It sets no cookie. `setCookie` returns a `SetCookie` with `HttpOnly` off, `Path=/`, `SameSite=Lax`, `Secure` on and no expiry, and it is a plain record you can change (`:199-262`).
  - One release, one reverse dependency (`wai-cryptocookie`), 56 downloads (Hackage page).
- The Pilot's own check is about 15 lines (`csrf.guard.ts` + `csrfTokensMatch`). In Haskell the same check fits in an `AuthHandler` or a WAI `Middleware` using `Web.Cookie.parseCookies` and a constant-time compare: `Data.ByteArray.constEq` from `memory`/`ram`, or crypton's re-export.

## Client IP behind a proxy

- **wai-extra 3.1.18 `Network.Wai.Middleware.RealIp`** (`@since 3.1.5`):
  - `realIpTrusted :: HeaderName -> (IP -> Bool) -> Middleware`. It reads the header only if the socket peer (`remoteHost`) passes the trust predicate.
  - It takes the **last non-trusted** address in the header, or the first address if every address is trusted. It ignores invalid entries, handles repeated headers, and writes the result back into `remoteHost` (`Network/Wai/Middleware/RealIp.hs:34-97`).
  - `realIp` = `realIpHeader "X-Forwarded-For"` trusting `defaultTrusted`: 127/8, 10/8, 172.16/12, 192.168/16, ::1/128 and fc00::/7 (`:61-72`).
  - `ipInRange` returns `False` for an IPv6 address tested against an IPv4 range (`:74-84`). An IPv4-mapped peer such as `::ffff:10.0.0.1` therefore does not match `10.0.0.0/8`.
- **Servant.** The `RemoteHost` combinator passes the WAI `remoteHost` (`SockAddr`) to the handler (`servant-server 0.20.3.0: src/Servant/Server/Internal.hs:883`).
- **Canonicalisation.**
  - iproute 1.7.15's `fromSockAddr` maps `SockAddrInet6` to `IPv6` without unmapping (`Data/IP/Addr.hs:684-687`).
  - `show` prints a mapped address as `::ffff:a.b.c.d` (`:233-237`).
  - There is an `ipv4ToIPv6` (`:676`) but no inverse. Unmapping (top 96 bits `0…0ffff`) is a few lines of your own code.

## argon2id compatible with node-argon2

### `argon2` 1.3.0.1 (hvr)

- It bundles the reference `phc-winner-argon2` C (release 20171227, `Changelog.md`). `verifyEncoded :: ShortText -> ByteString -> Argon2Status` checks the `$argon2id$` prefix and calls the C `argon2_verify` (`src/Crypto/Argon2.hs:172-184`).
- The C decoder `decode_string` reads exactly `$m=`, then `,t=`, then `,p=`, and fails with `ARGON2_DECODING_FAIL` on any other sequence (`phc-winner-argon2/src/encoding.c:333-338`).
- So **a node-argon2 0.45.1 hash (`m=…,p=…,t=…`) decodes as a failure, not a mismatch**. Re-ordering the PHC string to `m,t,p` before calling `verifyEncoded` is enough to decode it. ⚠️ This is from reading the source. Compiling the bundled C to confirm was not permitted in this session.
- `hashEncoded` writes `m,t,p`. The raw `hash` can be called with explicit `HashOptions` (`hashIterations`, `hashMemory`, `hashParallelism`, `hashVariant`, `hashVersion`, `hashLength`). `defaultHashOptions` is argon2i, t=3, m=4 MiB, p=1, so it is not node-argon2's setting (`:136-145`).
- The FFI hash calls are `ccall safe` (`src/Crypto/Argon2/FFI.hsc:53,83`). Last upload 2018, last metadata revision 2023.

### `password` 3.1.1.0 (`Data.Password.Argon2`, backend crypton)

- `checkPassword` splits the PHC string on `$`. `parseParameters` requires exactly three comma-separated items and accepts `m=`, `t=`, `p=` **in any order**. It takes the output length from the decoded hash and the salt from the decoded salt, re-adds padding before decoding, recomputes the hash and compares with `constEq` (`src/Data/Password/Argon2.hs:290-345`). That parses a node-argon2 hash. ⚠️ Not run.
- `defaultParams` is argon2id, m=65536, **t=2, p=1**, 16-byte salt and 32-byte output (`:181-194`). `hashPasswordWithParams` with t=3, p=4 matches node-argon2. Its output order is `m=,t=,p=` (`:226-228`).
- **Dependency bound.** The default build needs `crypton >= 1.1 && < 1.2` with `ram`. The `memory-dep` flag selects `crypton >= 0.31 && < 1.1` with `memory` (`password.cabal:130-147`). The Hackage revisions page shows no revision lifting this bound. crypton's current major is **2.x** (2.0.0 onward), so `password` 3.1.1.0 cannot share a build plan with any other package that needs crypton ≥ 2.
- 3.1.0.0 switched the default backend from cryptonite to crypton (`ChangeLog.md`).

### crypton 2.2.6 `Crypto.KDF.Argon2`

- `Options { iterations, memory, parallelism, variant, version }` and `hash` (`Crypto/KDF/Argon2.hs:75-82`). Raw only: crypton has no PHC string parser or encoder, so a caller parses the PHC string and base64 itself.
- The C call is `foreign import ccall unsafe "crypton_argon2_hash"` (`:171`). The bundled C uses pthreads unless `ARGON2_NO_THREADS` is defined (`cbits/argon2/thread.c:18`).
- 2.0.0 changed refused Argon2 parameters to come back as `CryptoError`, where they used to throw `ErrorCall` (`CHANGELOG.md`, 2.0.0).

## TOTP (maths only)

- **crypton 2.2.6 `Crypto.OTP`**:
  - It exports `hotp`, `totp`, `totpVerify`, `TOTPParams`, `defaultTOTPParams :: TOTPParams SHA1`, `mkTOTPParams`, `ClockSkew`, `OTPDigits (OTP4 … OTP9)` and `type OTP = Word32` (`Crypto/OTP.hs:29-70, 185-260`).
  - Keys are any `ByteArrayAccess`, so a 20-byte secret decoded from base32 works as it is. Base32 decoding is not in `Crypto.OTP`; `memory`/`ram` `Data.ByteArray.Encoding` has `Base32`.
  - 2.0.0 made TOTP candidates compare "without an early exit", and HOTP refuses digests too short for dynamic truncation (`CHANGELOG.md`, 2.0.0).
- **one-time-password 3.0.1.0**:
  - Moved to the Haskell Cryptography Group in 3.0.0.0, which also switched it to `sel` (libsodium) (`CHANGELOG.md`). It depends on `sel ^>=0.1`, `cryptohash-sha1` and `chronos` (`one-time-password.cabal:52-62`).
  - `hotpSHA1` takes a `Sel.HMAC.SHA256.AuthenticationKey`, gets its bytes and runs HMAC-SHA-1 over them (`src/OTP/HOTP.hs:45-60`). `sel` 0.1.0.0 builds such a key only from exactly `cryptoAuthHMACSHA256KeyBytes` (32) bytes, from hex (`src/Sel/HMAC/SHA256.hs:291-304`). So **a 20-byte secret cannot be loaded as it is**.
  - HMAC zero-pads a short key to the block size, so the 20-byte secret followed by 12 zero bytes gives the same HMAC-SHA-1. That follows from RFC 2104, and I checked it with Python's `hmac` on a random 20-byte key. Whether the library's types allow it cleanly is ⚠️ unverified.
  - `totpSHA1Check` compares with `elem` on the rendered codes, not in constant time (`src/OTP/TOTP.hs:163-166`).

## In-memory keyed lock

- Hackage names probed: `named-lock` (last upload 2009), `keyed-lock`, `keyed-locks`, `lock-map` (do not exist). None is a maintained keyed lock.
- The parts:
  - `base` `MVar ()` per key held in an `MVar (Map key …)` or `IORef`, with `bracket` / `modifyMVar` for exception safety.
  - Or `stm-containers` 1.2.2 `StmContainers.Map` of `TMVar ()`. It was uploaded 2026-01-04 and is MIT.
  - In both cases, removing a key's entry when no one is waiting is the caller's code, as it is in the Pilot (`reviewer-auth.ts:62-70`).

## Checking responses against the OpenAPI 3.1 file

### `openapi3` 3.2.5 / `servant-openapi3` 2.1.0.0: 3.0 only

- `lowerOpenApiSpecVersion = 3.0.0`, `upperOpenApiSpecVersion = 3.0.3`. `FromJSON OpenApiSpecVersion` `fail`s outside that range with "The provided version … is out of the allowed range" (`openapi3 3.2.5: src/Data/OpenApi/Internal.hs:96-101, 1444-1453`). **Decoding `docs/http-api.openapi.yaml` (`openapi: 3.1.0`) fails at the top-level field.**
- The 3.0 model has no `const` and represents `nullable` rather than type arrays (`openapi-hs` 4.0.0 changelog, listing what it changed from openapi3).
- `servant-openapi3` 2.1.0.0 depends on `openapi3 >=3.2.5 && <3.3` (`servant-openapi3.cabal:79`). Its tests (`Servant.OpenApi.Test`, e.g. `validateEveryToJSON`) check handlers' `ToJSON` output against schemas **generated from the Haskell types**, not against a hand-written file. Neither package diffs a generated spec against an existing one.

### `openapi-hs` 5.0.0: a 3.1 fork of openapi3

- Changelog 4.0.0: "the data model now represents OpenAPI 3.1 / JSON Schema 2020-12 instead of 3.0. Supported spec versions are 3.1.x." It removes `nullable` in favour of type arrays. It adds `const`, `prefixItems`, `if/then/else`, `contains*`, `unevaluated*`, `dependent*`, `$defs`/`$ref`/`$dynamicRef` and webhooks. It renames openapi3 to openapi-hs and keeps the `Data.OpenApi.*` namespace. 4.1.0 adds `1XX`…`5XX` range keys. 5.0.0 drops the optics modules.
- Accepted versions: `3.1.0` to `3.1.1` inclusive (`src/Data/OpenApi/Internal.hs:125-130, 1700-1708`). `openapi: 3.1.0` is in range.
- **Validation API:**
  - `validateJSON :: Definitions Schema -> Schema -> Value -> [ValidationError]`, plus `validateJSONWithPatternChecker`. The default checker accepts every `pattern`, which does not matter here because the contract has none (`src/Data/OpenApi/Internal/Schema/Validation.hs:163-170`).
  - It checks type, enum, `const`, `if/then/else`, `oneOf` (exactly one), `allOf`, `additionalProperties`, and a best-effort `unevaluated*` (`:304-342, 529-545, 649-656`). It does not assert `format`.
  - A schema `$ref` is looked up by name in the `Definitions` you pass, normally `components.schemas` (`:293-301`). `#/components/responses/…` and `#/components/parameters/…` refs are separate `Referenced` values that the calling code resolves itself.
- `servant-openapi-hs` 5.1.0 is the matching fork of servant-openapi3 and needs `base >=4.21 && <4.23`.
- Package facts: tested with GHC 9.12.4 and 9.14.1 only (Hackage page). One maintainer (shinzui). 2 GitHub stars. 4 direct reverse dependencies, all of them shinzui's or small.
- ⚠️ Whether `openapi-hs` decodes the whole contract (flow-style YAML through the `yaml` package, the `in: cookie` parameter, `text/html: {}` bodies) was **not run**.

### `jsonschema` 0.3.0.1 (DPella): a standalone 2020-12 validator

- `validate :: Value -> Value -> Either [ValidationError] ()` and `validateJSONSchema :: Value -> Value -> Bool`. The README describes "the core 2020-12 validation + applicator vocabularies", with `format` and `content*` as annotations only.
- Local `$ref` is resolved as a JSON Pointer against the root schema (`src/JSONSchema/Validation.hs:68-111, 683-690`). The public `validate` uses the schema itself as the root, so a `#/components/schemas/X` ref resolves only if the schema passed in is a document that contains `components`.
- A `$ref` **short-circuits its sibling keywords** ("if present, ignore other keywords", `:104-105`). That is draft-7 behaviour; 2020-12 evaluates siblings.
- `base >=4.18 && <4.22`, which is GHC 9.6 to 9.12. MPL-2.0.

### Coverage and the Rust precedent

- #15's Rust answer: a helper wraps the router the tests call and checks status, required headers and cookies, and body schema against the matching operation (`oas3` + `jsonschema` 2020-12). A coverage test then fails on any operation or listed response that was never exercised.
- Nothing on Hackage provides that harness or the coverage count. With either validator above it is test-side code. Coverage would be keyed on method + path template, because the contract has no `operationId`s.

## In-process tests and throwaway databases

- **wai-extra `Network.Wai.Test`.** It runs an `Application` in-process (`runSession`, `request`, `srequest`) and returns `SResponse { simpleStatus, simpleHeaders, simpleBody }` (`Network/Wai/Test.hs:97-196`). It keeps a client cookie jar (`getClientCookies`, `setClientCookie`, `assertClientCookieValue`, `:10-35, 71`). A test can set `remoteHost` on the `Request` it builds.
- **hspec-wai 0.12.1.**
  - Wraps that in `with :: IO Application -> SpecWith ((), Application) -> Spec`, `get`/`post`/`request`, and `shouldRespondWith` with `ResponseMatcher` (`src/Test/Hspec/Wai.hs:74-198`).
  - Its `request` always builds from `defaultRequest` with method, headers and path only (`:187-190`). A non-default `remoteHost` therefore needs `Network.Wai.Test` directly.
- **servant-client 0.20.3.0** over warp's `testWithApplication` is the over-a-real-socket alternative. It needs a Servant API type to derive the client from.
- **Throwaway Postgres:**
  - `tmp-postgres` 1.34.1.0 starts its own `initdb`/`postgres` and `createdb`. Hackage lists it as "Tested with ghc ==8.6.5". Its last upload was 2019, with a metadata revision in 2024 and 43 open issues.
  - `ephemeral-pg` 0.3.1.0 (2026, 0 stars) is built on `hasql`, not `postgresql-simple` or `persistent`.
  - `tmp-proc-postgres` and `testcontainers` start Postgres in Docker.
  - With an existing server, the PostgreSQL manual says any database can be named as the template for `CREATE DATABASE`. "The principal limitation is that no other sessions can be connected to the source database while it is being copied" ([Template Databases](https://www.postgresql.org/docs/current/manage-ag-templatedbs.html)).
  - `WAL_LOG` is the default strategy and suits small templates. `FILE_COPY` forces a checkpoint before and after ([CREATE DATABASE](https://www.postgresql.org/docs/current/sql-createdatabase.html)).

## Not examined

- External OpenAPI 3.1 response validators outside Haskell, called from the test suite. `openapi-hs`'s changelog mentions `vacuum` only as a linter for documents.
- `servant-quickcheck` (r5 2026-08-22). It property-tests servant APIs against general predicates and does not read an OpenAPI file.
- How long an argon2id hash with m=64 MiB, t=3, p=4 takes in either Haskell binding, and whether crypton's `unsafe` FFI call holding a capability for that long matters under the threaded RTS. Runtime tuning is listed on #35 as not yet specified.
