<!-- Copyright © RestrIQ. All rights reserved. -->

# RestrIQ troubleshooting

Find the symptom, then branch on `code`, never on the `error` text. "Rule 3" is the
cache rule in `SKILL.md`: on a `401` from the Check API, mint a new token once and retry
the check once; if the retry also returns `401`, stop and surface the code. Observed
bodies are in `references/api.md`.

## Token endpoint: `POST https://console.restriq.net/api/token`

| Symptom | Cause | Fix |
|---|---|---|
| `400 invalid_request`, `Request body must be a JSON object` | The body is not a JSON object (malformed JSON, an array, a string). | Send a JSON object with `Content-Type: application/json`. |
| `400 invalid_site`, `site must be an absolute http(s) URL, e.g. https://shop.example.com` | The configured `site` is not an absolute `http(s)` URL. | Fix the setting to the store's canonical URL, e.g. `https://northwind.example.com`. |
| `401 invalid_credentials`, `Invalid API key` | Wrong Key ID or wrong Secret. | Fix the stored credentials. Check whether they were copied whole from **Connect** on the API Keys page, and whether the Key ID and Secret were swapped. Do not retry. |
| `401 invalid_credentials`, `Missing or invalid credentials` | No Basic `Authorization` header reached the endpoint. | Send Basic auth: Key ID as user, Secret as password. Check whether the stored values are empty. |
| `401 key_revoked` | The key was revoked in the console. | Issue a new key in the console and store it. Do not retry. |
| `403 api_key_expired`, `API key has expired. Contact sales.` | The key has expired. | Surface the message to the merchant; they contact RestrIQ sales. Do not retry. |
| `403 site_mismatch` (`application/problem+json`) | The key is pinned to another storefront's authority (scheme, host, non-default port). | Use that storefront's key, or a new key for this one. Check whether `site` changed between `http`/`https`, `www`/bare host, or a port. Do not retry. |
| `405 method_not_allowed` | Wrong method. | Use `POST`. |
| `500 internal_error` | Server error. | Surface it. Keep the current token while it is still within its lifetime. |
| HTML instead of JSON (a login page) | A `GET` was sent: the endpoint answers `303 See Other` to `/login`, and a client that follows redirects shows the login page. | Send `POST`. Check whether the HTTP client defaults to `GET` when no body is set. |

## Check API: `POST <check base URL>/check`

| Symptom | Cause | Fix |
|---|---|---|
| `400 invalid_request` | The request failed validation; `error` names the field (the observed messages, and one from source, in `SKILL.md` section 7). | Fix the request. Do not retry it unchanged. |
| `400 invalid_request`, `Invalid JSON` with `details` | The body is not valid JSON. | Serialize the body with a JSON library, not string concatenation. |
| `400 invalid_destination`, `zip code not found in database` or `state code not found in database` (from source, not observed) | The ZIP or state is well-formed but not a known US ZIP or state in RestrIQ's database. | Surface it to the buyer as an address problem and ask for a corrected shipping address. Do not retry unchanged. |
| `401 unauthorized` | No `Authorization` header was sent. | Re-mint once (rule 3), then: send `Authorization: Bearer <JWT>`. Check whether a proxy or HTTP client strips the header. |
| `401` with `WWW-Authenticate: Basic realm="Restricted"` | No bearer was sent (this is the `unauthorized` response). | Send `Bearer <JWT>`. The header is not an invitation to use Basic: never put the Key ID and Secret on `/check`. |
| `401 token_expired` | The token's lifetime has passed. | Re-mint once (rule 3), then: if it persists, the cached token is not being replaced; check the cache stores the mint time and refreshes at 0.8 × `expires_in`. |
| `401 token_invalid_signature`, `invalid token signature` | The bearer is not a token the console issued, or it was altered. | Re-mint once (rule 3), then: surface the code. Check whether the bearer is the `token` from the exchange, sent unmodified. |
| `401 token_malformed` | The bearer value is not a well-formed token. | Re-mint once (rule 3), then: send the `token` from the exchange, never the Key ID or Secret. Check for truncation or a doubled `Bearer ` prefix. |
| `401 token_not_valid_yet` | The token is not valid yet. | Re-mint once (rule 3), then: surface the code. |
| `401 token_claims_invalid` | The token's claims were rejected. | Re-mint once (rule 3), then: surface the code. |
| `401 token_audience_invalid` | The token was issued for a different Check API host (`aud`). | Re-mint once (rule 3), then: check the configured base URL against the key's instance. |
| `401 key_not_found` | The API key behind the token is unusable. | Re-mint once (rule 3); if the check still fails with this code, surface it: the key needs attention in the console. |
| `401 key_expired` | The API key behind the token is unusable. | Re-mint once (rule 3); if the check still fails with this code, surface it: the key needs attention in the console. |
| `401 key_not_yet_valid` | The API key behind the token is unusable. | Re-mint once (rule 3); if the check still fails with this code, surface it: the key needs attention in the console. |
| `401 token_invalid` | Any other token failure. | Re-mint once (rule 3), then surface the code. |
| `403 prefix_restricted` | On any check, with or without `items`, a UPC in `upcs` is outside the token's authorised prefixes; `details` carries the UPC. | Check with the merchant whether the key should cover that prefix. |
| `403 token_required` | `items` were sent without a bearer token. | Send `Authorization: Bearer <JWT>` with `items`. |
| `405 invalid_request`, `Method not allowed` | Wrong method. | Use `POST`. |
| `404 page not found` (`text/plain`) | A path other than `/check` (`/`, `/health`, `/openapi.json`). | Call `<base>/check`. Check whether the base URL setting already ends in `/check`. |
| Token works on one host and not another | A token is valid only for the Check API host in its `aud`; others answer `401 token_audience_invalid`. | Check the configured base URL against the key's instance: `https://api.restriq.net` for RestrIQ-hosted keys, or your self-hosted Check API base URL. Use the key provisioned for the host you call, and clear the cached token when the base URL setting changes. |
| `not_found` on every UPC | The UPCs are not in the RestrIQ catalog. | Check whether the connector sends real UPCs and not SKUs or internal ids, and whether they have a UPC-A form. Submit real product records (`items` or `POST /api/items`) with every attribute you have; accepted items typically appear within 24 hours. Meanwhile apply the configured unknown-UPC decision. |
| A result does not match its cart line | Results are matched by the UPC string as sent. | Look the result up by the exact string you sent (`0111111111111` comes back as `0111111111111`). |

## Catalog items: `POST https://console.restriq.net/api/items`

| Symptom | Cause | Fix |
|---|---|---|
| `403 prefix_restricted`, `UPC not within authorized prefixes` | An item's UPC is outside the token's authorised prefixes (no `details`). | Check with the merchant whether the key should cover that prefix. |
| `item: "invalid"` | Failed content validation, no UPC-A form, or a bad GS1 check digit; the item was dropped. | Fix the record: 1 to 20 string attributes, a UPC-A form, a valid check digit. |
| `item: "accepted"` but the check still says `not_found` | The catalog pipeline is asynchronous. | Wait; accepted items typically appear in restriction data within 24 hours. |
| `item: "accepted"` from `items` on `/check`, never appears, especially self-hosted | The Check API relays `/check` items to the console in the background only when deployed with `ITEMS_API_URL`; without it, items are accepted and dropped. `accepted` is not a delivery receipt. | Submit directly: `POST https://console.restriq.net/api/items` with the same token. Self-hosted operators can enable the relay on their instance. |
