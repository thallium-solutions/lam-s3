# @lam/s3

<p align="center">
  <img src="assets/logo-full.png" alt="Lammergeier logo" width="420">
</p>

`@lam/s3` is a Lammergeier client for AWS S3 and compatible object stores such
as Cloudflare R2, MinIO, Wasabi, Backblaze B2, and custom gateways.

Version 2 is a deliberately breaking, Result-first API. Expected configuration,
filesystem, SDK, and network failures are never converted to `false`, `""`, or
empty collections. Every fallible operation has one unsuffixed `Result[T]`
method; callers can inspect it, propagate it with `?`, handle it with `do/catch`,
or explicitly opt into a panic with `unwrap()`.

## Requirements and installation

`@lam/s3` 2.x requires Lam 1.18 or newer.

```bash
# Local path
lamc install /path/to/lam-s3

# Git
lamc install https://github.com/thallium-solutions/lam-s3.git

# Registry
lamc install @lam/s3
```

The canonical package name is `@lam/s3`, regardless of the checkout directory
or installation source.

## Configuration

```bash
export S3_ACCESS_KEY_ID="..."
export S3_SECRET_ACCESS_KEY="..."
export S3_ENDPOINT="account-id.r2.cloudflarestorage.com"
export S3_PUBLIC_ENDPOINT="bucket.example.com" # optional
export S3_REGION="auto"                         # optional; default: auto
export S3_BUCKET="bucket-name"
export S3_USE_PATH_STYLE="true"                 # optional; default: true
```

Do not commit credentials. `S3_ENDPOINT` may include an `http://` or `https://`
prefix. An explicit scheme is kept — local gateways such as MinIO or SeaweedFS
(`S3_ENDPOINT=http://localhost:9000`) are reached over plain HTTP — and a bare
host defaults to HTTPS. `S3.endpointUrl(endpoint)` returns the exact URL handed
to the SDK.

| Variable | Required | Purpose |
|----------|----------|---------|
| `S3_ACCESS_KEY_ID` | Yes | S3-compatible access key. |
| `S3_SECRET_ACCESS_KEY` | Yes | S3-compatible secret key. |
| `S3_ENDPOINT` | Yes | API endpoint. |
| `S3_PUBLIC_ENDPOINT` | No | CDN/custom domain used by `publicUrl`. |
| `S3_REGION` | No | Region; defaults to `auto`. |
| `S3_BUCKET` | Yes | Bucket used by object operations. |
| `S3_USE_PATH_STYLE` | No | Defaults to `true`; `false`, `0`, or `no` disables it. |

## Quick start

Client construction can fail and therefore uses an explicit factory. It never
panics internally while loading AWS configuration.

```lammergeier
from @lam/s3 import S3Client
from lamerrors import Result

func uploadNote() -> Result[bool] {
    s3: S3Client = S3Client.fromEnv()?
    s3.put("notes/hello.txt", "hello from Lam", "text/plain")?

    body: str = s3.get("notes/hello.txt")?
    print(body)
    print(s3.publicUrl("notes/hello.txt"))

    return s3.delete("notes/hello.txt")
}

func main() {
    do {
        uploadNote()?
    } catch err {
        print(f"s3 failed: {err}")
    }
}
```

### Direct configuration

`connect` is overloaded. Pass either an `S3Config` or the individual fields:

```lammergeier
from @lam/s3 import S3Client, S3Config
from lamerrors import Result

func configuredClient() -> Result[S3Client] {
    cfg: S3Config = S3Config(
        "access-key",
        "secret-key",
        "account-id.r2.cloudflarestorage.com",
        "app-bucket",
        "auto",
        "cdn.example.com",
        true,
    )
    return S3Client.connect(cfg)
}

func directClient() -> Result[S3Client] {
    return S3Client.connect(
        "access-key",
        "secret-key",
        "account-id.r2.cloudflarestorage.com",
        "app-bucket",
    )
}
```

Calling `S3Client()` only creates an inert value for compiler/native plumbing.
Operations on it return `S3ConfigError`; applications should use `connect` or
`fromEnv`.

## Overloaded object data

`put` uses Lam overloads instead of type-suffixed method names:

```lammergeier
from @lam/s3 import S3Client
from lambytes import Bytes
from lamerrors import Result

func examples(s3: S3Client) -> Result[bool] {
    # str overload; content type defaults to text/plain.
    s3.put("notes/today.txt", "ready")?

    # list[int] overload; content type defaults to application/octet-stream.
    raw: list[int] = [0, 1, 2, 255]
    s3.put("raw/blob.bin", raw)?

    # Object bodies are returned losslessly as str. Convert only when a Lam
    # byte buffer is required.
    encoded: str = s3.get("raw/blob.bin")?
    roundTrip: list[int] = Bytes.toBytes(encoded)
    print(len(roundTrip))
    return Result.Ok(true)
}
```

A `put(key, data: bytes, contentType)` overload uploads a native byte slice
without conversion, and `put(key, stream: any, contentType)` accepts a native Go
value that implements `io.Reader`. Because every overload accepts an `any`
argument, use the explicit `putStream(key, stream, contentType)` spelling for
`any`-typed values. Normal Lam data should use the typed `str`, `bytes` or
`list[int]` overload.

## Large objects: multipart uploads, ranged and streaming reads

Objects above a few hundred MiB should be uploaded in parts: each part except
the last must be at least 5 MiB on AWS S3, at most 10,000 parts are allowed
(`S3.maxParts()`), and an unfinished upload can be resumed by listing its parts
or discarded with `abortMultipartUpload`. Reads can fetch a byte range or
stream the body in chunks without buffering the whole object.

```lammergeier
from @lam/s3 import S3Client, S3ObjectReader, S3Part
from lamerrors import Result

func uploadInParts(s3: S3Client, key: str, chunks: list[bytes]) -> Result[int] {
    uploadId: str = s3.createMultipartUpload(key, "video/mp4")?
    parts: list[S3Part] = []
    do {
        for index, chunk in enumerate(chunks) {
            part: S3Part = s3.uploadPart(key, uploadId, index + 1, chunk)?
            parts.append(part)
        }
        _ = s3.completeMultipartUpload(key, uploadId, parts)?
    } catch err {
        _ = s3.abortMultipartUpload(key, uploadId)
        return Result.Err(err)
    }
    return Result.Ok(len(parts))
}

func preview(s3: S3Client, key: str) -> Result[str] {
    return s3.getRange(key, 0, 1024)          # first KiB
}

func countBytes(s3: S3Client, key: str) -> Result[int] {
    reader: S3ObjectReader = s3.open(key)?     # or open(key, offset, length)
    total: int = 0
    while true {
        chunk: bytes = reader.readBytes(1048576)?
        if len(chunk) == 0 {
            break
        }
        total += len(chunk)
    }
    _ = reader.close()?
    return Result.Ok(total)
}
```

`S3ObjectReader.raw` is the native `io.ReadCloser`, for Go interop such as
copying a range straight into an HTTP response.

## Encryption with customer keys (SSE-C)

```lammergeier
plain: S3Client = S3Client.fromEnv()?
keyed: S3Client = plain.withCustomerKey(hexKey)?     # 64 hex characters
_ = keyed.put("reports/q3.pdf", data, "application/pdf")?
body: str = keyed.get("reports/q3.pdf")?            # plain.get(...) fails
_ = other.copyFrom("reports/q3.pdf", "archive/q3.pdf", hexKey)?
_ = keyed.copyLarge("big.iso", "copy.iso", size)?   # objects above 5 GiB
```

The store keeps only a hash of the key: reading needs the same key, and
losing it loses the data. Every object operation of a keyed client sends the
key (puts, gets, ranges, streams, stat, multipart, copies); presigned URLs
are refused. `copyLarge` uses `UploadPartCopy` and falls back to a
part-by-part copy through the client where the store does not implement it.

## Buckets

`bucketExists()` returns `Ok(false)` only when the configured bucket is
missing; `createBucket()` returns `Ok(false)` when it already exists and is
owned by the same credentials; `ensureBucket()` creates it on demand, which is
convenient for development and test environments.

## Endpoints supplied by users (SSRF protection)

Applications that let their users bring their own bucket must not let a
crafted endpoint reach internal services (cloud metadata at
`169.254.169.254`, databases on private networks, `localhost`). Set
`blockPrivateNetworks` on the configuration:

```lammergeier
cfg: S3Config = S3Config(keyId, secret, userEndpoint, userBucket, "eu-west-1")
cfg.blockPrivateNetworks = true
client: S3Client = S3Client.connect(cfg)?
```

Every connection is then checked **after DNS resolution**, so a name that
resolves (or later re-resolves) to a private address is refused as well:
loopback, RFC 1918, link-local, CGNAT (`100.64.0.0/10`), benchmarking,
multicast, reserved, unspecified, IPv6 ULA/link-local and NAT64 ranges.
Environment proxies are not used by such clients (a proxy would hide the
destination). `S3.isPublicAddress(ip)` exposes the same check.

## Object Lock: WORM retention and legal holds

S3 Object Lock keeps object versions from being deleted or overwritten until
a retain-until date (**retention**) or until it is released (**legal
hold**). It needs a bucket created with Object Lock, which also enables
versioning (neither can be turned off later). Verified against AWS S3
semantics and SeaweedFS 4.x; MinIO behaves the same.

```lammergeier
from @lam/s3 import S3Client, S3ObjectLockConfig, S3Retention
from lamerrors import Result, Error
from lamtime import Time

func archive(client: S3Client, key: str, body: str) -> Result[bool] {
    _ = client.createBucketWithObjectLock()?              # Ok(false) if it exists
    lock: S3ObjectLockConfig = client.objectLockConfig()?
    if not lock.enabled {
        return Result.Err(Error("Config", "the bucket has no Object Lock"))
    }
    _ = client.put(key, body)?
    _ = client.setRetention(key, "COMPLIANCE", Time.nowUnix() + 7 * 86400)?
    _ = client.setLegalHold(key, true)?                   # until released
    r: S3Retention = client.retention(key)?
    print(f"{r.mode} until {r.untilUnix}, held={client.legalHold(key)?}")
    return Result.Ok(true)
}
```

- **Modes.** `GOVERNANCE` retention can be shortened or removed (and the
  version deleted) with `bypassGovernance = true` by an identity that has
  `s3:BypassGovernanceRetention`; `COMPLIANCE` retention can only be
  extended, by anyone, and nothing deletes the version before the date. A
  legal hold blocks deletion in both modes, bypass included.
- **Versions.** In a versioned bucket `delete(key)` only adds a delete
  marker; the locked data version stays (and is billed). `versions(prefix)`
  lists versions and delete markers, `deleteVersion(key, versionId,
  bypassGovernance = false)` removes one for good.
- **Refusals** (deleting or shortening a locked version) are
  `S3ObjectLocked` errors; other Object Lock failures are
  `S3ObjectLockError`, argument errors `S3ArgumentError` before any request.
- **Removing a governance retention**: `setRetention(key, "", 0, true)`.
  Stores that reject an empty retention (SeaweedFS) get a governance
  retention ending one second later instead.
- `setObjectLockDefault(mode, days)` sets the bucket's default retention for
  versions written without one (`""` removes it).

## Files, metadata, lists, and URLs

```lammergeier
from @lam/s3 import S3Client, S3Object
from lamerrors import Result

func workflow(s3: S3Client) -> Result[bool] {
    s3.uploadFile("report.pdf", "reports/report.pdf", "application/pdf")?
    s3.downloadFile("reports/report.pdf", "/tmp/report.pdf")?

    obj: S3Object = s3.stat("reports/report.pdf")?
    print(f"{obj.size} bytes, {obj.contentType}")

    objects: list[S3Object] = s3.list("reports/")?
    keys: list[str] = s3.keys("reports/")?
    print(len(objects) == len(keys))

    getUrl: str = s3.presignGet("reports/report.pdf", 900)?
    putUrl: str = s3.presignPut("incoming/report.pdf", "application/pdf", 900)?
    print(getUrl)
    print(putUrl)
    return Result.Ok(true)
}
```

## API reference

### `S3Config`

| API | Return | Description |
|-----|--------|-------------|
| `S3Config(accessKeyId="", secretAccessKey="", endpoint="", bucket="", region="auto", publicEndpoint="", usePathStyle=true)` | `S3Config` | Builds direct configuration. |
| `S3Config.fromEnv()` | `S3Config` | Reads the `S3_*` environment variables. |
| `cfg.isReady()` | `bool` | Checks that all required fields are populated. |
| `cfg.validate()` | `Result[bool]` | Returns `S3ConfigError` for the first missing field. |

Class constructors in this package are defined with `init`. Lam continues to
accept the legacy `__init__` spelling as an alias at the language level, but the
package source and examples use `init`.

### `S3Client` factories

| API | Return | Description |
|-----|--------|-------------|
| `S3Client.connect(cfg)` | `Result[S3Client]` | Validates a config and constructs the AWS SDK client. |
| `S3Client.connect(accessKeyId, secretAccessKey, endpoint, bucket, region="auto", publicEndpoint="", usePathStyle=true)` | `Result[S3Client]` | Overloaded direct factory. |
| `S3Client.fromEnv()` | `Result[S3Client]` | Loads environment config and constructs a client. |

### `S3Client` operations

| API | Return | Description |
|-----|--------|-------------|
| `put(key, data: str, contentType="text/plain")` | `Result[bool]` | Uploads string data. |
| `put(key, data: list[int], contentType="application/octet-stream")` | `Result[bool]` | Overloaded byte-buffer upload. |
| `put(key, data: bytes, contentType="application/octet-stream")` | `Result[bool]` | Overloaded native byte-slice upload. |
| `put(key, stream: any, contentType="application/octet-stream")` | `Result[bool]` | Overloaded native `io.Reader` upload. |
| `putStream(key, stream: any, contentType="application/octet-stream")` | `Result[bool]` | Unambiguous native `io.Reader` upload for `any` values. |
| `getRange(key, offset, length)` | `Result[str]` | Reads a byte range as a binary-safe string. |
| `getRangeBytes(key, offset, length)` | `Result[bytes]` | Reads a byte range as `bytes`. |
| `open(key, offset=0, length=-1)` | `Result[S3ObjectReader]` | Streams the object (or a range) in chunks. |
| `createMultipartUpload(key, contentType="application/octet-stream")` | `Result[str]` | Starts a multipart upload and returns its upload id. |
| `uploadPart(key, uploadId, partNumber, data: bytes \| str \| list[int])` | `Result[S3Part]` | Uploads one part (1..10000). |
| `uploadPartReader(key, uploadId, partNumber, reader: any, size)` | `Result[S3Part]` | Uploads one part streamed from a native `io.ReadSeeker` of exactly `size` bytes (re-read on SDK retries); `S3TypeError` for other values. |
| `uploadPartFile(key, uploadId, partNumber, localPath, offset = 0, length = -1)` | `Result[S3Part]` | Uploads `length` bytes of a local file from `offset` (to the end when negative), streamed from disk; `S3FileError` when the file cannot be opened. |
| `listParts(key, uploadId)` | `Result[list[S3Part]]` | Lists uploaded parts, following pagination. |
| `completeMultipartUpload(key, uploadId, parts)` | `Result[bool]` | Assembles ascending parts into the object. |
| `abortMultipartUpload(key, uploadId)` | `Result[bool]` | Discards an unfinished upload. |
| `bucketExists()` / `createBucket()` / `ensureBucket()` | `Result[bool]` | Bucket checks and creation. |
| `createBucketWithObjectLock()` | `Result[bool]` | Creates the bucket with Object Lock (and versioning); `Ok(false)` when it exists. |
| `objectLockConfig()` | `Result[S3ObjectLockConfig]` | `enabled`, `defaultMode`, `defaultDays`, `defaultYears` (`enabled` false without Object Lock). |
| `setObjectLockDefault(mode, days)` | `Result[bool]` | Default retention of new versions (`""` removes it). |
| `setRetention(key, mode, untilUnix, bypassGovernance = false, versionId = "")` | `Result[bool]` | Places, extends, shortens (governance + bypass) or removes (`""`, `0`) a retention; `S3ObjectLocked` when refused. |
| `retention(key, versionId = "")` | `Result[S3Retention]` | `mode` and `untilUnix` (empty when none); `isActive(nowUnix)`. |
| `setLegalHold(key, on, versionId = "")` / `legalHold(key, versionId = "")` | `Result[bool]` | Places, releases or reads a legal hold. |
| `versions(prefix = "")` | `Result[list[S3ObjectVersion]]` | Versions and delete markers (`key`, `versionId`, `size`, `etag`, `lastModifiedUnix`, `isLatest`, `isDeleteMarker`), all pages. |
| `deleteVersion(key, versionId, bypassGovernance = false)` | `Result[bool]` | Permanently deletes one version or delete marker; `S3ObjectLocked` when locked or held. |
| `get(key)` | `Result[str]` | Downloads an object body without hiding failures. |
| `uploadFile(localPath, key, contentType="application/octet-stream")` | `Result[bool]` | Uploads a local file. |
| `downloadFile(key, localPath)` | `Result[bool]` | Downloads to a local file. |
| `exists(key)` | `Result[bool]` | Returns `Ok(false)` only for missing keys; other HEAD failures are errors. |
| `list(prefix="")` | `Result[list[S3Object]]` | Lists object metadata. |
| `keys(prefix="")` | `Result[list[str]]` | Projects a listing to keys. |
| `stat(key)` | `Result[S3Object]` | Reads object metadata with HEAD. |
| `delete(key)` | `Result[bool]` | Deletes one object. |
| `deleteMany(keys)` | `Result[int]` | Deletes objects and returns their count. |
| `copy(sourceKey, destKey, contentType="")` | `Result[bool]` | Copies within the configured bucket. |
| `move(sourceKey, destKey, contentType="")` | `Result[bool]` | Copies, then deletes the source. |
| `presignGet(key, expiresSeconds=900)` | `Result[str]` | Generates a temporary GET URL. |
| `presignPut(key, contentType="application/octet-stream", expiresSeconds=900)` | `Result[str]` | Generates a temporary PUT URL. |
| `publicUrl(key)` | `str` | Builds a deterministic public URL from `publicEndpoint`. |

`S3Object` fields are `key`, `size`, `etag`, `lastModified`, `contentType`, and
`metadata`. `S3Part` has `partNumber`, `etag` and `size`. `S3ObjectReader` has
`key`, `offset`, `size` (bytes in the opened range), `objectSize`, `contentType`,
`etag`, `raw`, and the methods `readBytes(maxBytes=65536) -> Result[bytes]`,
`read(maxBytes=65536) -> Result[str]` and `close() -> Result[bool]`.
Argument errors (empty keys or upload ids, part numbers outside 1..10000,
non-ascending parts, negative offsets) are returned as `S3ArgumentError`
before any network call.

`S3Config.blockPrivateNetworks` (default `false`) refuses private and
reserved addresses at connection time; `S3.isPublicAddress(ip)` is the same
check. `S3.checkLockMode(mode)` normalizes an Object Lock mode (`S3ArgumentError`
for anything but governance/compliance). `S3.publicUrl(publicEndpoint, key)`
builds a URL without a client, and
`S3.hasEnv()` checks whether required environment configuration is present.
Public URL helpers always emit HTTPS and path-escape keys while preserving `/`.

## 1.x to 2.x API map

| Removed 1.x API | 2.x replacement |
|-----------------|-----------------|
| `S3Client(cfg)` | `S3Client.connect(cfg) -> Result[S3Client]` |
| `S3Client.connect(...) -> S3Client` | `S3Client.connect(...) -> Result[S3Client]` |
| `S3Client.fromEnv() -> S3Client` | `S3Client.fromEnv() -> Result[S3Client]` |
| `putText` / `tryPutText` | `put(str) -> Result[bool]` |
| `putBytes` / `tryPutBytes` | `put(str, contentType) -> Result[bool]` |
| `putBuffer` / `tryPutBuffer` | `put(list[int]) -> Result[bool]` |
| `putStream` / `tryPutStream` | `put(any) -> Result[bool]` at the native boundary |
| `getText` / `tryGetText` / `getBytes` / `tryGetBytes` | `get -> Result[str]` |
| `getBuffer` / `tryGetBuffer` | `Bytes.toBytes(client.get(key)?)` |
| `uploadFile` / `tryUploadFile` | `uploadFile -> Result[bool]` |
| `downloadFile` / `tryDownloadFile` | `downloadFile -> Result[bool]` |
| `delete` / `tryDelete` | `delete -> Result[bool]` |
| `deleteMany` / `tryDeleteMany` | `deleteMany -> Result[int]` |
| `exists` / `tryExists` | `exists -> Result[bool]` |
| `list` / `tryList` | `list -> Result[list[S3Object]]` |
| `listKeys` / `tryListKeys` | `keys -> Result[list[str]]` |
| `stat` / `tryStat` | `stat -> Result[S3Object]` |
| `copy` / `tryCopy` | `copy -> Result[bool]` |
| `move` / `tryMove` | `move -> Result[bool]` |
| `presignedGetUrl` | `presignGet -> Result[str]` |
| `presignedPutUrl` | `presignPut -> Result[str]` |

There are no compatibility aliases for these removed methods; 2.0 is a major
release specifically so failures cannot accidentally remain silent.

## Testing

Offline tests compile and run without credentials and are the default:

```bash
python3 tests/run_lams3_tests.py --verbose
```

Live tests (`tests/live_*.lam`: roundtrip, multipart, ranged and streaming
reads) are opt-in, use isolated `lam-s3-tests/` prefixes, and must never be run
as part of offline verification. They work against any S3-compatible endpoint,
including a disposable local one:

```bash
docker run --rm -d --name lam-s3-live -p 8333:8333 \
  -v "$PWD/s3.json:/etc/seaweedfs/s3.json:ro" chrislusf/seaweedfs:4.48 \
  server -dir=/data -s3 -s3.port=8333 -s3.config=/etc/seaweedfs/s3.json
export LAM_S3_LIVE_TESTS=1 S3_ENDPOINT=http://127.0.0.1:8333 S3_BUCKET=lam-s3-live
# set S3_ACCESS_KEY_ID / S3_SECRET_ACCESS_KEY to the identity in s3.json
python3 tests/run_lams3_tests.py --live --verbose
```

`live_multipart_range.lam` creates the bucket with `ensureBucket()` when needed.

The package pins AWS SDK for Go v2 modules, including
`github.com/aws/aws-sdk-go-v2/service/s3 v1.105.0`.

## License

Copyright 2026 Thallium Solutions di Busconi Alessandro. Distributed under the
Apache License, Version 2.0. See `LICENSE`, `NOTICE`, and `CHANGELOG.md`.
