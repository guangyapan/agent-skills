# Business API contract

Source: 光鸭盘开放平台 API v1.1, updated 2026-08-27.

Read this reference for signed business calls, response models, file browsing, details, downloads, VOD, user information, and thumbnails.

## Base URL and common headers

- Production: `https://openapi.guangyapan.com`
- Test: `https://openapi-test.guangyapan.com`

Every documented business call is a `GET` and requires:

| Header | Required | Meaning |
|---|---:|---|
| `Authorization` | yes | `Bearer {access_token}` |
| `client_id` | yes | assigned third-party client ID |
| `timestamp` | yes | current Unix timestamp in seconds |
| `sign` | yes | request signature below |
| `traceparent` | recommended | W3C-style trace context, if available |

These names differ from the `x-*` account API headers.

## Request signature

Build the source text in this exact order, without spaces or URL encoding:

```text
client_id={client_id}&timestamp={timestamp}&secret={sign_secret}
```

Set `sign` to the lowercase hexadecimal MD5 digest of the UTF-8 bytes. Do not hash query parameters or the access token, reorder fields, encode the source, or encode the digest again. The timestamp must be within 300 seconds of server time.

Perform signing only where `sign_secret` can remain secret. If a browser must call through a trusted backend, have that backend inject the signature and avoid turning it into a general-purpose signing oracle.

## Shared response envelope

```json
{
  "code": 0,
  "msg": "success[0]",
  "data": {}
}
```

`code === 0` means business success. A successful HTTP status with a nonzero `code` is still an API failure.

## File list

`GET /openapi/v1/file/get_file_list`

Query:

| Parameter | Type | Required | Notes |
|---|---|---:|---|
| `parentId` | string | no | parent folder ID; omit for root when supported |
| `page` | integer | no | defaults to `0` |
| `pageSize` | integer | yes | page size |
| `orderBy` | integer | no | field enum below |
| `sortType` | integer | no | sort enum below |
| `fileTypes` | integer array | no | type enum below; wire serialization is not documented |
| `resType` | integer | no | resource type enum below |

`orderBy`: `0` name, `1` size, `2` creation time, `3` update time, `4` extension.

`sortType`: `0` ascending, `1` descending.

`fileTypes`: `0` unknown, `1` image, `2` video, `3` audio, `4` document, `5` archive, `6` subtitle, `7` font, `8` installer, `9` torrent, `10` code.

`resType`: `0` unknown, `1` file, `2` folder.

Response data:

```ts
interface FileListData {
  total: number;
  list: FileInfo[];
}

interface FileInfo {
  fileId: string;
  fileName: string;
  fileSize?: number;
  bizId?: string;
  parentId?: string;
  depth: number;
  mineType?: string;
  fileType?: number;
  resType: number;
  ext?: string;
  ctime: number;
  fullParentIds?: string;
  thumbnail?: string;
}
```

Keep `fileId`, `bizId`, and parent IDs as strings. Preserve the documented wire spelling `mineType`; do not silently rename it to `mimeType` at the HTTP boundary. Fields absent in the sample for folders should be treated as optional unless the platform confirms otherwise.

## File detail

`GET /openapi/v1/file/get_file_detail?fileId={file_id}`

`fileId` is required. Response data:

```ts
interface FileDetailData {
  fileInfo: FileInfo;
  videoResource?: Array<{
    info: {
      resolution: { width: number; height: number };
      duration: number;
      bitRate: number;
      frameRate: number;
      videoCodec: string;
      audioCodec: string;
      videoType: string;
      defaultResolution?: boolean;
      resolutionName: string;
    };
    bizId: string;
  }>;
  sizeInfo?: {
    size: number;
    subDirCount: number;
    subFileCount: number;
  };
}
```

`videoResource` appears only for video files. `sizeInfo` appears only for folders. Duration is in seconds; `sizeInfo.size` and file sizes are bytes.

## Regular file download URL

`GET /openapi/v1/file/get_res_download_url?fileId={file_id}`

`fileId` is required. Response data:

```ts
interface DownloadUrlData {
  signedURL: string;
  urlDuration: number;
}
```

`urlDuration` is seconds. Treat `signedURL` as opaque and temporary.

## Video-on-demand URL

`GET /openapi/v1/file/get_vod_download_url?fileId={file_id}&bizId={biz_id}&requestId={request_id}`

| Parameter | Required | Notes |
|---|---:|---|
| `fileId` | yes | file ID |
| `bizId` | yes | video resource business ID |
| `requestId` | no | omit on first request; reuse the returned value after pause/resume or URL expiry |

Response data:

```ts
interface VodUrlData {
  signedURL: string;
  urlDuration: number;
  speedupSignature: string;
  requestId: string;
}
```

Do not create a new logical VOD task when resuming or refreshing the same one; pass its previous `requestId`.

## User information

`GET /openapi/v1/user/get_user_info`

No query parameters. Response data:

```ts
interface UserInfoData {
  userId: string;
  nickName: string;
  avatar: string;
  vipStatus: 1 | 2 | 3;
  vipLeftSeconds: number;
  totalSpace: number;
  usedSpace: number;
}
```

`vipStatus`: `1` non-VIP, `2` active VIP, `3` expired VIP. `vipLeftSeconds` is meaningful when status is `2`. Space values are bytes and documented as `int64`; in JavaScript, use a lossless strategy if values may exceed `Number.MAX_SAFE_INTEGER`.

## Thumbnails

File list or detail responses may contain a `thumbnail` URL. Use the returned URL as the base. The service may accept `w` and `h` query parameters, for example a returned `/v0/thumbnails/{biz_id}/128/128?...` URL with `&w=300&h=300`. Modify with URL APIs so existing query parameters and signatures remain intact; do not synthesize the path.

## Contract gaps to surface

- The array encoding for `fileTypes` is not documented. Match an existing verified client or confirm whether the server expects repeated parameters, comma-separated values, or another form.
- The response schema does not state nullability or exhaustive optionality.
- HTTP status mappings, download redirect behavior, and `traceparent` requirements are not specified.
