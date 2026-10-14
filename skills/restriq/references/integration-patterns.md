<!-- Copyright © RestrIQ. All rights reserved. -->

# RestrIQ integration patterns

How to place the check, cache the token, and build a connector. Field rules and error
codes are in `SKILL.md`; observed responses are in `references/api.md`.

## Token cache

One cached token per storefront, stored with the time it was minted. Every check goes
through this algorithm:

1. **Load** the cached entry for this storefront: `token`, `expires_in`, and the time it
   was minted.
2. **Check age.** If there is no entry, or the time elapsed since minting is at least
   0.8 × `expires_in`, refresh (step 3). Otherwise go to step 4.
3. **Refresh, single-flight.** Exactly one caller runs the exchange
   (`POST https://console.restriq.net/api/token` with Basic `KEY_ID:SECRET` and the
   `site` / `telemetry` body). Every other caller that found the token stale waits for
   that exchange and uses its result. Store the new `token`, its `expires_in`, and the
   time it was minted. If the exchange fails with `invalid_credentials`, `key_revoked`,
   `api_key_expired`, or `site_mismatch`, stop: retrying cannot fix it; surface the code
   to the merchant.
4. **Check.** `POST <check base URL>/check` with `Authorization: Bearer <JWT>`.
5. **On a `401` from the Check API**, mint a new token once (through step 3, still
   single-flight) and retry the check once.
6. **If the retry also returns `401`, stop** and surface the `code`. Do not loop, and do
   not mint again for this check.

Where the cache lives:

- Put it where every worker, process, and node of the storefront can read it (the
  platform's cache or database), so the storefront holds one token, not one per worker.
- Treat the cached token like the Secret: encrypted at rest where the platform allows,
  never logged, never shown in an admin page.
- Never mint per request, per check, or per worker. A token is valid for the whole of
  `expires_in` (observed: 7775999 seconds).

## Where the check belongs

One request per cart or order, every UPC in it. Match each result back to its line by
the UPC string you sent. Gate on the top-level `shippable`.

Decide, as a merchant setting, what happens when RestrIQ cannot be reached: block, or
hold for review. Never ship silently.

### Checkout

After the shipping address is known, before payment. Destination is the shipping ZIP
(5 digits; drop ZIP+4). `reference` is the cart or order id.

```json
{"site":"https://northwind.example.com","reference":"ORDER-123","destination":{"zip":"10001"},"upcs":["333333333333","222222222222"]}
```

When `shippable` is `false`, block payment and show each blocked item's `reasons`
exactly as returned, next to that item. Offer `modifications` as suggestions.

### POS

At tender. There is usually no shipping address, so send the state.

```json
{"site":"https://northwind.example.com","reference":"ORDER-124","destination":{"state":"NY"},"upcs":["111111111111","222222222222"]}
```

Show the `reasons` to the operator on the register. A `recipient_type` other than
`individual` is for a verified recipient only (for `ffl`, a verified FFL).

### ERP / OMS

At order import or allocation. One check per order with all its UPCs; `reference` is
the order number. Store the result (top-level `shippable` and each line's `reasons`) on
the order so the decision can be audited.

```json
{"site":"https://northwind.example.com","reference":"ORDER-125","destination":{"zip":"90210"},"upcs":["222222222222","200000000005","111111111111"]}
```

A held or re-released order is checked again before it ships: rules are evaluated as of
`effective_date`, which defaults to today.

## Building a connector

### One key and one token per store

- A key is pinned to the first `site` it is exchanged with: one key serves one
  storefront. A platform install with several storefronts (e.g. Northwind Traders,
  Fabrikam, and Contoso) holds one Key ID / Secret pair and one cached token per
  storefront.
- A staging or test storefront on a different host is a different authority and needs
  its own key.

### `site`

- `site` is the store's canonical URL: the absolute `http(s)` base URL customers use,
  e.g. `https://northwind.example.com`.
- Read it from configuration and send the identical value on every token exchange and
  every check. `http` vs `https`, `www` vs bare host, or a non-default port is a
  different authority, and the exchange fails with `403 site_mismatch`.
- Do not derive it from the incoming request's `Host` header, which changes with proxies,
  admin hosts, and alternate domains.

### `telemetry`

Send these keys on every exchange. The values are free-form strings for RestrIQ support
and never affect issuance:

```json
{"platform":"magento","platform_version":"2.4.7","connector_version":"1.0.0","language_version":"php 8.3.10"}
```

| Key | Value |
|---|---|
| `platform` | The commerce, POS, or ERP platform. |
| `platform_version` | That platform's version. |
| `connector_version` | Your connector's version. |
| `language_version` | Runtime and version, e.g. `php 8.3.10`. |

### Settings

- **Check API base URL**: a setting, default `https://api.restriq.net`; a self-hosted
  merchant enters their self-hosted Check API base URL. A token is valid only on the
  host it was minted for (`aud`), so clear the cached token when this setting changes.
- **Catalog submissions**: fixed at `https://console.restriq.net/api/items` for every
  deployment, self-hosted included. Not derived from the Check API base URL.
- **Token endpoint**: fixed at `https://console.restriq.net/api/token`. Not a setting.
- **Unknown UPCs**: block (default) or let through (`unknown_upc_shippable: true`).
- **RestrIQ unreachable**: block, or hold for review.

### Secret storage

Store the Key ID and Secret in the platform's own secret store: Magento encrypted
config, Shopify app secrets, or a vault-injected environment. WordPress/WooCommerce has
no built-in secret store (`wp_options` is plain text), so a WooCommerce connector must
encrypt the values itself or read them from the environment.
Never in source, never in plugin settings that export in plain text, never in logs.
Treat the token the same way.

## Unknown UPCs

A UPC not in the catalog comes back `shippable: false` with `reasons: ["not_found"]`.

1. **Is the result `not_found`?** If not, act on `shippable` as usual.
2. **What did the merchant configure?**
   - **Block** (the default): the item is not shippable. Tell the customer or operator
     it cannot be verified.
   - **Let unknown items through**: send `unknown_upc_shippable: true` on the check. Unknown
     UPCs then come back `shippable: true` with no reasons. Never make this choice on the
     merchant's behalf.
3. **Submit the product record.** RestrIQ can only characterise items it knows about, and
   new UPCs appear constantly, so a `not_found` with a product record at hand is a
   submission, not a dead end. Send every attribute the product has (up to 20), not just
   a name:
   - **`POST https://console.restriq.net/api/items`**, for catalog sync outside a
     check (product create or update, a nightly job) and whenever delivery matters. Body
     `{"items": [...]}`, at most 200 per request, same bearer token. Response
     `{"results": [{"upc", "item"}]}`, answered by the console itself.
   - **`items` on `/check`**, when the record is at hand during the check (checkout, order
     import). Same request, same bearer token; each item's `upc` must also be in `upcs`.
     The result gains `item`: `accepted`, `known`, or `invalid`. The Check API relays
     accepted items to the console in the background, best-effort; `accepted` is not a
     delivery receipt. `api.restriq.net` relays; a self-hosted Check API relays only if
     the operator enabled it (`ITEMS_API_URL`), otherwise post directly to the console.
   - Each item is `CatalogItem`: `upc` plus 1 to 20 string `attributes`, conventionally
     `brand`, `name`, `model`, `description`, plus whatever characterises the product
     (`category`, `caliber`, `capacity`, `blade_length`, and so on). A new entry needs a
     UPC-A form and a valid GS1 check digit, or it comes back `invalid`.
4. **Wait for the pipeline.** `accepted` is asynchronous: accepted items typically appear
   in restriction data within 24 hours. Until then the UPC stays `not_found`; keep
   applying step 2.

Both paths feed the live RestrIQ catalog. Send real product data only, never test,
sample, or placeholder items, and never from a test suite.
