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
prefix; the client normalizes it before configuring the AWS SDK.

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

A third `put(key, stream: any, contentType)` overload accepts a native Go value
that implements `io.Reader`. This is an intentional SDK/native boundary; normal
Lam data should use the typed `str` or `list[int]` overload.

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
| `put(key, stream: any, contentType="application/octet-stream")` | `Result[bool]` | Overloaded native `io.Reader` upload. |
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
`metadata`.

`S3.publicUrl(publicEndpoint, key)` builds a URL without a client, and
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

The live roundtrip is opt-in, uses the isolated `lam-s3-tests/live-roundtrip/`
prefix, and must never be run as part of offline verification:

```bash
export LAM_S3_LIVE_TESTS=1
# set S3_* credentials explicitly
python3 tests/run_lams3_tests.py --live --verbose
```

The package pins AWS SDK for Go v2 modules, including
`github.com/aws/aws-sdk-go-v2/service/s3 v1.105.0`.

## License

Copyright 2026 Thallium Solutions di Busconi Alessandro. Distributed under the
Apache License, Version 2.0. See `LICENSE`, `NOTICE`, and `CHANGELOG.md`.
