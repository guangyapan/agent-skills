# Errors and production behavior

Source: 光鸭盘开发者 API v1.1, updated 2026-08-31.

Read this reference for error classification, polling, duplicate submissions, task ownership, retries, observability, and production readiness.

## Documented usage requirements

- Comply with applicable law, the platform user agreement, and content restrictions.
- Operate only within the receiver-authorized directory and TOKEN scope. Do not access, scrape, or retain unauthorized data.
- Follow platform frequency and retry requirements; do not flood or forge requests.
- The developer is responsible for uploaded content and API activity. The platform may rate-limit, revoke, or trace a violating TOKEN.

## Error codes

HTTP-layer guidance in the contract: authentication failures are commonly `401`, throttling is `429`, and unsupported endpoints are `404`. Exact payloads are not specified.

| Code | Documented meaning |
|---:|---|
| `0` | success |
| `18001` | UploadToken missing or deleted |
| `18002` | caller is not the bound uploader |
| `18003` | uploader and receiver are the same account |
| `18006` | file does not belong to uploader |
| `18007` | receiver space is insufficient |
| `18008` | destination directory is missing |
| `18009` | no permission to query the upload task |
| `18010` | request too frequent; retry later |
| `18011` | no share-approved files available to upload |
| `18012` | file count exceeds the limit |
| `18013` | server busy; retry later |
| `18014` | same files already uploaded; do not repeat the operation |
| `18015` | leaf-file count for one upload exceeds the limit |
| `18016` | upload task ended and details cannot be appended |
| `18020` | developer credential invalid or deleted |
| `18021` | signature verification failed |
| `18022` | signature expired |
| `18023` | nonce already used |
| `18024` | valid developer credential count reached the limit |
| `18025` | endpoint is not in the developer allowlist |
| `18026` | developer is blocked |

## Timing and duplicate submission

- The same upload parameters cannot be submitted again within one minute.
- While the earlier task is unfinished and within ten minutes, do not submit the same upload again.
- After `pre_upload`, the document recommends calling upload after 15 minutes and within six hours. It permits upload while review status is `2` (reviewing) or `3` (complete); the upload copies only passing targets.
- The document recommends polling pre-upload status at intervals of at least three seconds.
- The document recommends an upload-status polling interval written as `≥ 1-3 seconds`; it does not define a single exact minimum.
- Success and skipped result lists remain available for 72 hours after terminal completion.

Keep these as separate timers. Do not interpret a polling interval as permission to resubmit a mutating endpoint.

## Implementation retry guidance

The source lists error meanings and requires reasonable retry behavior but does not define a complete retry algorithm, attempt cap, or retry header. The following is implementation guidance, not protocol:

- Generate a new nonce, timestamp, and signature for every retry.
- Automatically retry only read-only status, detail, or result-list requests for transient transport failures, HTTP `429`, `18010`, or `18013`, using exponential backoff, jitter, and a small attempt cap.
- Do not blindly retry `pre_upload`, `upload_by_fileid`, or rename after an ambiguous timeout. If a task ID was received, query that task. If no task ID was received, surface the ambiguity and respect the duplicate window before any user-authorized resubmission.
- Do not retry credential, permission, ownership, signature, invalid-parameter, or terminal business failures unchanged.
- Stop polling on documented terminal states, task-ownership failure, user cancellation, or a caller-defined deadline. Return the last task state so the caller can resume safely.

## Task ownership and TOKEN behavior

- Store the originating developer identity with each review and upload task. A task can only be queried by the corresponding developer account.
- A TOKEN may bind to the first uploader after its first successful use; subsequent calls from another uploader can fail with `18002`.
- The uploader and receiver cannot be the same account.
- Scope operations to the receiver-authorized destination directory and do not assume any additional receiver-account permission.
- `use_count` is cumulative for the TOKEN; `success_count` and `skipped_count` are scoped to the current upload task.

## Observability and redaction

Record endpoint, latency, HTTP status, business code, retry count, safe task identifier, and a redacted developer identifier. Avoid raw request/response logging when it may contain file names or account data.

Never log:

- `client_secret`
- complete receiver TOKEN or `token_id`
- reusable request headers or signatures
- file names or personal data unless the user's logging policy explicitly permits them

Detect clock drift and nonce reuse separately because they require different fixes.

## Production checklist

- Calls use `dapi.guangyapan.com`, `POST`, JSON bodies, and the documented signed headers.
- Developer credentials and receiver TOKENs remain on a trusted boundary and are redacted.
- Signature tests verify SHA-512 over raw MD5 bytes, exact field order, Unix seconds, and lowercase hex.
- Nonces are cryptographically random, 16-32 characters, unique per request, and regenerated for retries.
- Source IDs belong to the authenticated developer; task IDs remain associated with that developer.
- The review and upload state machines use their distinct enum types and terminal states.
- All three pagination conventions are handled without off-by-one or cursor reconstruction.
- Mandatory duplicate-submission windows and 72-hour result retention are handled, while documented polling and review timing remain recommendations.
- Rename and transfer submission require explicit user intent; ambiguous network outcomes are surfaced rather than repeated blindly.
- No undocumented test host, mandatory `dt` header, rate limit, or retry header has been invented.
