# Rennsic API reference

Base URL `https://rennsic.com`, overridable with `RENNSIC_BASE_URL`.
Every request carries `Authorization: Token $RENNSIC_API_TOKEN`.

## GET /api/vin/{vin}/

One credit for a VIN this account has not looked up before. Free for any VIN it
already paid for.

A miss is a complete, successful response. `summary` and `listings` are simply
absent:

```json
{ "found": false, "cached": false, "vin": "1HGBH41JXMN109186" }
```

A hit:

```json
{
  "found": true,
  "cached": false,
  "vin": "1HGBH41JXMN109186",
  "summary": {
    "total_trips": 197,
    "listing_count": 3,
    "rental_history_count": 2,
    "identity": { "year": 2022, "make": "Tesla", "model": "Model 3", "trim": null },
    "usage_stats": {
      "overall_percentile": 91,
      "fleet_median_trips": 9.0,
      "make_percentile": 88,
      "make_median_trips": 12.0
    }
  },
  "listings": [
    {
      "trips_on_record": 142,
      "current": {
        "id": 1234567,
        "vin": "1HGBH41JXMN109186",
        "year": 2022,
        "make": "Tesla",
        "model": "Model 3",
        "trim": null,
        "type": "car",
        "transmission": "automatic",
        "tripCount": 142,
        "numberOfRentals": 140,
        "numberOfReviews": 118,
        "numberOfFavorites": 22,
        "url": "https://...",
        "listing_created_time": "2022-04-18T00:00:00Z",
        "location": { "city": "Miami", "state": "FL", "country": "US", "display": "Miami, FL" },
        "registration": { "state": "FL" },
        "main_image_url": "https://...",
        "other_image_urls": ["https://..."]
      },
      "history": [
        {
          "id": 1234566,
          "listing_created_time": "2021-08-02T00:00:00Z",
          "tripCount": 96,
          "numberOfReviews": 74,
          "url": "https://..."
        }
      ]
    }
  ]
}
```

### Top level

| Field | Notes |
| --- | --- |
| `found` | False when no listing for the VIN is on record. Still a paid, valid answer. |
| `cached` | True when this account had already paid for the VIN, so no credit was charged. The data is re-aggregated on every call, not stored and replayed. |
| `vin` | The normalized (upper-cased, trimmed) VIN that was searched. |
| `summary` | Present only when `found` is true. |
| `listings` | Present only when `found` is true. One entry per rental history, ordered by trips descending. |

### summary

| Field | Notes |
| --- | --- |
| `total_trips` | The headline. Sum across rental histories of each history's highest observed trip count. |
| `listing_count` | Individual listing records behind the answer. Higher than `rental_history_count` when an owner relisted the same car. |
| `rental_history_count` | Distinct owners who listed this VIN. Two histories mean two separate commercial operators. |
| `identity` | `year`, `make`, `model`, `trim`, decoded from the VIN against NHTSA data. Any field can be null when the decode is incomplete. Authoritative over the raw listing text. |
| `usage_stats` | Percentile context, or null when there is not enough comparable data to rank fairly. |

### summary.usage_stats

| Field | Notes |
| --- | --- |
| `overall_percentile` | Rank of `total_trips` against the whole fleet, 0 to 100. |
| `fleet_median_trips` | Fleet median trip count, for context on how far above typical this car sits. |
| `make_percentile` | Rank against the same make. Null when that make's sample is too small to rank reliably. |
| `make_median_trips` | Median trip count for the make. Null under the same condition. |

### listings[]

| Field | Notes |
| --- | --- |
| `trips_on_record` | This history's authoritative count: the maximum observed across its snapshots. Use this, not `current.tripCount`, which can be lower after a relisting reset the counter. |
| `current` | The most recent listing snapshot for this owner. |
| `history` | Older snapshots of the same rental history, newest first. Empty when only one snapshot exists. |

`current` fields: `id`, `vin`, `year`, `make`, `model`, `trim`, `type`,
`transmission` (`automatic` or `manual`), `tripCount`, `numberOfRentals`,
`numberOfReviews`, `numberOfFavorites`, `url`, `listing_created_time`,
`location` (`city`, `state`, `country`, `display`), `registration` (`state`),
`main_image_url`, `other_image_urls`. The raw `make`, `model`, and `trim` here
are host-entered text and may disagree with `summary.identity`.

`history[]` entries are compact: `id`, `listing_created_time`, `tripCount`,
`numberOfReviews`, `url`.

## GET /api/history/

Free. Every lookup on the account, newest first, 25 per page. `page` selects the
page; `page_size` raises it up to 100.

```json
{
  "count": 42,
  "next": "https://rennsic.com/api/history/?page=2",
  "previous": null,
  "results": [
    {
      "vin": "1HGBH41JXMN109186",
      "success": true,
      "cached": false,
      "credits_deducted": true,
      "searched_at": "2026-07-29T04:32:09Z"
    }
  ]
}
```

`success` records whether that lookup found a match. `credits_deducted` marks
the call that paid for the VIN; `cached` is its inverse. A VIN appearing here at
all can be re-looked-up free.

## GET /api/credit-usage/

Free.

```json
{ "credits_remaining": 81, "requests_this_month": 12, "requests_total": 340 }
```

`requests_this_month` and `requests_total` count API calls, not credits spent.
Free re-runs are counted too.

## Errors

| Status | Meaning | What to do |
| --- | --- | --- |
| 400 | The VIN is structurally impossible (wrong length, illegal characters, or I/O/Q in a 17-character VIN). No credit charged. | Fix the VIN. Check for I/O/Q mistyped for 1/0/0. |
| 401 | Missing or invalid token. | Tell the user to check `RENNSIC_API_TOKEN` against the API console. Do not retry. |
| 402 | Out of credits. | Tell the user to buy credits or upgrade. Do not retry. |
| 403 | No active Pro or Dealer subscription, or the calling IP is not on the account's whitelist. | Relay the message in the response body. Do not retry. |
| 429 | Rate limited. VIN lookups allow 30 per minute. | Wait, then resume. Do not parallelize around it. |

Error bodies carry a plain-language `error` (or `detail`) string. Relay it
rather than paraphrasing it into something vaguer.

## Caveats

Host identity, precise addresses, and license plate data are deliberately absent
from API responses. Location is city, state, and country only. Do not tell a
user this data exists behind another parameter; it does not.

Coverage is the indexed peer-to-peer rental dataset and the dates it was
crawled. A `found: false` is evidence of absence within that scope. It is not
proof the vehicle was never rented, never listed on another platform, or never
used commercially off platform. State that limit plainly when a negative result is load-bearing for a
purchase decision.
