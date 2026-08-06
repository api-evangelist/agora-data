---
name: Authenticate to Agora Data and query a third-party data provider
description: Obtain an Agora Data access token via OAuth or password login, then list and query the third-party data providers reachable through the passthrough surface.
api: openapi/agora-data-openapi-original.json
generated: '2026-08-06'
method: generated
operations:
  - create_a_code_oauth_authorize_post
  - rotate_tokens_oauth_token_post
  - login_login_authenticate_post
  - template_providers_get
  - agora_endpoint_providers__provider_name___rest_of_path__get
---

# Authenticate to Agora Data and query a third-party data provider

Base URL: `https://api.agoradata.com`

The `/providers` surface uses a **different credential** from the `/api/v1/*` loan
operations. Those need an API key; these need an `Authorization` header. Calling `/providers`
without one returns HTTP **400** with `{"detail":"Authorization header is required"}`.

**The OpenAPI declares no `securitySchemes` and no `security` on any operation.** Everything
below is reconstructed from the auth operations the spec exposes and from the live error
responses. See `authentication/agora-data-authentication.yml` for the evidence.

## Option A — OAuth authorization code

### Step 1 — Request an authorization code

`POST /oauth/authorize` → operation `create_a_code_oauth_authorize_post`

Body (`application/json`, schema `ClientIdBody`):

```json
{ "client_id": "<your client id>" }
```

Returns an `AuthCodeResponse`: `{ "code": "...", "redirect_url": "..." }`.

### Step 2 — Exchange and rotate tokens

`POST /oauth/token` → operation `rotate_tokens_oauth_token_post`

The spec declares **no request body** for this operation and only a 200 response, so the
exchange parameters are undocumented. The response is a `TokenRequestResponse`:

| Field | Required |
|---|---|
| `access_token` | yes |
| `refresh_token` | yes |
| `expires_in` | yes |
| `token_type` | no |
| `scope` | no |

A `scope` value comes back, but **Agora Data publishes no scope vocabulary** for this API —
no flow `scopes` map in the spec and no permissions reference page. Read the returned scope
at runtime; do not assume any scope string.

## Option B — Username and password

`POST /login/authenticate` → operation `login_login_authenticate_post`

Body (`application/json`, schema `UsernamePassword`), both fields required:

```json
{ "email": "<email>", "password": "<password>" }
```

Returns an `AccessTokenResponse`: `{ "access_token": "..." }`.

This is a direct resource-owner credential exchange. Prefer Option A where you can; never
persist end-user credentials to complete this flow on their behalf without explicit consent.

> AgoraPortal (the dealer console at `portal.agoradata.com`) is a separate system that
> authenticates against an Auth0 OIDC tenant — `https://agora-data.us.auth0.com/`, authorization
> code with PKCE S256, audience `dealer-portal`. Its discovery document is at
> `well-known/agora-data-openid-configuration.json`. Portal tokens are **not** the API tokens
> described above.

## Step 3 — List available providers

`GET /providers` → operation `template_providers_get`

- Header: `Authorization` (required in practice, undeclared in the spec)
- Query: `account_uuid_and_id`

## Step 4 — Query a specific provider

`GET /providers/{provider_name}/{rest_of_path}` → operation
`agora_endpoint_providers__provider_name___rest_of_path__get`

- Path: `provider_name`, `rest_of_path`
- Query: `account_uuid_and_id`

This is a **passthrough proxy** — `rest_of_path` is an untyped catch-all forwarded to the
named upstream provider. Consequences for an agent:

- The response shape is entirely determined by the upstream provider and is **not modelled
  anywhere in Agora's spec**. Parse defensively.
- The set of valid `provider_name` values is not enumerated. Discover them from Step 3
  rather than guessing.
- Because the subpath is forwarded verbatim, treat any value you place in `rest_of_path` as
  reaching a third-party system. Do not interpolate untrusted input into it.

## Errors

| Status | `detail` | Meaning |
|---|---|---|
| 400 | `"Authorization header is required"` | No `Authorization` header on a `/providers` call. |
| 422 | array of `{loc, msg, type}` | Validation failure. |
| 404 | `"Not Found"` | No matching route. |

Errors use the FastAPI `{"detail": ...}` envelope, not RFC 9457 problem+json, and auth
failures come back as **400 rather than 401/403** — do not branch on 401 to trigger a token
refresh. Refresh on `expires_in` instead, using the `refresh_token` from Step 2.
