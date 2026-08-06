---
name: Import a BHPH loan portfolio file into Agora Data
description: Upload a dealer management system loan data file to Agora Data and poll the asynchronous import job to completion.
api: openapi/agora-data-openapi-original.json
generated: '2026-08-06'
method: generated
operations:
  - return_import_format_api_v1_import_get
  - file_import_api_v1_import__dms_vendor___upload_subtype__post
  - priority_upload_api_v1_import_priority__dms_vendor___upload_subtype__post
  - loan_upload_status_api_v1_uploads__api_activity_uuid__get
---

# Import a BHPH loan portfolio file into Agora Data

Base URL: `https://api.agoradata.com`

## Before you start

- You need an **API key**. Every `/api/v1/*` operation requires one; without it the API
  returns HTTP **400** with `{"detail":"Api Key is required"}`. **The header name is not
  published** in the OpenAPI or on any public page — get it from Agora Data integration
  support. Do not guess it.
- You need the `account_uuid_and_id` value for the account you are importing into. The spec
  declares it as an *optional* query parameter, but it is the tenant selector. **Always send
  it explicitly** — omitting it leaves the account context unspecified.
- You need the `dms_vendor` and `upload_subtype` path values. Both are free strings with **no
  enum in the spec**, so the accepted values are not machine-readable. Confirm them with
  Agora Data rather than inferring.

## Step 1 — Fetch the expected import format

`GET /api/v1/import` → operation `return_import_format_api_v1_import_get`

Query: `account_uuid_and_id`

This returns the field/column layout the import expects. The response is declared as
`application/json` with **no schema**, so inspect it at runtime; do not assume a shape.

## Step 2 — Upload the loan data file

`POST /api/v1/import/{dms_vendor}/{upload_subtype}` → operation
`file_import_api_v1_import__dms_vendor___upload_subtype__post`

- Path: `dms_vendor`, `upload_subtype`
- Query: `account_uuid_and_id`, `priority_upload` (boolean, default `false`)
- Header: `content-type` — the spec declares this as a **required, explicit header
  parameter** on the operation, which is unusual. Send it.
- Body: `multipart/form-data` with the file in the **`dataFile`** field
  (schema `Body_file_import_import__dms_vendor___upload_subtype__post`).

For the dedicated priority lane, use
`POST /api/v1/import/priority/{dms_vendor}/{upload_subtype}` → operation
`priority_upload_api_v1_import_priority__dms_vendor___upload_subtype__post` instead of
setting `priority_upload=true`.

Capture the `api_activity_uuid` from the response — it is the only correlation handle for
the rest of the job.

## Step 3 — Poll for completion

`GET /api/v1/uploads/{api_activity_uuid}` → operation
`loan_upload_status_api_v1_uploads__api_activity_uuid__get`

- Path: `api_activity_uuid`
- Query: `account_uuid_and_id`

There is **no push notification** for import completion and **no published polling interval
or terminal-state vocabulary**. Poll on a conservative backoff and inspect the response for
the status field at runtime.

## Retry rule — read this before retrying anything

**This API has no idempotency key.** No `Idempotency-Key` header or equivalent exists on any
write operation, including these file imports. If an upload times out you cannot safely
replay it: a retry may import the same loan portfolio twice.

Instead:

1. Do **not** blind-retry a POST that timed out.
2. If you captured an `api_activity_uuid`, poll step 3 first and let the job resolve.
3. If no `api_activity_uuid` came back, check `GET /api/v1/loans` (operation
   `get_loans_by_status_api_v1_loans_get`) for the account before re-uploading.
4. Only re-upload once you have confirmed the first attempt did not land.

## Errors

All errors use the FastAPI envelope `{"detail": ...}` as `application/json` — **not** RFC 9457
problem+json.

| Status | `detail` | Meaning |
|---|---|---|
| 422 | array of `{loc, msg, type}` | Request failed validation. `loc` points at the offending field. |
| 400 | `"Api Key is required"` | API key missing. Not declared in the spec. |
| 404 | `"Not Found"` | No route matches. Not declared in the spec. |

Note that **400 and 404 are undeclared in the OpenAPI** — a generated client will not model
them. Auth failure is returned as 400, not 401 or 403.

## Legacy path warning

An unversioned tree duplicates most of this flow:
`POST /import/{dms_vendor}/{upload_subtype}` (`file_import_import__dms_vendor___upload_subtype__post`)
and `GET /status/{api_activity_uuid}` (`get_import_status_status__api_activity_uuid__get`).
Both are live and neither is marked deprecated. **Prefer the `/api/v1` operations above.**
