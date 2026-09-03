# Developer TOKEN authentication

Source: 光鸭盘开发者 API v1.1, updated 2026-08-31.

Read this reference for actor boundaries, developer credentials, receiver UploadToken setup, request signing, and secret handling.

## Actors and credentials

- The **developer** obtains `client_id` and `client_secret` and transfers files from the developer's own Guangyapan drive.
- The **receiver** creates a TOKEN and authorizes a particular destination directory. The developer supplies that value as `token_id` only to pre-upload and upload operations.
- The developer account must be a member when creating credentials, reading its drive, and uploading.
- An account may have at most two simultaneously valid developer credentials. A blocked developer cannot authenticate or create credentials.
- A receiver account may have at most five simultaneously valid TOKENs.

## Documented setup paths

Developer credentials:

- Web: account settings in the upper-right → TOKEN management → become a developer → copy `client_id` and `client_secret`.
- PC: third-party apps in the left sidebar → TOKEN management → become a developer → copy `client_id` and `client_secret`.

Receiver TOKEN:

- App: settings → account and security → TOKEN management → create TOKEN → copy TOKEN.
- Web: account settings in the upper-right → TOKEN management → create TOKEN → copy TOKEN.
- PC: third-party apps in the left sidebar → TOKEN management → create TOKEN → copy TOKEN.

The documented production API host is:

```text
https://dapi.guangyapan.com
```

No test host is documented.

## Common request headers

All documented endpoints use `POST` with a JSON body and these headers:

| Header | Required | Value |
|---|---:|---|
| `Content-Type` | yes | `application/json` |
| `client_id` | yes | developer credential ID |
| `nonce` | yes | unique random string, 16-32 characters |
| `timestamp` | yes | current Unix timestamp in seconds |
| `sign` | yes | lowercase hexadecimal signature below |

The timestamp must differ from server time by no more than 300 seconds. A nonce is single-use; generate a new one even when retrying a request.

## Signature algorithm

Construct the UTF-8 source text in this exact order, without spaces, URL encoding, or extra fields:

```text
client_id={client_id}&client_secret={client_secret}&nonce={nonce}&timestamp={timestamp}
```

Then:

1. Compute MD5 over the UTF-8 bytes, retaining the raw 16-byte digest.
2. Compute SHA-512 over those 16 bytes.
3. Encode the SHA-512 digest as lowercase hexadecimal.

Equivalent Node.js implementation:

```js
import { createHash } from "node:crypto";

export function createDeveloperSign({ clientId, clientSecret, nonce, timestamp }) {
  const source =
    `client_id=${clientId}` +
    `&client_secret=${clientSecret}` +
    `&nonce=${nonce}` +
    `&timestamp=${timestamp}`;
  const md5Bytes = createHash("md5").update(source, "utf8").digest();
  return createHash("sha512").update(md5Bytes).digest("hex");
}
```

Equivalent Python implementation:

```python
import hashlib


def create_developer_sign(client_id, client_secret, nonce, timestamp):
    source = (
        f"client_id={client_id}"
        f"&client_secret={client_secret}"
        f"&nonce={nonce}"
        f"&timestamp={timestamp}"
    )
    md5_bytes = hashlib.md5(source.encode("utf-8")).digest()
    return hashlib.sha512(md5_bytes).hexdigest()
```

Do not call `.hexdigest()` after MD5 and then hash that text. Error `18021` indicates signature verification failure; `18022` indicates an expired timestamp; `18023` indicates nonce reuse.

## Security boundary

- Keep `client_secret` on a trusted backend or local trusted process. A browser or untrusted client should call a narrowly scoped backend, not receive the secret or a general signing oracle.
- Store receiver TOKENs as secrets because they authorize writes to a receiver-selected directory. Redact them from request logs, traces, examples, screenshots, and errors.
- Keep the task's originating developer identity with each `task_id`; status and result queries made by another developer credential return `18009`.
- A TOKEN may bind to the first successful uploader. A different uploader can then receive `18002`.
- Use developer credentials only with the endpoints documented by this API contract.

## Undocumented header in the example

The rename example sends `dt: 11`, but the common request-header contract does not define it. Preserve it only when an existing verified client requires it; otherwise omit it and seek platform confirmation if rename fails.
