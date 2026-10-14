---
name: restriq
description: Integrate RestrIQ, the shipping-restriction API for regulated goods (firearms, ammunition, magazines, knives), into an eCommerce checkout, POS, ERP, or order system. Use whenever the user mentions RestrIQ, restriq.net, a ship/no-ship or "can this ship to" check, blocking restricted or illegal shipments at checkout, state shipping restrictions for regulated products, POST /check, POST /api/token on the RestrIQ console, or RestrIQ error codes such as invalid_credentials, invalid_request, not_found, or token_invalid_signature. Also when building a RestrIQ connector for Shopify, BigCommerce, WooCommerce, Magento, NetSuite, SAP, Epicor, Dynamics, or a POS. Also for questions about what RestrIQ's rules cover: magazine capacity limits, handgun rosters (CA, MA, MD, DC), the Illinois assault weapons ban, ammunition bans, FOID/FID or ID requirements, city and county rules, shipping to an FFL, catalog loading, or why a product came back allowed or blocked. Covers credentials and token lifecycle, the check request and response, unknown UPCs and catalog items, rule coverage, and every documented error.
license: Proprietary. Copyright © RestrIQ. All rights reserved.
---
<!-- Copyright © RestrIQ. All rights reserved. -->

# RestrIQ integration

## 1. What RestrIQ is

RestrIQ blocks illegal shipments before checkout. Send one request per cart or order: the
destination plus every UPC in it. The answer is a top-level `shippable` verdict and a
per-UPC result carrying the reasons an item cannot ship there.

Every host, field, code, and message below is documented by RestrIQ. Do not
add endpoints, headers, limits, or codes that are not on this page or in `references/`.

## 2. Hosts and credentials

| Purpose | URL |
|---|---|
| Token exchange (fixed) | `https://console.restriq.net/api/token` |
| Catalog items (fixed) | `https://console.restriq.net/api/items` |
| Check API base URL (configuration) | `https://api.restriq.net` for RestrIQ-hosted merchants; the merchant's own Check API URL when self-hosted |

- Make the Check API base URL a setting with `https://api.restriq.net` as its default.
  Never make the token endpoint configurable.
- A token is issued for one Check API host and is rejected by any other with
  `401 token_audience_invalid`. A self-hosted merchant still mints tokens from the console.
- Credentials come from `https://console.restriq.net/api-keys` → **Connect** → the
  **Developer Credentials** dialog: **Key ID** and **Secret** (each with a copy button),
  an **Issue Token** button that mints a token in the browser, and a curl one-liner.
- Store the Key ID and Secret in the platform's own secret store: Magento encrypted
  config, Shopify app secrets, or a vault-injected environment. WordPress/WooCommerce has
  no built-in secret store (`wp_options` is plain text), so a WooCommerce connector must
  encrypt the values itself or read them from the environment. Never in source, never in plugin
  settings that export in plain text, never in logs. Treat the token the same way.

## 3. Token lifecycle

Exchange the key for a bearer token with HTTP Basic auth (Key ID as user, Secret as
password) and a JSON body:

```json
{"site": "<canonical storefront URL>", "telemetry": {"platform": "...", "platform_version": "...", "connector_version": "...", "language_version": "..."}}
```

- `site` is required by this integration: the absolute `http(s)` base URL of the
  storefront, e.g. `https://northwind.example.com`. The first exchange that sends a
  `site` pins the key to that site's authority (scheme, host, non-default port). A later
  exchange from a different authority fails with `403 site_mismatch`
  (`application/problem+json`). Read `site` from configuration and send the same value
  every time; `http` vs `https` or `www` vs bare host is a different authority.
- `telemetry` is a free-form object for RestrIQ support triage; it never affects
  issuance. Send `platform`, `platform_version`, `connector_version`, `language_version`.
- Success is `200` with `Cache-Control: no-store`:
  `{"token":"<JWT>","expires_in":7775999,"token_type":"Bearer"}`. `expires_in` is in
  seconds (the observed value is about 90 days). Treat the token as opaque; do not decode
  it to schedule refresh.

Cache rules:

1. One API key and one cached token per storefront. Multiple storefronts need one key each.
2. Store the token with the time it was minted. Refresh when 80% of `expires_in` has
   elapsed.
3. On any `401` from the Check API, mint a new token once and retry the check once. If
   the retry also returns `401`, stop and surface the code.
4. Make minting single-flight: concurrent requests that find the token stale wait on one
   exchange instead of each starting their own.
5. Never mint per request.

If the exchange itself fails with `invalid_credentials`, `key_revoked`, `api_key_expired`,
or `site_mismatch`, retrying cannot fix it: surface the code to the merchant.

## 4. The check call

`POST <check base URL>/check` with `Authorization: Bearer <JWT>` and
`Content-Type: application/json`.

| Field | Rules |
|---|---|
| `destination` | Required. Exactly one of `zip` (exactly 5 digits) or `state` (2 upper-case letters, e.g. `CA`). Sending both, or neither, is `400`. |
| `upcs` | Required, at least one. Digit strings of 12, 13, or 14 digits, no separators. |
| `site` | Send it on every check: the same canonical storefront URL used for the token. Logged for support; it does not change the result. |
| `reference` | Your order or cart id, e.g. `ORDER-123`. Echoed back in the response. |
| `recipient_type` | `individual` (default), `ffl`, `law_enforcement`, `military`. |
| `effective_date` | `YYYY-MM-DD`. Evaluates time-sensitive rules as of that date; defaults to today. |
| `unknown_upc_shippable` | Boolean, default `false`. When `true`, UPCs not in the catalog come back shippable. |
| `items` | Optional catalog records for UPCs in this request. See section 6. |

UPC rules:

- A 12-digit UPC-A, or a 13- or 14-digit GTIN. A GTIN with zero padding (13 digits
  beginning `0`, 14 beginning `00`) is reduced to its UPC-A. A code with no UPC-A form is
  answered `not_found`.
- The check digit is not validated.
- Each result echoes the UPC exactly as sent, so match results back to cart lines by the
  string you sent.
- Put every UPC of the cart or order in one request.

Set `recipient_type: ffl` only when the recipient is a verified FFL; otherwise send
`individual` or omit it. What each recipient type waives depends on the rule data:
never promise a specific waiver.

## 5. Reading the response

```json
{"shippable": false, "results": [{"upc": "...", "shippable": false, "reasons": ["..."], "modifications": [{"type": "...", "required": false, "note": "..."}]}], "reference": "ORDER-123"}
```

- Gate on the top-level `shippable`. It is `false` if any result is `false`.
- `reasons` are display strings that may carry statute citations, e.g.
  `Magazine capacity >10; §265.36`. Show them to the customer or operator exactly as
  given, next to the item. Do not reword, translate, or summarise them.
- `modifications` are suggestions to make an item compliant: `type` (e.g.
  `mag_capacity_reduce`), `note` (e.g. `Replace with 10-round magazine`), and `required`
  (whether the change is required for compliance). Present them as suggestions; the item
  is still not shippable as ordered.
- `reasons: ["not_found"]` means the UPC is not in the catalog, and the item comes back
  `shippable: false`. Make an explicit, configured decision: block (the default), or send
  `unknown_upc_shippable: true` because the merchant chose to let unknown items through.
  Never treat `not_found` as allowed on your own. Then submit the product record
  (section 6) so the UPC is characterised and the next check can answer.
- `item` appears only when you sent `items` (section 6).
- `reference` is echoed when you sent one.

## 6. Unknown UPCs and the catalog

RestrIQ can only characterise items it knows about, and new UPCs appear constantly. So
whenever a check returns `not_found` for a UPC and the integration has the product
record, submit it for review. Build this into the integration rather than leaving it to
the merchant: a connector that submits what it sells gets `not_found` less and less.

Two ways to submit. Both feed the live RestrIQ catalog, so send real product data only,
never test or placeholder items.

1. **Direct**: `POST https://console.restriq.net/api/items` with the same bearer token,
   body `{"items": [...]}`, at most 200 items. Response `{"results": [{"upc", "item"}]}`.
   Synchronous: the console itself answers. Use it for product create or update hooks,
   catalog sync jobs, and anywhere delivery matters.
2. **On the check**: `items` on `/check`, in the same request, when the record is at hand
   (checkout, order import). Each item's `upc` must also be in `upcs`. Requires a bearer
   token. The Check API answers `item` itself and relays accepted items to the console in
   the background, best-effort, with your token. `accepted` means the item was valid and
   queued, not that it was delivered. On `https://api.restriq.net` the relay is on.

Self-hosted Check API: `items` on `/check` reach the catalog only if the instance was
deployed with the relay enabled (`ITEMS_API_URL` pointing at the console's `/api/items`).
Without it the instance still answers `accepted` and drops the item. Unless the operator
confirms the relay is on, submit directly to the console.

A catalog item is `{"upc": "...", "attributes": {...}}`: 1 to 20 string attributes, keys
up to 64 characters, values up to 500. Send as many attributes as the product has, not
the minimum: `brand`, `name`, `model`, and `description` are the conventional keys, and
anything that characterises the product helps review, e.g. `category`, `caliber`,
`capacity`, `blade_length`, `barrel_length`, `action`, `manufacturer`. A new entry needs
a UPC-A form and a valid GS1 check digit.

`item` status per UPC:

| `item` | Meaning |
|---|---|
| `accepted` | Entered the catalog pipeline (asynchronous, not confirmed delivery). Accepted items typically appear in restriction data within 24 hours. |
| `known` | Already cataloged; the item was ignored. |
| `invalid` | Failed content validation, or no UPC-A form, or a bad GS1 check digit; dropped. The request still succeeds. |

On every check, with or without `items`, the Check API returns `403 prefix_restricted`
when a UPC in `upcs` is outside the token's authorised prefixes (`details` carries the
UPC). With `items` and no bearer token it returns `403 token_required`. `POST /api/items`
returns the same `403 prefix_restricted` when an item's UPC is outside those prefixes.

An `accepted` item does not change the current answer: the UPC stays `not_found` until
the catalog has it. Apply the section 5 decision meanwhile.

## 7. Errors

JSON error bodies are `{"error": "...", "code": "..."}` (sometimes with `details`);
`site_mismatch` is problem+json. Branch on `code`, never on `error` text.

Token endpoint (`POST https://console.restriq.net/api/token`):

| Status | `code` | Cause and action |
|---|---|---|
| 400 | `invalid_request` | `Request body must be a JSON object`. Send a JSON object. |
| 400 | `invalid_site` | `site must be an absolute http(s) URL, e.g. https://shop.example.com`. Fix the configured site. |
| 401 | `invalid_credentials` | `Invalid API key` (wrong Key ID or Secret) or `Missing or invalid credentials` (no Basic header). Fix the stored credentials. |
| 401 | `key_revoked` | Key revoked in the console. Issue a new key. |
| 403 | `api_key_expired` | `API key has expired. Contact sales.` |
| 403 | `site_mismatch` | `application/problem+json` (RFC 9457) with `code` as a member. The key is pinned to another storefront; use that storefront's key. |
| 405 | `method_not_allowed` | Wrong method. Use `POST`. |
| 500 | `internal_error` | Server error. |

A `GET` to the token endpoint returns `303 See Other` to `/login` with an HTML body. A
client that follows redirects sees an HTML login page, not JSON: the cause is the method.

Check API (`POST <base>/check`):

| Status | `code` | Cause and action |
|---|---|---|
| 400 | `invalid_request` | One of the messages below. Fix the request; do not retry it unchanged. |
| 400 | `invalid_destination` | From source, not observed: `zip code not found in database` or `state code not found in database`. A well-formed ZIP or state that is not a known US one. Surface it to the buyer as an address problem; do not retry unchanged. |
| 401 | `unauthorized` | Re-mint once (rule 3), then: no `Authorization` header was sent. The response carries `WWW-Authenticate: Basic realm="Restricted"`; ignore it and send `Bearer <JWT>`. |
| 401 | `token_expired` | Re-mint once (rule 3), then: if it persists, the cached token is not being replaced; check the cache. |
| 401 | `token_invalid_signature` | `invalid token signature`. Re-mint once (rule 3), then: the bearer is not a token the console issued, or it was altered; surface the code. |
| 401 | `token_malformed` | Re-mint once (rule 3), then: the bearer value is not a well-formed token. Send the `token` from the exchange, never the Key ID or Secret. |
| 401 | `token_not_valid_yet` | Re-mint once (rule 3), then: the token is not valid yet; surface the code. |
| 401 | `token_claims_invalid` | Re-mint once (rule 3), then: the token's claims were rejected; surface the code. |
| 401 | `token_audience_invalid` | Re-mint once (rule 3), then: the token was issued for a different Check API host. Check the configured base URL against the key's instance. |
| 401 | `key_not_found`, `key_expired`, `key_not_yet_valid` | The API key behind the token is unusable. Re-mint once per the cache rule; if the check still fails with this code, surface it: the key needs attention in the console. |
| 401 | `token_invalid` | Any other token failure. Re-mint once (rule 3), then surface the code. |
| 403 | `prefix_restricted` | On any check, a UPC in `upcs` is outside the token's authorised prefixes; `details` carries the UPC. Check with the merchant whether the key should cover that prefix. |
| 403 | `token_required` | `items` sent without a bearer token. |
| 405 | `invalid_request` | `Method not allowed`. Use `POST`. |

The observed `400 invalid_request` messages, and one from source:

| `error` | Fix |
|---|---|
| `destination must have either zip or state, not both` | Send exactly one. |
| `destination must have either zip or state` | Add one. |
| `zip must be exactly 5 digits` | Send the 5-digit ZIP; drop ZIP+4. |
| `UPC must be 12, 13, or 14 digits` | Digits only, no separators. |
| `at least one UPC is required` | Do not call with an empty cart. |
| `recipient_type must be one of: individual, ffl, law_enforcement, military` | Map to one of those. |
| `effective_date must be in YYYY-MM-DD format` | Reformat the date. |
| `state must be exactly 2 uppercase letters` (from source) | Send the two-letter USPS code, uppercase. |

Malformed JSON is also `400 invalid_request` with `error: "Invalid JSON"` and a `details`
string. The Check API serves only `/check`: `/`, `/health`, and `/openapi.json` are `404`.

## 8. Where the check belongs

- **Checkout:** after the shipping address is known and before payment. Destination is
  the shipping ZIP; `reference` is the cart or order id. Block payment when `shippable`
  is `false` and show each blocked item's `reasons`.
- **POS:** at tender. With no shipping address, send `destination.state`.
- **ERP / OMS:** at order import or allocation. One check per order with all its UPCs,
  `reference` = the order number; store the result on the order.
- Decide explicitly, as a merchant setting, what happens when RestrIQ cannot be reached
  (block, or hold for review). Do not silently ship.

## 9. Do not

- Mint a token per request, per check, or per worker.
- Omit `site` on the token exchange or the check.
- Send both `zip` and `state`.
- Send the Key ID and Secret as the bearer token, or put them in a Basic header on `/check`.
- Treat `not_found` as allowed without an explicit, configured decision.
- Leave a `not_found` UPC unsubmitted when the product record is at hand, or submit it
  with a bare `name` when fuller attributes are available.
- Reword, shorten, or translate legal `reasons`.
- Invent rate limits, retry headers, batch endpoints, or any endpoint not listed here.
- Log the Key ID, the Secret, or a token.

## 10. Worked example

Mint a token:

```sh
curl -sS -X POST https://console.restriq.net/api/token \
  -u KEY_ID:SECRET \
  -H 'Content-Type: application/json' \
  -d '{"site":"https://northwind.example.com","telemetry":{"platform":"magento","platform_version":"2.4.7","connector_version":"1.0.0","language_version":"php 8.3.10"}}'
```

```json
{"token":"<JWT>","expires_in":7775999,"token_type":"Bearer"}
```

Check a cart shipping to New York:

```sh
curl -sS -X POST https://api.restriq.net/check \
  -H 'Authorization: Bearer <JWT>' \
  -H 'Content-Type: application/json' \
  -d '{"site":"https://northwind.example.com","reference":"ORDER-123","destination":{"zip":"10001"},"upcs":["333333333333","222222222222","200000000006","111111111111"]}'
```

```json
{"shippable":false,"results":[{"upc":"333333333333","shippable":true},{"upc":"222222222222","shippable":false,"reasons":["Magazine capacity >10; §265.36"],"modifications":[{"type":"mag_capacity_reduce","required":false,"note":"Replace with 10-round magazine"}]},{"upc":"200000000006","shippable":true},{"upc":"111111111111","shippable":true}],"reference":"ORDER-123"}
```

Block this order: top-level `shippable` is `false`. Show `Magazine capacity >10; §265.36`
against `222222222222` and offer `Replace with 10-round magazine` as a suggestion.

## 11. References

- `references/api.md`: observed request and response bodies and headers, error codes from
  source that the OpenAPI document omits, the token's JWT claims, and the `aud` contract.
  Open when reading a response or a token. For a field, type, or limit, open
  `references/openapi.json`.
- `references/integration-patterns.md`: checkout, POS, and ERP placement and token caching
  in detail. Open when designing the connector.
- `references/troubleshooting.md`: symptom-to-cause guide for each error code. Open when a
  call fails.
- `references/coverage.md`: what the rules model and do not (magazine limits by state,
  rosters, Illinois, ammunition, ID requirements, local rules, recipient types, catalog
  timing) and how to diagnose an unexpected allowed result. Open for any question about
  what RestrIQ blocks or why.
- `references/openapi.json`: the RestrIQ OpenAPI document. Open for machine-readable
  schemas or to generate types.
