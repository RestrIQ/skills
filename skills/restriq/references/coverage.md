<!-- Copyright © RestrIQ. All rights reserved. -->

# RestrIQ coverage: what the rules model

Open this for questions about RestrIQ's rules rather than its protocol: whether a law is
modeled, why a product came back allowed or blocked, and how recipients, rosters, and the
catalog affect the answer. These statements are RestrIQ's own. Where a question is not answered here
(a specific product, a specific local ordinance, a state-by-state exemption), do not
guess: tell the user to send RestrIQ the UPCs and destinations behind it.

## First, rule out an unknown product

Most "this should have been blocked" reports are a UPC RestrIQ does not have yet, checked
with `unknown_upc_shippable: true`, which makes unknown UPCs come back shippable. Before
concluding that a rule is missing:

1. Re-run the check without `unknown_upc_shippable` (default `false`). A result of
   `reasons: ["not_found"]` means the product is not in the catalog. That is the cause.
2. Submit the product record with every attribute you have (`SKILL.md` section 6). A new
   product is usually checkable the next morning.
3. If the product is known and the answer still looks wrong, send RestrIQ the UPC,
   destination, and request.

## Magazines

Magazine capacity is recorded for every product in the catalog.

| Limit | States |
|---|---|
| More than 10 rounds blocked | CA, NY, NJ, MA, CT, DC, MD, RI, WA |
| More than 15 rounds blocked | CO, VA |
| More than 17 rounds blocked | DE |
| Separate handgun and long-gun limits | IL, VT, HI |

- Capacity cannot be sent on a check to change the answer. A check answers from the
  catalog only. When submitting a product the catalog does not have, include `capacity`:
  it is used to classify the product.
- A standalone magazine that comes back allowed in one of these states is almost always a
  UPC the catalog does not have, checked with `unknown_upc_shippable: true` (see above).

## Handguns and their included magazine

- A handgun is classified by the capacity of the magazine it ships with. A handgun sold
  with a 17-round magazine is blocked wherever a 17-round magazine is.
- Where a compliant configuration exists, the result carries a modification, e.g.
  `mag_capacity_reduce` with the note `Replace with 10-round magazine`.
- In CA, MA, and MD the handgun must also be on the state roster (next section).

## Handgun rosters (CA, MA, MD, DC)

- The CA, MA, and MD rosters are refreshed from the official sources every night.
- DC's roster is the union of those three: a handgun on any of them can ship to DC.
- A rostered handgun is allowed. An off-roster handgun is blocked, with the code section
  cited in `reasons` (e.g. `Handguns must be on CA roster; Penal Code §32000`).
- Whether a shipment to a dealer is exempt from the roster depends on the state. This
  reference does not say which. Send RestrIQ the state and they confirm it. Do not assume
  `recipient_type: "ffl"` lifts it: a CA roster block was observed unchanged with `ffl`.

## Illinois: Protect Illinois Communities Act (2023)

Modeled (720 ILCS 5/24-1.9). It covers both the listed firearms and the feature test:
pistol grip, thumbhole stock, folding or telescoping stock, flash hider, threaded barrel,
barrel shroud, forward grip, grenade launcher, and buffer tube. Magazine limits under
§24-1.10 are 10 rounds for long guns and 15 for handguns.

## Ammunition

- All ammunition is blocked to CA, NY, MA, and DC, regardless of caliber. One caliber
  allowed while another is blocked is not expected: the allowed one is almost always a
  UPC the catalog does not have (see the first section).
- Ammunition to NY, MA, and DC is blocked outright, not conditionally.

## ID and age-check requirements

Not returned today. Every result is shippable or blocked with a reason; there is no
"ships only with ID or age check" result.

- The ID requirements for IL (FOID), NJ (FID), CT, and RI are on file but are not
  returned as a separate result yet.
- MD, DE, and CO ID requirements are not covered yet.
- A merchant who must collect a FOID, FID, or similar needs their own step for it.
  RestrIQ's answer does not tell them when.

## City and county rules

Local rules for jurisdictions such as NYC and Chicago were reviewed by attorneys,
who found the state rules already cover them. For a specific local rule
(e.g. Denver, Cook County), send it to RestrIQ to pass on for review. Do not state that a
particular local ordinance is or is not modeled.

## Recipient types (shipping to an FFL)

- Send `"recipient_type": "ffl"` on the check. The values are `individual` (default),
  `ffl`, `law_enforcement`, and `military`.
- Each restriction defines which recipient types are exempt from it. An FFL recipient
  lifts some restrictions but not all of them. Never promise a specific waiver.
- Send a type other than `individual` only for a verified recipient of that type.

## Loading a catalog and how fast changes show up

- Submit products to `POST https://console.restriq.net/api/items` with the bearer token,
  at most 200 per request. Send everything you have: `brand`, `name`, `model`,
  `caliber`, `capacity`, `category`, `action`, and so on. Products can also go on a check
  in `items` (`SKILL.md` section 6).
- A new product is usually checkable the next morning. Rule updates publish on the same
  daily schedule.
- For a large initial load (tens of thousands of products), agree a timeline with
  RestrIQ before sending it.
