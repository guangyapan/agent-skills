# Errors, limits, and production behavior

Source: 光鸭盘开放平台 API v1.1, updated 2026-08-27.

Read this reference when implementing error handling, retries, throttling, observability, or a production-readiness review.

## Business error codes

| Code | Meaning | Default handling |
|---:|---|---|
| `0` | success | continue |
| `101` | internal business error | bounded retry only when the operation is safe; otherwise surface |
| `111` | access denied | verify user/app permissions; do not retry blindly |
| `112` | invalid request parameters | fix the request; do not retry unchanged |
| `116` | invalid signature | verify exact source string, secret, timestamp units, and clock; do not retry unchanged |
| `117` | invalid access token | refresh once, retry once, then require authorization |
| `120` | invalid client ID | verify environment and `client_id`; do not retry unchanged |
| `146` | file not found | treat as missing or stale ID |
| `149` | file deleted | treat as terminal |
| `164` | resolution requires VIP | offer an allowed resolution or explain membership requirement |
| `167` | file cannot be consumed | treat as terminal; file failed review |
| `430` | user is not a member | check VIP state and gate the capability |

Inspect HTTP status and parse the business envelope defensively. Preserve `code`, `msg`, trace context, endpoint, and safe request metadata in structured errors, but redact tokens, secrets, signed URLs, authorization codes, and personal data.

## Per-IP rate limits

| Endpoint | Limit |
|---|---:|
| `/openapi/v1/file/get_file_list` | 5 requests/second/IP |
| `/openapi/v1/file/get_file_detail` | 2 requests/second/IP |
| `/openapi/v1/file/get_res_download_url` | 2 requests/second/IP |
| `/openapi/v1/file/get_vod_download_url` | 2 requests/second/IP |
| `/openapi/v1/user/get_user_info` | 2 requests/second/IP |

Centralize throttling per outbound IP and endpoint because many users can share the same backend egress IP. Leave headroom for concurrent processes rather than scheduling exactly at the documented ceiling.

## Retry policy

- Retry only transient transport failures, throttling, and documented internal error `101` when the request is safe.
- Use exponential backoff with jitter and a small attempt cap. Honor server retry guidance if one is returned, although no retry header is specified by v1.1.
- Never retry `112`, `116`, or `120` unchanged.
- For `117`, coordinate one token refresh across concurrent callers, replay the business request once, and stop if it still fails.
- Device Code polling is not a generic retry loop: use its `interval` and stop at device-code expiry.
- Re-fetch an expired temporary URL through its corresponding API. For VOD, reuse the prior `requestId`.

## Observability

- Generate a valid `traceparent` when the application already supports distributed tracing. A typical form is `00-{32 hex trace id}-{16 hex parent id}-01`.
- Record endpoint, latency, HTTP status, business code, retry count, and a redacted trace identifier.
- Do not record full authorization headers, tokens, secrets, signed URLs, PKCE values, or raw response bodies that may contain personal data.
- Detect clock drift because a timestamp difference greater than 300 seconds invalidates signatures.

## Production checklist

- Environment hosts and assigned `client_id`/`project_id` match.
- OAuth callback URI is registered, exact, and validated.
- PKCE verifier and state are session-bound, single-use, and expire promptly.
- Secrets and refresh tokens are stored outside browser/public bundles.
- Token expiry uses returned `expires_in`; concurrent refresh is coordinated.
- Signature tests assert the exact source string and Unix-seconds timestamp.
- File IDs and documented `int64` values remain lossless.
- `fileTypes` serialization has been verified with the platform.
- All endpoint rate limits are enforced per egress IP.
- Signed download and thumbnail URLs are treated as opaque temporary values.
- Logs and fixtures contain no real credentials or user data.

## Version record

| Version | Date | Change |
|---|---|---|
| v1.0 | 2026-05-30 | Initial documented Open API set |
| v1.1 | 2026-08-27 | Added total and used space to `get_user_info` |
