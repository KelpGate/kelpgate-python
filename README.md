# Kelpgate Client for Python

[![PyPI](https://img.shields.io/pypi/v/kelpgate-client)](https://pypi.org/project/kelpgate-client/)
[![CI](https://github.com/KelpGate/kelpgate-python/actions/workflows/ci.yml/badge.svg)](https://github.com/KelpGate/kelpgate-python/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

Official Python client for the [Kelpgate](https://kelpgate.cloud) malware-scanning API. It provides synchronous and asyncio clients with identical interfaces, manages OAuth 2.0 access tokens, retries transient failures idempotently and maps API responses to typed results and exceptions. The package includes FastAPI and Django integrations and a command-line interface. Its only runtime dependency is [httpx](https://www.python-httpx.org/).

The package is open source under the MIT licence. The Kelpgate scanning service is a commercial hosted product; using it requires a Kelpgate account.

## Contents

- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Scan results](#scan-results)
- [Errors](#errors)
- [Retries and idempotency](#retries-and-idempotency)
- [FastAPI](#fastapi)
- [Django](#django)
- [Command-line interface](#command-line-interface)
- [Failure policy](#failure-policy)
- [Security](#security)
- [Troubleshooting](#troubleshooting)
- [Compatibility and versioning](#compatibility-and-versioning)
- [Development](#development)
- [Support](#support)
- [Licence](#licence)

## Requirements

- Python 3.9 or later.
- A Kelpgate API client (client ID and secret) created in the [Kelpgate Console](https://console.kelpgate.cloud).

## Installation

```bash
pip install kelpgate-client
```

Optional integrations:

```bash
pip install "kelpgate-client[fastapi]"
pip install "kelpgate-client[django]"
```

The distribution is `kelpgate-client`; the import package is `kelpgate`.

## Configuration

```python
from kelpgate import KelpgateClient

client = KelpgateClient(
    "https://api.kelpgate.cloud",
    token_url="https://auth.kelpgate.cloud/oauth2/token",
    client_id="...",
    client_secret="...",
)
```

| Parameter | Default | Description |
|---|---|---|
| `api_url` | required | Base URL of the Kelpgate API (first positional argument) |
| `token_url` | required with client credentials | OAuth 2.0 token endpoint |
| `client_id`, `client_secret` | | Client credentials |
| `bearer_token` | | Static access token, used instead of client credentials |
| `scope` | `"scan"` | OAuth 2.0 scope requested |
| `timeout` | `60.0` | Per-request timeout in seconds |
| `max_retries` | `2` | Retries for network errors and HTTP 429, 502, 503 and 504 |
| `retry_delay` | `0.5` | Base delay in seconds for exponential backoff |
| `max_bytes` | 128 MiB | Largest file the client sends; larger files fail before upload |
| `transport` | | Custom `httpx` transport, for proxies or tests |

`api_url` and `token_url` must use HTTPS; plain HTTP is accepted only for `localhost`.

`KelpgateClient.from_env()` and `AsyncKelpgateClient.from_env()` read:

| Variable | Default |
|---|---|
| `KELPGATE_CLIENT_ID`, `KELPGATE_CLIENT_SECRET` | |
| `KELPGATE_TOKEN` | Static token, instead of client credentials |
| `KELPGATE_API_URL` | `https://api.kelpgate.cloud` |
| `KELPGATE_TOKEN_URL` | `https://auth.kelpgate.cloud/oauth2/token` |

Use clients as context managers so that connections are released:

```python
with KelpgateClient.from_env() as client:
    ...
```

## Usage

```python
from kelpgate import KelpgateClient, assert_clean

with KelpgateClient.from_env() as client:
    result = client.scan_file("upload.pdf")
    if result.infected:
        ...  # reject the upload

    # Alternatively, raise InfectedError for an infected file:
    assert_clean(client.scan_bytes(data, filename="report.docx"))
```

The asyncio client has the same methods:

```python
from kelpgate import AsyncKelpgateClient

async with AsyncKelpgateClient.from_env() as client:
    result = await client.scan_bytes(data, filename="report.docx")
```

| Method | Description |
|---|---|
| `scan_file(source, ...)` | Scan a path, `bytes` or binary file object |
| `scan_bytes(data, ...)` | Scan in-memory content |
| `scan_url(url, ...)` | Have the service fetch and scan a URL |
| `scan_file_async(source, ...)`, `scan_url_async(url, ...)` | Submit a background scan; returns `AcceptedScan` with `scan_id`. Accepts `webhook_url`. |
| `get_async_result(scan_id)` | Returns `PendingScan` until the result is available, then `ScanResult` |
| `wait_for_result(scan_id, interval=1, timeout=120)` | Poll until the result is available |
| `health()` | Service readiness; requires no credentials |

Upload methods accept `filename` and `idempotency_key`. Paths and seekable file objects are streamed; other streams are buffered in memory up to `max_bytes`. The 0.1 names `submit_file`, `submit_url`, `get_result` and `wait` remain available as aliases.

## Scan results

An infected file is a result, not an exception. `ScanResult` is an immutable dataclass:

| Attribute | Type | Description |
|---|---|---|
| `infected` | `bool` | `True` for malware or a policy rejection |
| `status` | `str` | `CLEAN` or `INFECTED` |
| `signature` | `str` | Detection names, joined with ` \| ` |
| `policy_rejected` | `bool` | Rejected by policy (for example an encrypted archive) rather than detected |
| `degraded` | `bool` | An engine did not complete; coverage was reduced |
| `sha256` | `str \| None` | SHA-256 of the scanned content |
| `rule_authors` | `tuple[str, ...]` | Authors of matching detection rules. Preserve them when displaying or forwarding a detection, as required by the Detection Rule License. |
| `scan_id`, `request_id` | `str \| None` | Identifiers; quote `request_id` in support requests |
| `filename`, `timestamp`, `s3_path`, `download_url` | `str \| None` | As returned by the service |
| `raw` | `dict` | Response body as received |

Integrations that require full coverage should treat `degraded` as not verified.

## Errors

All exceptions derive from `KelpgateError` and expose `status`, `code`, `request_id` and `body`.

| Exception | Condition |
|---|---|
| `ConfigError` | Invalid parameters or idempotency key; raised before any request |
| `AuthError` | Credentials rejected (401), or organization or client not permitted (403) |
| `QuotaExceededError` | Monthly quota exhausted (402). Exposes `quota`, `usage` and `upgrade_url`. |
| `PayloadTooLargeError` | File over the plan or scanner limit (413), or over `max_bytes` |
| `UnsupportedFileError` | File type not permitted, or archive rejected by policy (415) |
| `RateLimitedError` | Concurrency limit reached (429). Exposes `retry_after`. |
| `ScanError` | Invalid request (400), content that could not be parsed (412), or an unreadable local path |
| `ServerError` | HTTP 5xx after retries |
| `NetworkError`, `RequestTimeout` | The service could not be reached or did not respond in time |
| `InfectedError` | Raised by `assert_clean`; exposes `result` |
| `KelpgateError` | Any other status, including 404 and 422 |

## Retries and idempotency

Every scan request carries an `Idempotency-Key`. The client generates one per call and reuses it for its own retries, so a retried scan is not counted twice against the quota. Token renewal is serialised, and a rejected access token is discarded and the request retried once with a new token.

Supply a key to make retries across process restarts or job re-runs idempotent:

```python
client.scan_file(path, idempotency_key=f"upload-{job.id}")
```

Keys are 1 to 128 characters from `A-Z a-z 0-9 . _ : -`. The service binds a key to the request content; reusing a key for a different file is counted as a new scan.

## FastAPI

```python
from fastapi import Depends, FastAPI, UploadFile
from kelpgate import AsyncKelpgateClient
from kelpgate.integrations.fastapi import scan_dependency

app = FastAPI()
kelpgate = AsyncKelpgateClient.from_env()

@app.post("/upload")
async def upload(file: UploadFile = Depends(scan_dependency(kelpgate))):
    ...  # reached only for files that scanned clean
```

An infected file raises `HTTPException(422)`. When no verdict is available the dependency raises `402`, `413`, `415` or `503` according to the error, unless created with `on_error="allow"`. For an upload field with another name, call `scan_upload(client, upload)` directly.

## Django

```python
# settings.py
KELPGATE = {"CLIENT_ID": "...", "CLIENT_SECRET": "..."}  # or the KELPGATE_* environment variables

# models.py
from kelpgate.integrations.django import KelpgateValidator

class Document(models.Model):
    file = models.FileField(upload_to="documents/", validators=[KelpgateValidator()])
```

An infected file fails validation with code `infected`. When no verdict is available, validation fails with code `scan_unavailable`, unless the validator is created with `on_error="allow"`.

## Command-line interface

```bash
export KELPGATE_CLIENT_ID=... KELPGATE_CLIENT_SECRET=...
kelpgate scan ./archive.zip ./report.pdf
kelpgate scan --recursive --workers 8 ./uploads
kelpgate scan-url https://example.com/file.exe
kelpgate health
```

| Exit code | Meaning |
|---|---|
| `0` | All files clean |
| `1` | At least one file infected |
| `2` | Error |

`--json` prints one JSON object per line. Recursive scans do not follow symbolic links; `--workers` defaults to 4.

## Failure policy

When the service cannot return a verdict, the client raises. The integrations convert the exception according to `on_error`:

| Condition | Client | FastAPI, `block` | Django, `block` | `allow` |
|---|---|---|---|---|
| Service unreachable, timed out, or 5xx after retries | `NetworkError`, `RequestTimeout`, `ServerError` | `503` | `scan_unavailable` | Continue |
| Authentication failed | `AuthError` | `503` | `scan_unavailable` | Continue |
| Quota exhausted | `QuotaExceededError` | `402` | `scan_unavailable` | Continue |
| File too large | `PayloadTooLargeError` | `413` | `scan_unavailable` | Continue |
| File type not permitted | `UnsupportedFileError` | `415` | `scan_unavailable` | Continue |

Use `block` where preventing malware takes priority over availability, and `allow` where uploads must continue during a service outage.

## Security

| Aspect | Behaviour |
|---|---|
| Data sent | The file content and its name, over HTTPS, to the configured `api_url` only |
| Transport | HTTPS required except for `localhost`; redirects are never followed |
| Credentials | Never included in `repr()` or exception messages |
| Tokens | Held in memory only and renewed before expiry |
| Local storage | The package writes nothing to disk |
| Service retention | Clean files are retained briefly and infected files quarantined, according to the Kelpgate plan |

Report vulnerabilities according to the [security policy](https://github.com/KelpGate/.github/blob/main/SECURITY.md).

## Troubleshooting

Verify an installation with the [EICAR test file](https://www.eicar.org/download-anti-malware-testfile/), which every scanner reports as infected:

```bash
kelpgate scan ./eicar.com    # INFECTED, exit code 1
```

| Symptom | Cause |
|---|---|
| `ConfigError: ... must use https` | `api_url` or `token_url` is not HTTPS |
| `AuthError` with code `TOKEN_REQUEST_FAILED` | Invalid client ID or secret, or the client lacks the `scan` scope |
| `AuthError` with status 403 | Organization suspended, or client not linked to an organization |
| `RateLimitedError` | Too many concurrent scans; the client honours `Retry-After`. Reduce concurrency. |
| `RequestTimeout` on large files | Increase `timeout`, or use `scan_file_async` with `wait_for_result` |
| `degraded` is `True` | An engine did not complete for that scan |

## Compatibility and versioning

| Component | Supported |
|---|---|
| Python | 3.9, 3.10, 3.11, 3.12, 3.13 (tested in CI) |
| httpx | 0.25 or later, below 1.0 |
| FastAPI | 0.100 or later (`fastapi` extra) |
| Django | 4.2 or later (`django` extra) |
| Kelpgate API | Version 1 (`/api/v1`) |

The package is fully typed (`py.typed`). It follows [Semantic Versioning](https://semver.org); fixes, including security fixes, are released for the latest version only. Changes are listed in the [changelog](CHANGELOG.md). The organisation-wide policy is in [versioning and releases](https://github.com/KelpGate/.github/blob/main/docs/versioning-and-releases.md).

## Development

```bash
pip install -e ".[dev]"
pytest
ruff check .
mypy
```

Contributions follow the [contributing guide](https://github.com/KelpGate/.github/blob/main/CONTRIBUTING.md).

## Support

See [getting help](https://github.com/KelpGate/.github/blob/main/SUPPORT.md) or contact support@kelpgate.cloud.

## Licence

MIT. See [LICENSE](LICENSE). The Kelpgate scanning service is subject to its own terms.
