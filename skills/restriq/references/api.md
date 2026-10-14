<!-- Copyright © RestrIQ. All rights reserved. -->

# RestrIQ API: what `openapi.json` does not say

Field types, enums, patterns, and limits live in `references/openapi.json`. Read these
schemas there instead of copying them:

| Schema | Used by |
|---|---|
| `TokenRequest`, `TokenResponse` | `POST https://console.restriq.net/api/token` |
| `CheckRequest`, `CheckResponse`, `UPCResult`, `Modification` | `POST <check base URL>/check` |
| `CatalogItem` | `items` on `/check`, and `POST https://console.restriq.net/api/items` |
| `ErrorResponse` | Every JSON error: `{"error", "code", "details"?}` |
| `ProblemDetails` | `403 site_mismatch` only (`application/problem+json`, RFC 9457) |

This file adds what was observed against production on 2026-11-05, the codes the
services return that the OpenAPI document does not list, and the contracts behind the
token. Credentials and tokens below are `KEY_ID`, `SECRET`, and `<JWT>`.

## Credentials

- **Key ID**: 11 characters, base64url, no prefix.
- **Secret**: 43 characters, base64url, no prefix.
- Both come from `https://console.restriq.net/api-keys` → **Connect**. They are used only
  as HTTP Basic credentials on the token endpoint, never on `/check`.

## Token endpoint: observed responses

`POST https://console.restriq.net/api/token`, `-u KEY_ID:SECRET`.

| Request | Status | Body |
|---|---|---|
| No body | 200 | `{"token":"<JWT>","expires_in":7775999,"token_type":"Bearer"}` |
| Body `{}`, `Content-Type: application/json` | 200 | same |
| Body with `telemetry` | 200 | same |
| Wrong Secret (`-u KEY_ID:wrong`) | 401 | `{"error":"Invalid API key","code":"invalid_credentials"}` |
| Wrong Key ID (`-u wrong:SECRET`) | 401 | `{"error":"Invalid API key","code":"invalid_credentials"}` |
| No `Authorization` header | 401 | `{"error":"Missing or invalid credentials","code":"invalid_credentials"}` |
| Body `{bad` | 400 | `{"error":"Request body must be a JSON object","code":"invalid_request"}` |
| Body `{"site":"not a url"}` | 400 | `{"error":"site must be an absolute http(s) URL, e.g. https://shop.example.com","code":"invalid_site"}` |
| `GET` | 303 | `<a href="/login">See Other</a>.` (`text/html`) |

Headers of note: a `200` carries `Cache-Control: no-store` and
`Content-Type: application/json`. The `303` points at `/login`; a client that follows it
receives an HTML login page.

`TokenResponse` in the OpenAPI document shows `expires_in: 86400` as its example. The
observed value is `7775999` (about 90 days). Read `expires_in` from every response; never
hard-code either number.

Documented, not observed (exercising them would revoke, expire, or pin the key used for these captures):

| Status | `code` | Body |
|---|---|---|
| 401 | `key_revoked` | `ErrorResponse` |
| 403 | `api_key_expired` | `ErrorResponse`, `error` = `API key has expired. Contact sales.` |
| 403 | `site_mismatch` | `ProblemDetails`, `application/problem+json`, `code` as an extension member |
| 405 | `method_not_allowed` | `ErrorResponse` (from source) |
| 500 | `internal_error` | `ErrorResponse` (from source) |

`site` pins the key to the site's authority (scheme, host, non-default port) on the first
exchange that sends one. The OpenAPI document says omitting `site` "is currently allowed
for older callers"; this integration always sends it.

## The token

The token is a JWT. Observed header: `alg` `ES256`, `kid`, `typ` `JWT`. Observed claims:

| Claim | Observed |
|---|---|
| `iss` | `https://console.restriq.net` |
| `sub` | present |
| `aud` | array of one URL |
| `exp`, `iat` | `exp - iat` = 7,776,000 s (90 days) |
| `jti` | present |

These are listed so a log or a support ticket can be read. A caller treats the token as
opaque: do not decode it, do not validate it, and do not schedule refresh from `exp`.
`expires_in` from the exchange is the refresh signal.

### The `aud` contract

`aud` is the Check API authority the key is provisioned for. RestrIQ-hosted keys get
`https://api.restriq.net`. A self-hosted merchant runs the Check API in their own
infrastructure, still mints tokens at `https://console.restriq.net/api/token`, and gets
`aud` set to their instance. The Check API rejects a token whose `aud` does not match the
host it was sent to, with `401 token_audience_invalid`.

So: the token endpoint is fixed, the Check API base URL is configuration
(`https://api.restriq.net`, or your self-hosted Check API base URL), and a token minted
for one instance is not valid on another.

## Check API: observed responses

`POST https://api.restriq.net/check`, `Authorization: Bearer <JWT>`,
`Content-Type: application/json`. Shapes: `CheckRequest` and `CheckResponse`.

NY, one item blocked, `reference` echoed:

```json
{"site":"https://shop.example.com","reference":"ORDER-123","destination":{"zip":"10001"},"upcs":["333333333333","222222222222","200000000006","111111111111"]}
```
```json
{"shippable":false,"results":[{"upc":"333333333333","shippable":true},{"upc":"222222222222","shippable":false,"reasons":["Magazine capacity >10; §265.36"],"modifications":[{"type":"mag_capacity_reduce","required":false,"note":"Replace with 10-round magazine"}]},{"upc":"200000000006","shippable":true},{"upc":"111111111111","shippable":true}],"reference":"ORDER-123"}
```

CA by `state` (same result with `recipient_type` omitted and `"ffl"`):

```json
{"destination":{"state":"CA"},"upcs":["222222222222","200000000005","200000000008","111111111111"]}
```
```json
{"shippable":false,"results":[{"upc":"222222222222","shippable":false,"reasons":["Magazine ban reinstated; Duncan v. Bonta (9th Cir.)"],"modifications":[{"type":"mag_capacity_reduce","required":false,"note":"Replace with 10-round magazine"}]},{"upc":"200000000005","shippable":false,"reasons":["Handguns must be on CA roster; Penal Code §32000"]},{"upc":"200000000008","shippable":false,"reasons":["Suppressors prohibited; Penal Code §33410"]},{"upc":"111111111111","shippable":true}]}
```

Unknown UPCs, default, then with `"unknown_upc_shippable":true`:

```json
{"destination":{"zip":"75001"},"upcs":["000000000001","999999999999"]}
```
```json
{"shippable":false,"results":[{"upc":"000000000001","shippable":false,"reasons":["not_found"]},{"upc":"999999999999","shippable":false,"reasons":["not_found"]}]}
```
```json
{"shippable":true,"results":[{"upc":"000000000001","shippable":true},{"upc":"999999999999","shippable":true}]}
```

`effective_date` changes the answer: `222222222222` to ZIP `90210` is blocked today and
shippable on `2020-04-01`:

```json
{"shippable":true,"results":[{"upc":"222222222222","shippable":true}]}
```

GTIN-13 and GTIN-14 with zero padding, echoed exactly as sent:

```json
{"shippable":true,"results":[{"upc":"0111111111111","shippable":true},{"upc":"00222222222222","shippable":true}]}
```

In every observed response a shippable result has no `reasons` or `modifications` keys at
all, and `reference` is absent from the response when none was sent.

## Check API: observed errors

Validation, all `400`, `application/json`, `code` `invalid_request`:

| Request | Body |
|---|---|
| `"destination":{"zip":"75001","state":"TX"}` | `{"error":"destination must have either zip or state, not both","code":"invalid_request"}` |
| `"destination":{}` | `{"error":"destination must have either zip or state","code":"invalid_request"}` |
| `"zip":"7500"` | `{"error":"zip must be exactly 5 digits","code":"invalid_request"}` |
| `"upcs":["1111-1111"]` | `{"error":"UPC must be 12, 13, or 14 digits","code":"invalid_request"}` |
| `"upcs":[]` | `{"error":"at least one UPC is required","code":"invalid_request"}` |
| `"recipient_type":"dealer"` | `{"error":"recipient_type must be one of: individual, ffl, law_enforcement, military","code":"invalid_request"}` |
| `"effective_date":"01/15/2024"` | `{"error":"effective_date must be in YYYY-MM-DD format","code":"invalid_request"}` |
| Body `{nope` | `{"error":"Invalid JSON","code":"invalid_request","details":"invalid character 'n' looking for beginning of object key string"}` |
| `GET /check` | `405` `{"error":"Method not allowed","code":"invalid_request"}` |
| `GET /`, `/health`, `/openapi.json` | `404` `404 page not found` (`text/plain`) |

Authentication, `401`:

| Request | Headers of note | Body |
|---|---|---|
| No `Authorization` | `WWW-Authenticate: Basic realm="Restricted"` | `{"error":"Unauthorized","code":"unauthorized"}` |
| Bearer with a bad signature | | `{"code":"token_invalid_signature","error":"invalid token signature"}` |

Both carry `Access-Control-Allow-Origin: https://console.restriq.net` and `Vary: Origin`.
The `WWW-Authenticate: Basic` header does not mean the Check API takes the API key: send
`Bearer <JWT>`. Key order in error bodies varies; parse JSON, never match the raw text.

No rate-limit or retry headers were observed on any response.

## Codes from source, not in the OpenAPI document

Check API bearer failures are all `401` with `{"error", "code"}` and no
`WWW-Authenticate` header:

`token_expired`, `token_invalid_signature`, `token_malformed`, `token_not_valid_yet`,
`token_claims_invalid`, `token_audience_invalid`, `key_not_found`, `key_expired`,
`key_not_yet_valid`, and `token_invalid` for any other token failure.

Check API, `400` and `403`:

- `invalid_destination`: a well-formed ZIP or state not in RestrIQ's database;
  `error` is `zip code not found in database` or `state code not found in database`.
- `invalid_request` also has `state must be exactly 2 uppercase letters`.
- `prefix_restricted`: on any check, before `items` are considered, a UPC in `upcs` is
  outside the token's authorised prefixes. Body:
  `{"error":"UPC not within authorized prefixes","code":"prefix_restricted","details":"<the UPC>"}`.
- `token_required`: `items` were sent without a bearer token.

`POST https://console.restriq.net/api/items` returns the same `403 prefix_restricted`
(`{"error":"UPC not within authorized prefixes","code":"prefix_restricted"}`, no `details`).

## Catalog item delivery

- `POST /api/items` on the console is synchronous: the console validates the token,
  checks each item, and answers `results` itself.
- `items` on `/check` are judged by the Check API (`known`, `invalid`, `accepted`), and
  accepted items are forwarded to the console in the background, with your bearer token.
  Delivery is best-effort and the caller is never told whether it happened.
- Forwarding is enabled per Check API instance by `ITEMS_API_URL`. An instance without it
  still answers `accepted` and discards the items. RestrIQ-hosted `api.restriq.net` has
  it on; a self-hosted instance has it only if its operator set it.
- Consequence: `accepted` on `/check` is not a delivery receipt. For confirmed
  submission use the direct endpoint.

The OpenAPI `ErrorResponse.code` list includes `invalid_upc` and `invalid_destination`.
Production returns `invalid_request` for a malformed UPC (observed) and no handler emits
`invalid_upc` (source), so do not branch on it.
