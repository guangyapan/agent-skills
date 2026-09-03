# TOKEN transfer API contract

Source: 光鸭盘开发者 API v1.1, updated 2026-08-31.

Read this reference for source-file selection, rename, pre-upload review, transfer submission, task status, and result pagination.

## Shared protocol

Base URL: `https://dapi.guangyapan.com`.

Every documented endpoint is `POST` with `Content-Type: application/json` and the signed headers in [authentication.md](authentication.md).

Responses use this envelope:

```ts
interface ApiResponse<T> {
  code: number;      // 0 means business success
  msg: string;
  data?: T | null;   // failures may omit data or return null
}
```

Check both HTTP status and `code`. A successful HTTP status with nonzero `code` is a business failure.

## Source-file operations

These `/userres/v1` operations access the developer's own drive for choosing transfer sources.

### List files

`POST /userres/v1/file/get_file_list`

| Field | Required | Notes |
|---|---:|---|
| `parentId` | no | empty string means root |
| `page` | no | zero-based |
| `pageSize` | yes | at least 1 |
| `dirType` | no | `0` unknown, `1` normal, `2` transferred, `3` favorite, `4` trash, `5` direct-link |
| `orderBy` | yes | `0` name, `1` size, `2` created, `3` updated, `4` extension |
| `sortType` | yes | `0` ascending, `1` descending |
| `fileTypes` | no | integer array; empty means all |
| `resType` | no | `0` all, `1` file, `2` directory |

Response data contains `total` and `list: FileInfo[]`.

| Field | Documented type | Meaning |
|---|---|---|
| `fileId` | string | file or directory ID |
| `fileName` | string | name |
| `fileSize` | int64 | bytes; a directory may be 0 |
| `gcid` | string | file gcid |
| `parentId` | string | parent directory ID |
| `depth` | uint | directory depth |
| `mineType` | string | MIME type; preserve this protocol spelling |
| `fileType` | int | `0` unknown, `1` image, `2` video, `3` audio, `4` document, `5` archive, `6` subtitle, `7` font, `8` installer, `9` torrent, `10` code, `11` other |
| `dirType` | int | `0` unknown, `1` normal, `4` trash, `5` direct-link |
| `resType` | int | `0` unknown, `1` file, `2` directory |
| `ext` | string | extension |
| `fullParentIds` | string | full parent path, such as `/1/2/3` |
| `ctime` | int64 | creation time in Unix seconds |
| `utime` | int64 | update time in Unix seconds |
| `thumbnail` | string | thumbnail information |
| `auditStatus` | int | `0` unknown, `1` reviewing, `2` passed, `3` rejected, `4` unknown |

The document does not define `FileInfo` nullability or field optionality. Keep the documented wire types in the contract model; if the runtime cannot represent `int64`/`uint64` losslessly, handle that in the decoder or application model without claiming the server returns strings.

### Get file detail

`POST /userres/v1/file/get_file_detail`

Body: `{ "fileId": "{file-or-directory-id}" }`; the ID length is 1-40.

Response data contains the following documented fields:

| Field | Documented type | Meaning |
|---|---|---|
| `fileInfo` | FileInfo | file information above |
| `location` | string | directory-location description |
| `picInfo` | object | image information; may be empty for non-images |
| `picInfo.width` | uint | width |
| `picInfo.height` | uint | height |
| `picInfo.previewUrl` | string | preview URL |
| `videoResource` | object[] | video resources; may be empty for non-videos |
| `videoResource[].gcid` | string | video gcid |
| `videoResource[].info` | object | video information |
| `videoResource[].info.resolution.width` | uint | width |
| `videoResource[].info.resolution.height` | uint | height |
| `videoResource[].info.duration` | uint | seconds |
| `videoResource[].info.bitRate` | uint | bit rate |
| `videoResource[].info.frameRate` | uint | frame rate |
| `videoResource[].info.videoCodec` | string | video codec |
| `videoResource[].info.audioCodec` | string | audio codec |
| `videoResource[].info.videoType` | string | video type |
| `videoResource[].info.source` | uint | `0` original, `1` transcoded, `2` transcoded plus, `3` lossless |
| `videoResource[].info.hdrType` | uint | `0` none, `1` HDR10, `2` Dolby |
| `videoResource[].info.storageType` | uint | storage type |
| `videoResource[].info.defaultResolution` | bool | whether this is the default resolution |
| `videoResource[].info.resolutionName` | string | resolution label, such as `480P` |
| `videoResource[].info.needVipType` | int | required membership type |
| `videoResource[].info.mimeType` | string | MIME type |
| `sizeInfo` | object | size information |
| `sizeInfo.size` | uint64 | total bytes |
| `sizeInfo.subDirCount` | uint | child-directory count |
| `sizeInfo.subFileCount` | uint | child-file count |

The document says `picInfo` and `videoResource` may be empty for nonmatching media, but does not define the optionality of `sizeInfo`. Preserve absent fields and unknown additions rather than manufacturing defaults.

### Rename a file or directory

`POST /userres/v1/file/rename`

```json
{
  "fileId": "{file-or-directory-id}",
  "newName": "{1-to-255-character-name}"
}
```

Rename mutates the developer's own drive. Require clear user intent before calling it. The example-only `dt: 11` header is not part of the documented common header set; see the authentication reference.

## Pre-upload review

### Submit files for review

`POST /developer/v1/pre_upload`

```json
{
  "token_id": "{receiver-upload-token}",
  "file_ids": ["{source-file-or-directory-id}"]
}
```

- `token_id` is documented as 1-20 characters for pre-upload.
- Supply 1-20 root IDs belonging to the current developer's drive.
- The parameter table calls them directory IDs, but the example and upload contract include both files and directories. Do not silently restrict them without verified behavior.
- The service may reuse a previous task for the same `token_id` and file set.

Response data:

```ts
interface PreUploadTask {
  task_id: string;
  status: 0 | 1 | 2 | 3 | 4;
  total_count: number;
  passed_count: number;
  rejected_count: number;
  pending_count: number;
}
```

Status meanings: `0` not started, `1` submitting, `2` reviewing, `3` review complete, `4` submission failed.

### Query review status

`POST /developer/v1/pre_upload_status`

Body: `{ "task_id": "{pre-upload-task-id}" }`.

The document recommends a polling interval of at least 3 seconds. Stop at status `3` or `4`; do not treat a transport retry cap as permission to resubmit the review.

### Query review detail

`POST /developer/v1/pre_upload_detail`

| Field | Required | Notes |
|---|---:|---|
| `task_id` | yes | pre-upload task ID |
| `filter` | no | `all`, `passed`, `rejected`, or `pending`; default `all` |
| `page` | no | one-based; values at or below zero act as 1 |
| `page_size` | yes | 1-100 |

Response data contains task status, task-level counts, the total for the current filter, and detail rows with `file_id`, `file_name`, `file_size`, `audit_status`, and `can_share`.

## Submit and monitor the transfer

### Upload by source IDs

`POST /developer/v1/upload_by_fileid`

```json
{
  "token_id": "{receiver-upload-token}",
  "file_ids": ["{source-file-or-directory-id}"],
  "duplicate_name_policy": 0
}
```

- Supply 1-20 source file/directory IDs belonging to the current developer.
- `duplicate_name_policy` is optional: `0` (default) automatically appends a suffix; `1` skips duplicate files/directories. The API does not overwrite the existing target.
- The service copies review-passed leaf files to the receiver TOKEN directory. If none are usable, it may return `18011`.

Successful response data contains an upload `task_id` and normally `status: "pending"`.

### Query upload status

`POST /developer/v1/upload_status`

Body: `{ "task_id": "{upload-task-id}" }`.

Status is `pending`, `running`, `success`, or `failed`. The document recommends a polling interval written as `≥ 1-3 seconds`; preserve it as a recommendation and stop at `success` or `failed`. Response data also reports:

- `use_count`: cumulative successful uses of the TOKEN.
- `success_count`: leaf files copied successfully by this task.
- `skipped_count`: leaf files skipped by this task.

## Read terminal result lists

Result lists are retained for 72 hours and should be fetched after the upload reaches a terminal state.

### Skipped files

`POST /developer/v1/upload_skipped_list`

Request fields: `task_id`, optional `reason`, optional `cursor`, and optional `page_size` from 1-1000 (default 100).

Reason filters: `audit_rejected`, `audit_expired`, `audit_pending`, `space_exhausted`, `duplicate_name`, `duplicate_dir_name`, or `other`.

Rows contain `file_id` and `reason`.

### Successful files

`POST /developer/v1/upload_success_list`

Request fields: `task_id`, optional `cursor`, and optional `page_size` from 1-1000 (default 100). Rows contain `file_id`.

For both result endpoints:

- Omit `cursor` on the first page.
- Pass a nonempty `next_cursor` back unchanged on the next request.
- An empty `next_cursor` means pagination is complete.
- `total` is optional and available only on the first page; do not use later-page absence as an error.

## Pagination and state distinctions

Do not normalize these conventions incorrectly:

- File list uses zero-based page numbers.
- Pre-upload detail uses one-based page numbers.
- Upload result lists use opaque cursors.
- Pre-upload status is an integer enum; upload status is a string enum.
- A pre-upload `task_id` and an upload `task_id` belong to different workflows and endpoints.
