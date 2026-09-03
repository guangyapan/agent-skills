---
name: guangyapan-open-api
description: >
  Integrate and troubleshoot the 光鸭盘 (Guangyapan) third-party Open API.
  Use for Device Code or Web OAuth 2.0 + PKCE authorization, token refresh,
  signed user/file requests, temporary download URLs, error codes, and rate
  limits. Do not use for unrelated internal 光鸭盘 APIs or generic OAuth work.
---

# Integrate Guangyapan Open API

Use the Open API v1.1 contract dated 2026-08-27 as the source of truth for third-party integrations. Preserve its wire names and host boundaries exactly; do not silently normalize fields or fill in undocumented behavior.

## Route the task

- For choosing and implementing Device Code or Web OAuth + PKCE, token exchange, and refresh, read [references/authentication.md](references/authentication.md).
- For request signing and the user, file list, file detail, download, VOD, and thumbnail contracts, read [references/business-api.md](references/business-api.md).
- For response handling, errors, rate limits, retries, and production readiness, read [references/operations.md](references/operations.md).
- Read multiple references only when the task spans those areas.

## Workflow

1. Identify the target environment, runtime, authorization flow, and requested business capability. Inspect existing repository abstractions before adding a client.
2. Keep account authorization, browser authorization, and business API hosts separate. Select the environment consistently.
3. Implement credentials, PKCE state, token storage, signing, and retries on the appropriate trusted boundary for the runtime.
4. Model documented request and response fields explicitly. Keep ID and `int64`-like values lossless, and preserve API spellings such as `mineType`.
5. Handle both HTTP failures and the business response envelope. Apply endpoint-specific rate limits and bounded retry behavior.
6. Test deterministic signing, PKCE generation, callback validation, token refresh, query serialization, and representative business/error responses without embedding real credentials.

## Non-negotiable constraints

- Never put `sign_secret`, refresh tokens, or permanent credentials in browser bundles, public environment variables, fixtures, logs, examples, or source control.
- Generate the business signature from the exact text `client_id={client_id}&timestamp={timestamp}&secret={sign_secret}`, then lowercase hexadecimal MD5. The Unix timestamp is in seconds and must be within 300 seconds of the server.
- Account endpoints use `x-client-id`, `x-project-id`, and sometimes `x-device-id`; signed business endpoints use `client_id`, `timestamp`, `sign`, and `Authorization`. Do not substitute one header convention for the other.
- Treat `signedURL` and thumbnails as temporary server-returned URLs. Do not reconstruct them or cache them beyond their returned lifetime.
- Do not make live authorization or business requests unless the user has supplied the environment and credentials and authorized the external action. Use placeholders in generated code and tests by default.
- The source document conflicts on whether OAuth `state` equals `code_verifier`. Follow the ambiguity procedure in the authentication reference instead of exposing or guessing the verifier.

## Source limits

The contract does not define every HTTP status, token error payload, query-array encoding, refresh-token rotation rule, or retry header. Preserve an existing verified behavior when present; otherwise expose the uncertainty and request platform confirmation rather than inventing a protocol.
