# Authentication contract

Source: 光鸭盘开放平台 API v1.1, updated 2026-08-27.

Read this reference for environment selection, Device Code, Web OAuth 2.0 + PKCE, token exchange, and refresh.

## Hosts and assigned values

| Purpose | Production | Test |
|---|---|---|
| Account API | `https://openapi-account.guangyapan.com` | `https://openapi-account-test.guangyapan.com` |
| Business API | `https://openapi.guangyapan.com` | `https://openapi-test.guangyapan.com` |
| OAuth browser page | `https://www.guangyapan.com/oauth/` | Not documented |

The platform assigns `client_id`, `project_id`, and `sign_secret`. Confirm the actual environment assignment before calling a live endpoint. The supplied document does not provide a test OAuth browser-page host.

## Choose an authorization flow

- Use Device Code for PC, TV, native apps, or devices that cannot reliably complete a browser redirect.
- Use Authorization Code + PKCE for web browser flows.
- Do not invent a client-credentials flow; all documented business access follows user authorization.

## Device Code flow

### 1. Request a device code

`POST {account_base}/v1/auth/device/code`

Headers:

| Header | Required | Value |
|---|---:|---|
| `Content-Type` | yes | `application/json` |
| `x-client-id` | yes | assigned client ID |
| `x-device-id` | yes | stable device ID |
| `x-project-id` | yes | assigned project ID |

JSON body:

```json
{
  "scope": "",
  "client_id": "{client_id}",
  "project_id": "{project_id}"
}
```

`scope` is optional. The response contains:

```json
{
  "device_code": "{device_code}",
  "user_code": "{user_code}",
  "expires_in": 120,
  "interval": 3,
  "verification_url": "{authorization_page}",
  "verification_uri_complete": "{authorization_page_with_user_code}"
}
```

Use `verification_uri_complete` as returned. A native client may URL-encode it into `gyp://auth?url={verification_uri_complete}`; other clients may render it as a QR code.

### 2. Poll for tokens

`POST {account_base}/v1/auth/token`

Use the same account headers as the device-code request. JSON body:

```json
{
  "grant_type": "urn:ietf:params:oauth:grant-type:device_code",
  "device_code": "{device_code}",
  "client_id": "{client_id}"
}
```

- On `authorization_pending`, wait the returned device-code `interval` before the next request.
- Stop when authorization succeeds, `expires_in` elapses, the user cancels, or a terminal error occurs.
- Do not accelerate polling or retry indefinitely.

Successful token shape:

```json
{
  "token_type": "Bearer",
  "access_token": "{access_token}",
  "refresh_token": "{refresh_token}",
  "expires_in": 7200,
  "sub": "{user_id}"
}
```

Do not hardcode the example lifetime; honor the returned `expires_in`.

## Web OAuth 2.0 + PKCE flow

### 1. Generate PKCE and request state

- Generate a high-entropy `code_verifier` and keep it in short-lived, server-side or session-bound storage.
- Compute `code_challenge = BASE64URL(SHA256(code_verifier))` without `=` padding.
- Generate and store a separate unpredictable `state` value for callback correlation and CSRF protection when the platform accepts standard OAuth behavior.

### 2. Open the authorization page

Navigate the user to:

```text
https://www.guangyapan.com/oauth/?client_id={client_id}&response_type=code&scope={scope}&code_challenge={code_challenge}&code_challenge_method=S256&redirect_uri={redirect_uri}&state={state}
```

Required query values:

| Parameter | Value |
|---|---|
| `client_id` | assigned client ID |
| `response_type` | `code` |
| `scope` | documented fixed value `user offline` |
| `code_challenge` | generated challenge |
| `code_challenge_method` | `S256` |
| `redirect_uri` | registered callback URI |
| `state` | request-correlation value; see ambiguity below |

URL-encode parameter values, especially `scope` and `redirect_uri`.

### 3. Validate the callback and exchange the code

Expected callback:

```text
{redirect_uri}?code={code}&state={state}
```

Reject missing, mismatched, expired, or already-used state before exchanging the code.

`POST {account_base}/v1/auth/token` with JSON:

```json
{
  "client_id": "{client_id}",
  "grant_type": "authorization_code",
  "code": "{code}",
  "code_verifier": "{code_verifier}"
}
```

The response has the common token shape. The source documents the request body but does not explicitly restate the required account headers for this grant; preserve a known working implementation or confirm them with the platform.

### Documented `state` ambiguity

The overview says `state` should correlate the authorization request and callback to prevent illegal callbacks. The parameter table later says `state` is the example's `code_verifier`. Those statements conflict, and placing the verifier in the authorization URL weakens PKCE confidentiality.

- Do not silently equate the two values in a new implementation.
- Prefer independent `state` and `code_verifier` if verified against the platform.
- If compatibility requires equality, flag the security tradeoff and obtain platform confirmation before implementation.
- Never derive the verifier from untrusted callback data.

## Refresh an access token

`POST {account_base}/v1/auth/token`

Documented headers are `Content-Type: application/json`, `x-client-id`, and `x-project-id`. The refresh example omits `x-device-id`.

```json
{
  "client_id": "{client_id}",
  "grant_type": "refresh_token",
  "refresh_token": "{refresh_token}"
}
```

Successful responses return a new token shape such as:

```json
{
  "token_type": "Bearer",
  "access_token": "{access_token}",
  "refresh_token": "{refresh_token}",
  "expires_in": 1800,
  "sub": "{user_id}"
}
```

Honor returned values and replace a stored refresh token when the server rotates it. On business error `117`, perform at most one coordinated refresh-and-retry for the failed operation; if refresh fails, require fresh user authorization.

## Credential and token handling

- Send business access as `Authorization: Bearer {access_token}`.
- Avoid logging full access tokens, refresh tokens, authorization codes, verifiers, or secrets.
- Store refresh tokens in secure credential storage appropriate to the runtime.
- Refresh before or on expiry without creating parallel refresh storms; coordinate concurrent callers.
- Bind callback state and verifier to the initiating browser session and expire them after use.
