---
name: Review and remove loans from an Agora Data account
description: List loans by status in an Agora Data account and safely delete loans, including the guardrails this API does not provide itself.
api: openapi/agora-data-openapi-original.json
generated: '2026-08-06'
method: generated
operations:
  - get_loans_by_status_api_v1_loans_get
  - delete_loans_api_v1_loans__rest_of_path__delete
  - loan_upload_status_api_v1_uploads__api_activity_uuid__get
---

# Review and remove loans from an Agora Data account

Base URL: `https://api.agoradata.com`

## Before you start

- An **API key** is required on every `/api/v1/*` operation; without it the API returns HTTP
  400 `{"detail":"Api Key is required"}`. The header name is unpublished — obtain it from
  Agora Data integration support.
- Send `account_uuid_and_id` explicitly on every call. It is the tenant selector and is
  declared optional; do not rely on a default account context.

## Step 1 — List the loans

`GET /api/v1/loans` → operation `get_loans_by_status_api_v1_loans_get`

Query: `account_uuid_and_id`

The operation is titled "Get Loans By Status" but the spec declares **no status parameter**
and **no response schema**, so inspect the payload at runtime to find the status field.

**There is no pagination.** No `limit`, `offset`, `cursor` or `page` parameter is declared,
and no paging envelope is documented. On a large portfolio, expect a single unbounded
response and budget memory and timeouts accordingly.

## Step 2 — Delete loans

`DELETE /api/v1/loans/{rest_of_path}` → operation
`delete_loans_api_v1_loans__rest_of_path__delete`

- Path: `rest_of_path` — an untyped **catch-all** segment, not a typed loan identifier. The
  spec does not document what belongs here, so the deletion selector cannot be constructed
  from the contract alone. Confirm the expected form with Agora Data before calling it.
- Query: `account_uuid_and_id`

## Guardrails this API does not give you

This is a **destructive** operation on consumer lending records, and the contract provides
none of the usual protections:

- **No idempotency key** — a retry after a timeout is not safe.
- **No dry-run or preview** mode.
- **No soft delete or undo** documented.
- **No typed identifier** — the catch-all path makes an over-broad selector easy to send by
  accident.
- **No confirmation semantics** — no `confirm` parameter, no two-phase delete.

Therefore, as the calling agent:

1. **Always run Step 1 first** and materialise the exact set of loans you intend to remove.
2. **Require explicit human approval** naming the count and the account before issuing any
   DELETE. Do not delete as an inferred side effect of a broader instruction.
3. **Never retry a timed-out DELETE blindly.** Re-run Step 1 and compare before acting again.
4. Scope every call with `account_uuid_and_id` so a mistake cannot cross accounts.

## Errors

Errors use the FastAPI `{"detail": ...}` envelope, not RFC 9457 problem+json.

| Status | `detail` | Meaning |
|---|---|---|
| 422 | array of `{loc, msg, type}` | Validation failure. |
| 400 | `"Api Key is required"` | Credential missing. Undeclared in the spec. |
| 404 | `"Not Found"` | No matching route. Undeclared in the spec. |

A 4xx on a DELETE does **not** reliably tell you whether the deletion partially applied —
verify with Step 1.

## Legacy path warning

`DELETE /loans/{rest_of_path}` (`delete_loans_loans__rest_of_path__delete`) is a live,
unversioned duplicate of this operation and is not marked deprecated. Prefer the `/api/v1`
form.
