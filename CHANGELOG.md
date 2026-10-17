# @lam/s3 Changelog

All notable changes to `@lam/s3` are documented here.

## 2.2.0 - 2026-10-24

### Added

- Server-side encryption with customer-provided keys (SSE-C):
  `S3Client.withCustomerKey(hexKey)` returns a client sharing the connection
  whose puts, gets, ranged and streaming reads, stat/exists, multipart
  uploads and copies send the AES-256 key (32 bytes, 64 hex characters).
  Presigned URLs are refused for such clients (they cannot carry the key).
  `hasCustomerKey()`.
- `copyFrom(sourceKey, destKey, sourceCustomerKey = "")`: server-side copy
  where the source is encrypted with another customer key (or none).
- `copyLarge(sourceKey, destKey, size, sourceCustomerKey = "", partSize)`:
  multipart server-side copy (`UploadPartCopy`) for objects above the 5 GiB
  single-copy limit; stores that answer `NotImplemented` (no UploadPartCopy,
  or none with SSE-C, as SeaweedFS) are copied part by part through the
  client instead.

### Testing

- `tests/live_sse_c.lam` (live, `LAM_S3_LIVE_TESTS=1`): round trip with a
  key, reads without the key or with another key refused, ranged and
  streaming reads, multipart upload, copy between keys, large copy with and
  without keys, presign refusal, invalid keys. Verified against SeaweedFS.

## 2.1.0 - 2026-10-15

### Added

- Multipart uploads: `createMultipartUpload`, `uploadPart` (`bytes`, `str` and
  `list[int]` overloads), `listParts` (paginated), `completeMultipartUpload`
  and `abortMultipartUpload`, with `S3Part` values and `S3.maxParts()`.
- Ranged and streaming reads: `getRange`, `getRangeBytes` and
  `open(key, offset, length) -> Result[S3ObjectReader]` with chunked
  `readBytes`/`read`, `close`, and the native `raw` reader for Go interop.
- Bucket helpers `bucketExists`, `createBucket` and `ensureBucket`.
- `put(key, data: bytes)` overload and the explicit `putStream` spelling for
  `any`-typed `io.Reader` values.
- `S3.endpointUrl(endpoint)` and argument validation returning
  `S3ArgumentError` before any network call.
- Live multipart/range test runnable against local S3-compatible endpoints;
  `--live` now runs every `tests/live_*.lam`.

### Fixed

- An explicit `http://` API endpoint is no longer rewritten to `https://`, so
  plain-HTTP gateways such as MinIO and SeaweedFS work. Bare hosts still
  default to HTTPS, and public URLs are still always HTTPS.
- `put(key, list[int])` delegated to the stream overload and always failed with
  "value does not implement io.Reader"; it now uploads the bytes.
- `tests/live_roundtrip.lam` compiles again under Lam 1.18.

### Performance

- `get` and `getRangeBytes` read the response body into one exactly sized
  buffer instead of `io.ReadAll`, which re-grew and re-copied multi-MiB bodies
  many times (about 70% of the CPU of a ranged-read workload).
- `downloadFile` streams to disk instead of buffering the whole object in
  memory.

## 2.0.0 - 2026-09-14

### Changed

- Replaced every `tryX` plus silent fallback pair with one unsuffixed,
  payload-typed `Result[T]` API. SDK, network, filesystem, and configuration
  failures are no longer converted to `false`, empty strings, empty objects, or
  empty collections.
- Replaced direct client construction with `S3Client.connect(...)` overloads
  and made both `connect` and `fromEnv` return `Result[S3Client]`; AWS config
  loading now reports `S3ConfigError` instead of panicking internally.
- Consolidated string, `list[int]`, and native `io.Reader` uploads under
  overloaded `put` methods. Object downloads now use `get -> Result[str]`, and
  key-only listing is exposed as `keys -> Result[list[str]]`.
- Changed `exists` to return non-not-found HEAD failures as `S3HeadError` rather
  than silently treating every SDK error as a missing key.
- Replaced package constructors named `__init__` with `init`, restricted `any`
  to AWS SDK/native interop boundaries, and removed avoidable `go!` fallback
  wrappers while retaining native AWS SDK calls.
- Raised the package major version to 2.0.0 and compiler compatibility to Lam
  `^1.18`.

### Removed

- Removed all 1.x `tryX` methods, boolean/string/list fallback twins,
  type-suffixed `putText`/`putBytes`/`putBuffer`/`putStream` methods,
  `getText`/`getBytes`/`getBuffer`, `listKeys`, and fallback presigned URL
  wrappers. No deprecated aliases remain.

### Testing

- Updated offline coverage for Result-returning factories, failed construction
  without a panic, typed `Result[T]` payloads, overloaded `put` dispatch, inert
  client errors, direct configuration, environment configuration, presigning,
  and deterministic public URLs.
- Added an installed-package offline pipeline composing context, JSON, bytes,
  crypto, connect overloads, presigning, public URLs, and inert client failures.
- Updated the optional live roundtrip source to the 2.x API; it remains strictly
  opt-in and was not used for release verification.

## 2026-09-13

### Changed

- Raised compiler compatibility to Lam 1.16, documented path, Git, and registry
  installation, and added repository metadata plus package test/format scripts.
- Reused `lambytes` for buffer/string conversion while preserving the public S3
  API and AWS SDK boundaries.

### Fixed

- Made the test runner stage cases as a scoped consumer dependency without
  relying on the checkout's parent directories.
- Replaced stale pre-scope package identifiers in current source, tests, and
  notices.

## 2026-09-12

### Changed

- Renamed the canonical library module from its legacy unscoped name to
  `@lam/s3` and updated the manifest, imports, tests, and documentation.
- Updated the test runner to use `lamc` from `PATH`, require a compatible
  compiler before running, and stage the standalone checkout in the scoped
  extlibs layout expected by Lammergeier.
- Updated README paths and commands for the standalone `lam-s3` repository.

### Fixed

- Adjusted the presigned URL regression test to use compiler-supported string
  method syntax.

## 2026-07-11

### Added

- Added standalone Apache-2.0 `LICENSE` and package-specific `NOTICE` files for
  distribution from the external `thallium-solutions/lam-s3` repository.
- Added this package-specific changelog.

### Changed

- Updated package metadata to advertise `Apache-2.0` licensing.

## 2026-07-10

### Added

- Added Result-returning S3 operations for text/bytes object I/O, buffer
  upload/download, stream upload, file upload/download, object stat/head
  metadata, copy, move, key-only listing, bulk delete, config validation, and
  presigned GET/PUT URLs.
- Added direct initialization through `S3Config(...)` and `S3Client.connect(...)`
  for applications that load credentials outside environment variables.
- Added offline coverage for direct configuration, `Result`/`do-catch` config
  handling, presigned URLs, and object defaults.
- Expanded live R2/S3 roundtrip coverage using a dedicated test-object prefix.

### Changed

- Simplified environment configuration to use `S3_BUCKET` instead of separate
  public/private bucket variables.
- Updated AWS SDK for Go v2 pins, including
  `github.com/aws/aws-sdk-go-v2/service/s3 v1.105.0`.
- Expanded README examples for direct configuration, Result-first flows, files,
  buffers, streams, metadata, presigned URLs, and tests.

## 2026-05-09

### Added

- Added offline tests for environment configuration, readiness checks, static
  and client-level public URL generation, URL encoding variants, and
  `S3Object` defaults.
- Added README coverage for configuration, usage, API reference, offline tests,
  and optional live S3-compatible roundtrip testing.

### Changed

- Updated the package test runner to discover every `offline_*.lam` regression
  case automatically.

## 2026-05-07

### Added

- Added the initial S3 package under `third_party/` with S3-compatible
  configuration, object upload/download, listing, deletion, public URL helpers,
  and optional live integration testing.
