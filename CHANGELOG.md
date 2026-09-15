# @lam/s3 Changelog

All notable changes to `@lam/s3` are documented here.

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
