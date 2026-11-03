# Haskell libraries for extraction, pages and mail

- **Question:** Which Haskell libraries cover extraction, delivery and mail? The upstream HTTP client and typed JSON decoding that ignores dropped fields; CSV writing with full byte control; SHA-256; deflate-only zip; S3 against Garage; SMTP with login over a relay; server-side HTML and email templates with compile-time checking; and Template Haskell that reads the copy catalogue and the theme's hex literals to generate one typed function per key.
- **Issue:** rawinan-soma/dds-sharing-FP#39 (child of map #35). The Rust answers it replaces are #11 (extraction pipeline), #12 (delivery page outside the SPA) and #8 (side services).
- **Date:** 2026-11-03
- **Primary sources:** the Hackage API (`/package/<p>/preferred`, `/upload-time`, `/deprecated`) and each package's `.cabal` file, changelog and source as published on Hackage; the Stackage LTS 24.65 (GHC 9.10.3) and nightly-2026-10-28 (GHC 9.12.4) package pages; the GitHub API for repo status; Garage's S3 compatibility page and its source at tag `v2.5.0`; Google's `double-conversion` source; the Cabal source; and this repo's Node code at `432d1bd`. Every claim below names the source it came from. Anything a source did not say is marked **not stated**.

This note lists facts. It recommends nothing. The grilling tickets on #35 make the decisions.

## What the current service and earlier answers pin

Read from `apps/api/src` at `432d1bd` and from the closed tickets.

- **The Extract writer** (`extraction/extract-writer.ts`): prepends the UTF-8 BOM, ends every row with CRLF (the last row too), uppercases the named columns with JS `toUpperCase()`, then `trim()`s every cell, and quotes a cell only if it matches `/[",\r\n]/`, doubling `"`. The SHA-256 is taken over the same bytes as they are emitted. Numbers become text through JS `String(value)` (`project.ts:93`, `filter.ts:15`). #11 pins this byte-for-byte to the Pilot under a golden fixture (D7), with D8 as the one exception.
- **The archive** (`extraction/extract-archive.ts`): `yazl`, two entries (the Extract and the Data dictionary), built in memory into one `Buffer`. The comment says the archive "is deliberately never fingerprinted". #11 chose deflate for both entries.
- **The store** (`extraction/archive-store.ts`, `bucket-lifecycle.ts`): `putObject` (one call), `statObject`, `getObject` streamed through the app, and a backstop lifecycle rule `{ Filter: { Prefix: '' }, Expiration: { Days: 3 } }`. **No presigned URLs.** #8 says the same for the Port: "get (streamed through the app; no presigned URLs)". The ticket text asks about presigned GET; it is covered below as a capability only.
- **The upstream client** (`upstream/upstream-client.ts`): `fetch` with `AbortSignal.timeout(60_000)` per attempt, its own retry loop and call counting.
- **Copy interpolation** (`i18n/copy-catalogue.ts`): `template.replace(/\{(\w+)\}/g, …)`; a placeholder with no matching param is left as is.
- **Mail** in the Pilot is `nodemailer`. #8: STARTTLS on submission to `mailrelay.uc-workd.com` with a login; Mailpit with plaintext in dev and CI.

### The copy and theme paths

- `messages/messages.th.json` **does not exist yet.** At `432d1bd` the catalogue is `messages/th.json` (with a `$schema` key) and `messages/en.json`, and `project.inlang/settings.json` has `pathPattern: ./messages/{locale}.json`. #13's answer renames them to `messages/messages.th.json` / `messages.en.json` (no `$schema`) in "the first slice that builds the Elm app", and amends #12 so the code generator reads `messages/messages.th.json` directly. `docs/port-build-plan.md` (step 5) uses the new path.
- `messages/th.json` today holds 415 keys. 50 of them carry placeholders, drawn from 19 names (`attempts`, `cap`, `changes`, `count`, `days`, `field`, `hours`, `met`, `minutes`, `name`, `province`, `reference`, `region`, `reviewer`, `seconds`, `telephone`, `time`, `used`, `zone`). Every key matches `^[a-z][a-z0-9_]*$`, which is a valid Haskell variable name. No value contains `&`, `<` or `>`. (Counted with a script over the file.)
- `theme/theme.css` **does not exist on `main`.** #14's answer names one `theme.css`, starting from `apps/web/prototype-daisyui-theme/theme.css` on branch `prototype/daisyui-theme`. In that file the colours are `--color-*: #rrggbb;` lines (74 six-digit hex literals). Shadows use `rgb(… / …)`, which is not hex.

## Maintenance baseline

From the Hackage API (latest version, date of the latest upload), Stackage, and the GitHub API (archived flag, last push). "LTS" means LTS 24.65 (GHC 9.10.3); "nightly" means nightly-2026-10-28 (GHC 9.12.4).

| package | latest | uploaded | in LTS / nightly | repo |
|---|---|---|---|---|
| `http-client` | 0.7.19 | 2025-03-21 | — | snoyberg/http-client, pushed 2026-06-10 |
| `http-client-tls` | 0.4.0 | 2026-03-11 | — | same repo |
| `req` | 3.13.4 | 2026-06-12 | 3.13.4 / 3.13.4 | mrkkrp/req, pushed 2026-09-29 |
| `aeson` | 2.3.2.0 | 2026-09-17 | — | haskell/aeson, pushed 2026-11-03 |
| `cassava` | 0.5.4.1 | 2026-03-02 | 0.5.4.1 / 0.5.4.1 | haskell-hvr/cassava, pushed 2026-03-02 |
| `crypton` | 2.2.6 | 2026-11-03 | 1.0.6 / 1.1.5 | kazu-yamamoto/crypton, pushed 2026-11-03 |
| `cryptohash-sha256` | 0.11.102.1 | 2025-01-01 | — | — |
| `zip` | 2.2.2 | 2026-06-14 | 2.2.2 / 2.2.2 | mrkkrp/zip, pushed 2026-09-29 |
| `zip-archive` | 0.5 | 2026-09-21 | 0.4.3.2 / 0.4.3.2 | jgm/zip-archive, pushed 2026-11-02 |
| `amazonka`, `amazonka-s3`, `amazonka-core` | 2.0 | 2024-05-13 | absent / absent | brendanhay/amazonka, pushed 2026-10-25 |
| `minio-hs` | 1.7.0 | 2023-05-22 | absent / absent | minio/minio-hs, pushed 2025-05-13 |
| `aws` | 0.25.3 | 2026-06-24 | — | haskell-pkg-janitors/aws |
| `smtp-mail` | 0.5.0.1 | 2026-10-12 | 0.5.0.0 / 0.5.0.1 | haskell-github-trust/smtp-mail, pushed 2026-11-02 |
| `HaskellNet` | 0.6.2 | 2025-06-08 | — | qnikst/HaskellNet, pushed 2026-08-08 |
| `HaskellNet-SSL` | 0.4.0.2 | 2025-02-15 | 0.4.0.2 / 0.4.0.2 | dpwright/HaskellNet-SSL, pushed 2026-05-02 |
| `mime-mail` | 0.5.2 | 2026-04-13 | 0.5.1 / 0.5.2 | snoyberg/mime-mail, pushed 2026-10-08 |
| `lucid2` | 0.0.20260427 | 2026-05-10 | yes / yes | see below |
| `blaze-html` | 0.9.2.0 | 2024-07-04 | 0.9.2.0 / 0.9.2.0 | jaspervdj/blaze-html, pushed 2026-09-28 |
| `shakespeare` | 2.2.0 | 2025-11-04 | 2.1.7.1 / 2.2.0 | yesodweb/shakespeare, pushed 2025-11-04 |
| `heist` | 1.1.1.2 | 2026-08-29 | 1.1.1.2 / 1.1.1.2 | snapframework/heist, pushed 2024-07-04 |
| `ede` | 0.3.4.0 | 2024-12-05 | 0.3.4.0 / 0.3.4.0 | brendanhay/ede, pushed 2024-12-05 |
| `stache` | 2.4.0 | 2026-06-17 | 2.3.4 / 2.3.4 | mrkkrp/stache, pushed 2026-09-29 |
| `mustache` | 2.4.3.1 | 2025-05-11 | — | JustusAdam/mustache, pushed 2025-05-14 |
| `ginger` / `ginger2` | 0.10.6.0 / 2.2.0.0 | 2025-06-06 / 2025-06-05 | ginger 0.10.6.0 / same | tdammers/ginger, tdammers/ginger2 |
| `file-embed` | 0.0.16.0 | 2024-01-28 | — | snoyberg/file-embed, pushed 2026-10-30 |
| `double-conversion` | 2.0.5.0 | 2024-02-11 | — | — |

"—" means not checked. None of these packages is marked deprecated on Hackage. `cryptonite`, which `minio-hs` depends on, **is** deprecated: Hackage says `"is-deprecated": true`, in favour of `crypton` and the `cryptohash-*` packages, and its last upload was 2022-03-13.

## Upstream HTTP and JSON

### `http-client` / `http-client-tls` / `req`

- **TLS stack:** `http-client-tls` 0.4.0 depends on `crypton-connection` and `tls >= 2.1.2`. Its 0.4.0 changelog: MD5/Base16 now come from `cryptohash-md5` and `base16` "rather than `crypton` and `memory` (the latter is unmaintained)".
- **`req` 3.13.4** depends on `http-client >=0.7.13.1 && <0.8`, `http-client-tls >=0.3.2 && <0.5` and `crypton-connection >=0.4.2 && <0.5`. Its 3.13.3 changelog: "Works with `crypton-connection-0.4` and newer".
- **What the timeout covers:** in `http-client`, `ResponseTimeout` is "How to deal with timing out on retrieval of response headers". The default is `managerResponseTimeout`, "by default is 30 seconds" (`Network/HTTP/Client/Types.hs`). It does not bound reading the body. The Pilot's `AbortSignal.timeout` bounds the whole `fetch`, body included. `req` exposes it as `responseTimeout :: Int` (microseconds) (`Network/HTTP/Req.hs`).
- **`req` retries by default.** In `defaultHttpConfig`, `httpConfigRetryPolicy = retryPolicyDefault`, which is `constantDelay 50000 <> limitRetries 5` (`retry` 0.9.3.1, `Control/Retry.hs`). `httpConfigRetryJudge` retries on statuses 408, 504, 524, 598 and 599. `httpConfigRetryJudgeException` retries on `ResponseTimeout` and `ConnectionTimeout`. Non-2xx statuses throw `StatusCodeException` through `httpConfigCheckResponse`. All of these are fields that can be overridden. Plain `http-client` has no retry loop.

### `aeson` 2.3.2.0

- **Extra fields are ignored by default.** `defaultOptions` has `rejectUnknownFields = False`; set to `True` it fails "with an error message indicating which fields were unknown" (`Data/Aeson/Types/Internal.hs`).
- **The whole object exists before `FromJSON` runs.** `parseJSON :: Value -> Parser a` takes a built `Value`, so every field of an upstream row, dropped ones included, is held as a `Value` (an `Object` `KeyMap`) before the instance picks the 21 fields it wants. That differs from #11's serde `IgnoredAny`, where a dropped value is skipped without being built. A `FromJSON` instance can still count the unknown keys by reading the `KeyMap`'s keys.
- **The token layer:** `Data.Aeson.Decoding.Tokens` (since 2.1.2.0) exposes a token stream, `Tokens`/`TkRecord` with `TkPair !Key (Tokens …)`. A decoder written against it can walk past a field's tokens without building a `Value`. Even so, the lexer still unescapes each text token into `Text` (`TkText !Text`). Whether this layer is a supported public decoding API for user code: **not stated**, beyond its being an exposed module.
- **Numbers keep their written form at the token layer** as `NumInteger !Integer`, `NumDecimal !Scientific` or `NumScientific !Scientific`. In a `Value`, they are `Number Scientific`.

### Numbers as JS `String(n)` prints them

- `show :: Scientific` (scientific 0.3.10.0) prints `1` as `1.0`, and uses exponent form when the decimal exponent is `< 0` or `> 7` (`fmtAsGeneric`): `0.05` prints as `5.0e-2`. JS `String` gives `1` and `0.05`. So neither `show` nor `formatScientific Generic` reproduces `String(n)`.
- **`double-conversion` 2.0.5.0** binds Google's C++ `double-conversion` (the one written for V8). Its `toShortest` uses a converter built with `UNIQUE_ZERO`, `"Infinity"`, `"NaN"`, `'e'`, and decimal-in-shortest bounds `-6, 21` (`cbits/hs-double-conversion.cc`). Upstream's `EcmaScriptConverter()` uses the same arguments plus the `EMIT_POSITIVE_EXPONENT_SIGN` flag (`double-conversion/double-to-string.cc`). So the Haskell binding prints `1e21` where JS prints `1e+21`. Below 1e21, and above 1e-6 in magnitude, the two configurations match.
- Going through `Double` (`toRealFloat`) rounds large integers just as JS does. Keeping the `Scientific` keeps digits that JS would have lost.

## The Extract writer: CSV, trimming, case, hashing

### `cassava` 0.5.4.1

- **Encoding options:** `encDelimiter`, `encUseCrLf` (default `True`), `encIncludeHeader`, and `encQuoting` with `QuoteNone | QuoteMinimal | QuoteAll` (default `QuoteMinimal`) (`Data/Csv/Encoding.hs`).
- **Its quoting rule is the Pilot's.** Under `QuoteMinimal`, a field is quoted iff it contains `"`, the delimiter, LF or CR, and `"` is doubled. That is byte-for-byte `/[",\r\n]/` with `,` as the delimiter.
- **Row endings:** `unlines sep (b:bs) = b <> sep <> unlines sep bs`, so every record, the last one included, ends with the separator, as in the Pilot.
- **What it does not do:** emit a BOM, trim, or uppercase. It works on `ByteString` fields (`escape :: Quoting -> Word8 -> ByteString -> ByteString`), so trimming and case are done on `Text` before encoding either way. The hash is a separate pass, or a fold over the output's lazy chunks.
- The code is marked `-- TODO: Optimize` on `escape`. The changelog's last functional change is 0.5.4.0 (streaming decode, 2025-06). 0.5.4.1 is a bounds and GHC-support bump.

### Hand-written with `bytestring`'s `Builder`

- `bytestring` 0.12.2.0 provides `Data.ByteString.Builder`. `text` 2.1.4 stores UTF-8 internally and provides `encodeUtf8Builder`. A hand-written writer is the ADR 0009 shape #11 chose in Rust: one function, bytes plus hash plus counts. No library-specific facts beyond these.

### Where Haskell's `Text` functions differ from JS

- **Trim.** `Data.Text.strip` is `dropAround isSpace` (`text` 2.1.4). GHC's `isSpace` (`ghc-internal` 9.1401.0, `GHC/Internal/Unicode.hs`) is: for code points ≤ U+0377, `uc == 32 || uc - 0x9 <= 4 || uc == 0xa0` (space, TAB, LF, VT, FF, CR, NBSP); above that, `generalCategory c == Space` (Zs only). ECMAScript `String.prototype.trim` strips WhiteSpace (TAB, VT, FF, U+FEFF, any Zs) and LineTerminator (LF, CR, U+2028, U+2029) (ECMA-262, "WhiteSpace" and "LineTerminator" productions).
  - **U+FEFF, U+2028 and U+2029: JS strips them, `isSpace` does not.** (U+FEFF is category Cf; U+2028 and U+2029 are Zl and Zp.)
  - **U+0085: neither strips it.**
  - NBSP and the other Zs characters: both strip them (subject to each side's Unicode version).
  - #11's golden fixture puts U+FEFF and U+0085 at a value's edge, so a `strip`-based writer would differ from the Pilot on the U+FEFF case.
- **Uppercase.** `Data.Text.toUpper`'s haddock says "using simple case conversion" and, in the next sentence, that "ß" "maps to the two-letter sequence "SS"", which is a full (SpecialCasing) mapping. JS `toUpperCase` uses full mapping. `Data.Char.toUpper` is one-to-one per `Char`. What `text` actually does for every SpecialCasing entry beyond ß: **not stated** beyond that sentence.

### SHA-256

- **`crypton` 2.2.6** (`Crypto.Hash`): `hash`, `hashlazy`, and incremental `hashInit` / `hashUpdate` / `hashFinalize`. `show :: Digest a` prints lowercase hex (`Crypto/Hash/Types.hs`, via `encodeBase16`). It depends on `ram` (`>=0.20.1 && <0.23`); the 1.0.7 changelog says "Stop transitively depending on basement by depending on ram". 2.0.0 lists breaking changes in what input is accepted (RSA, Ed25519, PKCS#7, …) and says "No exported function changed its signature". The 2.2.6 changelog is a set of "Fixes from a security review", including "Hash: feed every update through updateBlocks".
- The LTS carries `crypton` 1.0.6 and the nightly 1.1.5. The 2.x line is in neither.
- `cryptohash-sha256` 0.11.102.1 is a standalone SHA-256 package, one of the packages Hackage names as `cryptonite`'s replacement.

## Deflate-only zip

### `zip` 2.2.2 (mrkkrp)

- **Methods:** `Store | Deflate | BZip2 | Zstd` (`Codec/Archive/Zip/Internal/Type.hs`). The cabal flags `disable-bzip2` and `disable-zstd` remove the `bzlib-conduit` and `conduit-zstd` (zstd C library) dependencies; both default to `False`. It is built on `conduit`.
- **It writes to a file path, not to memory.** `createArchive :: MonadIO m => FilePath -> ZipArchive a -> m a` "will silently overwrite the specified file". `commit` goes through `withNewFile`, which opens `openBinaryTempFile (takeDirectory fpath) ".zip"` and then `renameFile`s it into place (`Codec/Archive/Zip.hs`, `Internal.hs`). An in-memory `ByteString` archive means writing a temp file and reading it back. Entries are added with `addEntry :: CompressionMethod -> ByteString -> EntrySelector -> ZipArchive ()` or `sinkEntry` (a conduit source). The archive is read with `withArchive`.
- Recent changelog: 2.2.0 skips Zip64 extra fields for short strict input (`addEntry`); 2.0.1 "Fixed corruption of large entries when zip64 is used".

### `zip-archive` 0.5 (jgm)

- **Pure and in memory:** `fromArchive :: Archive -> Lazy.ByteString`, `toArchive` / `toArchiveOrFail`, `toEntry :: FilePath -> Integer -> Lazy.ByteString -> Entry`, `addEntryToArchive`. Methods are `Deflate | NoCompression`, compressed through `Codec.Compression.Zlib.Raw` (the `zlib` package) at zlib's default level. Dependencies are `zlib` and `digest`; no conduit (`zip-archive.cabal`).
- **`toEntry` stores an entry if deflate does not shrink it** ("only use compression if it helps!"). `Entry` is a plain record (`eCompressionMethod`, `eCRC32`, …), so an entry can be built with `Deflate` regardless. #11's "deflating both entries" would need that.
- **0.5 changelog (2026-09):** absolute and drive-qualified paths refused on extraction; file names decoded per the UTF-8 flag, with CP437 fallback; ZIP64 is not supported, and oversized entries now throw `Zip64NotSupported` instead of truncating silently; the general-purpose flag no longer claims maximum compression (writes `0x800`). Stackage still carries 0.4.3.2.

## S3 against Garage

### Garage's side (S3 compatibility page; source at `v2.5.0`)

- Signature v4: implemented; v2: missing. Path-style and vhost-style URLs: implemented. **Presigned URLs: implemented.** `PutObject`, `GetObject`, `HeadObject`, `CreateBucket`, `DeleteObject`: implemented. Missing endpoints "return a 501 Not Implemented", and "Some x-amz- headers are not implemented".
- **`PutBucketLifecycleConfiguration`: partially implemented.** "The only actions supported are AbortIncompleteMultipartUpload and Expiration (without the ExpiredObjectDeleteMarker field)", and "The deprecated Prefix member directly in the Rule structure/XML tag is not supported, specified prefixes must be inside the Filter structure/XML tag". In `src/api/common/xml/lifecycle.rs`, `<Filter>` may hold `And`, `Prefix`, `ObjectSizeGreaterThan` and `ObjectSizeLessThan`. Several conditions must be wrapped in `<And>`. `<Expiration>` needs exactly one of `<Days>` or `<Date>`.
- The page itself warns: "Some entries will be inexact or outdated. For any serious decision, you must make your own tests." Its update history ends at 2022-05-25.
- The default `s3_region` in Garage's configuration reference is `"garage"`. #8's `use_local_tz = false` condition on the lifecycle worker stands; nothing here changes it.

### `amazonka` / `amazonka-s3` 2.0

- **The Hackage release cannot be built on current GHCs without relaxing bounds.** `amazonka-core-2.0` (revision 3), `amazonka-2.0` (rev 1) and `amazonka-s3-2.0` (rev 1) all declare `base >=4.12 && <4.19`. base 4.19 ships with GHC 9.8, so the newest GHC these bounds allow is 9.6. `amazonka-core` also bounds `crypton >=0.32 && <0.35 || ^>=1.0.0`. The packages are in neither the current LTS nor the nightly.
- **Development continues on GitHub without a release.** The last GitHub release is `2.0.0` (2023-07-27). Commits on `main` since then include "Update crypton dependency bounds to support versions 2.0 and 2.1" (2026-09-28), a merge titled `build_with_9.14` (2026-09-30), and regenerated services (2026-10-25). Using that code means pinning a git revision (for example via `source-repository-package`).
- **Pointing it at Garage:** `setEndpoint :: Bool -> ByteString -> Int -> Service -> Service` (HTTPS?, host, port). `S3AddressingStyle = S3AddressingStyleAuto | S3AddressingStylePath | S3AddressingStyleVirtual`, set through `service_s3AddressingStyle`. S3's `defaultService` uses `Auto`, which rewrites to vhost style whenever the bucket is a valid DNS label. `Region` is an open newtype (`Region' {fromRegion :: Text}`), so `Region' "garage"` can be written. The service's `timeout` defaults to `Just 70` seconds (`Amazonka/S3/Types.hs`).
- **One `PutObject`:** `newPutObject` builds the request with `Request.putBody`. A strict or lazy `ByteString` body becomes `Hashed` (`toHashed x = HashedBytes (Crypto.hash x) x`), a single signed request. `Chunked` is the streaming alternative. amazonka-s3 has no automatic multipart.
- **Lifecycle:** `PutBucketLifecycleConfiguration` is sent with `Request.contentMD5Header . Request.s3vhost . Request.putXML`, so a Content-MD5 is computed for it. `LifecycleRule`'s XML writes `Filter` and the deprecated top-level `Prefix` only when they are `Just`. `LifecycleRuleFilter` is a record (`and`, `objectSizeGreaterThan`, `objectSizeLessThan`, `prefix`, `tag`), so `<Filter><Prefix></Prefix></Filter>` can be expressed.
- **Presign:** `Amazonka.Presign.presignURL :: AuthEnv -> Region -> UTCTime -> Seconds -> a -> ByteString`, plus `presignWith`, which takes a `Service -> Service` modifier (for the endpoint override).
- Size: `amazonka-s3.cabal` lists 290 `Amazonka.S3.*` modules. Licence MPL-2.0.

### `minio-hs` 1.7.0

- Last release 2023-05-22. It depends on the deprecated `cryptonite >= 0.25` and `cryptonite-conduit`. `tested-with: GHC == 8.6.5`. Last push to its repo was 2025-05-13; the repo is not archived (GitHub API). In neither LTS nor nightly.
- **Path style for non-AWS hosts:** "For AWS, always use virtual bucket style, unless bucket has periods. For MinIO and other non-AWS, default to path style" (`Network/Minio/API.hs`).
- **`putObject` with a known size ≤ 64 MiB is one PUT** (`putObjectSingle'`). Larger or unknown sizes go multipart (`Network/Minio/PutObject.hs`).
- **Presign:** `presignedGetObjectUrl`, `presignedPutObjectUrl`, `presignedHeadObjectUrl`, `presignedPostPolicy`.
- **No lifecycle call:** the `Network.Minio` export list has no bucket-lifecycle function (no match for "lifecycle").

### `aws` 0.25.3

- Its S3 commands are `CopyObject`, `DeleteBucket`, `DeleteObject(s)`, `GetBucket*`, `GetObject`, `HeadObject`, `PutBucket`, `PutBucketVersioning`, `PutObject`, `RestoreObject` and `Multipart`. **No lifecycle command** (`aws.cabal`).
- `RequestStyle = PathStyle | BucketStyle | VHostStyle`, and `S3SignV4` is supported. The `s3` default configuration uses `S3SignV2`, which Garage does not implement (`Aws/S3/Core.hs`). It depends on `crypton >= 0.34`.

## SMTP with login over a relay

### `smtp-mail` 0.5.0.1

- Depends on `crypton-connection >= 0.4.6 && < 0.5`, `crypton >= 1.1 && <2.3`, `ram`, `mime-mail` (`smtp-mail.cabal`). 0.5.0.1 changelog: "Switch to `ram` because the `memory` package is no longer maintained". 0.5.0.0 added `sendMailWithLoginOAuthSTARTTLS`.
- **Connection functions:** `connectSMTPSTARTTLS` (port 587), `connectSMTPS` (465), `connectSMTP` (25), the primed variants with an explicit port, and `connectSMTPWithHostNameAndTlsSettings[STARTTLS]` taking a `crypton-connection` `TLSSettings` (default `def`). There are one-shot senders: `sendMailWithLoginSTARTTLS`, `sendMailWithLoginTLS`, `sendMailWithLogin`.
- **Login:** `AUTH LOGIN` fails with `"authentication failed."` unless the final reply is 235. `login` is exported on its own (`Network/Mail/SMTP.hs`).
- **DATA:** it splits the rendered message on LF, strips a trailing CR, dot-stuffs, and writes each line with CRLF.
- **No timeout parameter:** no match for "timeout" in `Network/Mail/SMTP.hs`, so a bound would come from the caller (for example `System.Timeout`).

### `HaskellNet` 0.6.2 + `HaskellNet-SSL` 0.4.0.2

- `HaskellNet` covers SMTP, IMAP and POP3, and depends on `mime-mail >= 0.4 && < 0.6`. `Network.HaskellNet.SMTP` has `authenticate :: AuthType -> UserName -> Password -> SMTPConnection -> IO Bool` and `sendMail :: Mail -> SMTPConnection -> IO ()`, which takes the sender from `mailFrom`, the recipients from To/Cc/Bcc, and drops the Bcc header before rendering ("@since 0.6"). It throws `NoRecipients` when there is no recipient.
- `HaskellNet-SSL` adds `connectSMTPSTARTTLS[WithSettings]`, `connectSMTPSSL[WithSettings]`, the bracketed `doSMTPSTARTTLS` / `doSMTPSSL`, and `Settings { sslPort, sslMaxLineLength, sslLogToConsole, sslDisableCertificateValidation }`. It depends on `crypton-connection >= 0.3.1 && < 0.5`.

### `mime-mail` 0.5.2

- Builds the message: `simpleMail`, `addPart`, `plainPart`, `htmlPart` (both `QuotedPrintableText`, UTF-8), `addAttachmentBS`, and `renderMail'` (random boundary). 0.5 added inline images and `multipart/related`; 0.5.2 adds `Eq Mail`.
- **Non-ASCII headers** become one RFC 2047 `=?utf-8?Q?…?=` encoded word (`encodeIfNeeded` → `encodedWord`). The source shown does not split a long value into several encoded words. A Thai subject therefore becomes one long Q-encoded word.
- Header lines are rendered with `\n`; `smtp-mail`'s DATA step turns them into CRLF on the wire.
- **No Date and no Message-ID:** the source has no match for `Date` or `Message-ID` header generation. nodemailer adds both by default, so a Haskell sender would add them as explicit headers.

## Server-side HTML and email templates

| library | where the template lives | checked when | escaping |
|---|---|---|---|
| `lucid2` | Haskell code (combinators) | compile time | `toHtml` escapes, `toHtmlRaw` does not; `style_`/`script_` are `termRaw` |
| `blaze-html` | Haskell code (combinators) | compile time | escapes by default |
| `shakespeare` (Hamlet) | external `.hamlet` file through TH (`shamletFile`, `hamletFile`) or a quasi-quote (`shamlet`) | compile time | interpolation escapes |
| `stache` | `.mustache` files, parsed at compile time by TH (`compileMustacheDir`, `compileMustacheFile`, `mustache` QQ) | syntax at compile time; data at run time | Mustache rules |
| `heist` | template directory loaded at run time (`loadTemplates :: FilePath -> IO (Either [String] TemplateRepo)`) | run time | — |
| `ede` | file parsed at run time (`parseFile :: FilePath -> IO (Result Template)`), rendered against `HashMap Text Value` | run time | — |
| `ginger` / `ginger2` | Jinja templates, run time | run time | — |

- **`lucid2`:** the GitHub URL in its cabal file (`github.com/chrisdone/lucid`) now resolves to `chrisdone-archive/lucid`, which the GitHub API reports as **archived**. Its README says "lucid2 — the latest version", released under the "Immutable Publishing Policy". Releases still come: 0.0.20250303 "Support GHC 9.12", 0.0.20260427 "Support GHC 9.14". Where issues and PRs now go: **not stated**. Classes are plain string arguments (`class_ "btn btn-primary"`), the form Tailwind's scanner reads in #12's askama templates.
- **Hamlet:** `hamletFileWithSettings` reads the file through TH and calls `addDependentFile` (`Text/Shakespeare/Base.hs:317`), so editing a `.hamlet` file triggers recompilation. Classes can be written with Hamlet's `.class` shorthand or as `class="…"`. Whether Tailwind's scanner splits the shorthand form: not checked here (the same open question #12 raised against maud).
- **`stache`:** rendering is `renderMustache :: Template -> Value -> Lazy.Text`. A variable with no data is a run-time `MustacheVariableNotFound` warning (`renderMustacheW`), not a compile error. So a missing field is not caught when the template is built.
- `shakespeare` 2.2.0 adds `$component`. `heist` has had no repo push since 2024-07, although Hackage has a 2026-08 upload (probably a metadata revision; not confirmed).

## Template Haskell for copy and theme constants

- **Reading files at compile time:** `runIO` performs the read. `addDependentFile :: FilePath -> Q ()` "Record[s] external files that runIO is using … The compiler can then recognize that it should re-compile the Haskell file when an external file changes". It "Expects an absolute file path", "ghc -M does not know about these dependencies", and "The dependency is based on file content, not a modification time" (`ghc-internal` 9.1401.0, `GHC/Internal/TH/Monad.hs`). `getPackageRoot :: Q FilePath` exists for tools such as HLS that set `-package-root` instead of changing directory.
- **`file-embed` 0.0.16.0:** `embedFile :: FilePath -> Q Exp` (a `ByteString`), `embedStringFile`, and `makeRelativeToProject`, which "finds the first parent directory with a .cabal file, and uses that as the root directory". This is the counterpart of Rust's `include_bytes!` for the Data dictionary.
- **Files outside the package directory:** `messages/` and the theme sit at the repo root, outside a package in `apps/api`. Cabal's `cabal check` reports a path such as `extra-source-files: ../…` as "a relative path outside of the source tree. This will not work when generating a tarball with 'sdist'" (`Cabal/src/Distribution/PackageDescription/Check/Warning.hs`). A local `cabal build` does not need `sdist`.
- **Generating one function per key:** TH can decode the JSON with `aeson` inside `runIO` and emit a `SigD`/`FunD` pair per key, with one argument per `{placeholder}`. The 415 keys are all valid lowercase identifiers (see above). No Hackage package was found that does this for inlang or travelm-agency catalogues; the search was not exhaustive.
- **Theme hex literals:** the prototype's `--color-*: #rrggbb;` lines are a regular line format (see above). Reading them in TH is the same `runIO` + `addDependentFile` pattern; no library is involved.

## Unverified or not stated

- Whether Garage accepts an amazonka-signed `PutObject` and `PutBucketLifecycleConfiguration` unchanged (headers, checksums): not tested. The compatibility page asks readers to test.
- Whether the amazonka `main` branch builds on GHC 9.10 or 9.12 as it stands: only the commit titles were read.
- What `text`'s `toUpper` does for SpecialCasing entries other than ß.
- Whether Tailwind v4's scanner extracts classes from Hamlet's `.class` shorthand.
