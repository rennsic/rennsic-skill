---
name: rennsic-vin-history
description: Look up a vehicle's commercial rental history by VIN, using the Rennsic API. Use when the user asks whether a car or VIN was a rental or fleet vehicle, asks for a rental history or VIN background check, or is evaluating a used car whose conventional vehicle history report looks clean. Requires a Rennsic API token in RENNSIC_API_TOKEN.
---

# Rennsic VIN history

Rennsic answers one question: was this specific vehicle rented commercially on
a peer-to-peer rental platform, and how hard. Conventional vehicle history
reports do not carry that.

## Setup

- Token: read from `$RENNSIC_API_TOKEN`. If it is unset, stop and tell the user
  to get one from https://rennsic.com/api/reference/ (needs a Pro or Dealer
  subscription). Never ask the user to paste the token into the conversation.
- Base URL: `$RENNSIC_BASE_URL`, defaulting to `https://rennsic.com`.
- Auth header: `Authorization: Token $RENNSIC_API_TOKEN`. The keyword is
  `Token`, not `Bearer`. Bearer applies only to the MCP endpoint at `/api/mcp/`.

## Endpoints

VIN lookup. Costs one credit for a VIN this account has not looked up before.
Any VIN it has already paid for re-runs free, forever. A malformed VIN is
rejected with a 400 and no charge.

```bash
curl -s "${RENNSIC_BASE_URL:-https://rennsic.com}/api/vin/1HGBH41JXMN109186/" \
  -H "Authorization: Token $RENNSIC_API_TOKEN"
```

Search history, free, 25 per page:

```bash
curl -s "${RENNSIC_BASE_URL:-https://rennsic.com}/api/history/?page=1" \
  -H "Authorization: Token $RENNSIC_API_TOKEN"
```

Credit balance and request counts, free:

```bash
curl -s "${RENNSIC_BASE_URL:-https://rennsic.com}/api/credit-usage/" \
  -H "Authorization: Token $RENNSIC_API_TOKEN"
```

## Credit discipline

- Check `/api/credit-usage/` before any batch of lookups, and tell the user what
  the run will cost before spending their balance on it.
- Look up only VINs the user actually supplied or that appear in a document they
  gave you. Never loop over guessed, generated, or incremented VINs. Every new
  VIN spends a credit, and lookups are throttled at 30 per minute.
- Check `/api/history/` when you are unsure whether a VIN was already paid for.
  A VIN listed there re-runs free.
- A 402 means the balance is exhausted. Do not retry it. Tell the user to buy
  credits or upgrade their plan.
- A 429 means the rate limit is spent. Wait rather than hammering the endpoint.

## Reading the result

`found: false` is a verdict, not a failure. It means no commercial rental
history is on record for that VIN in the dataset. The credit is spent and the
answer is delivered. Do not retry the VIN, and do not report it to the user as
an error. Report it as what it is: no commercial rental history found.

`summary.total_trips` is the headline. It is built in layers, so read it in
layers:

- Every active listing of the VIN is grouped by owner. One owner listing the
  same car repeatedly is one rental history, not several.
- Each rental history counts its highest observed trip count, so a relisting
  that reset a counter to zero cannot understate the record.
- `total_trips` is the sum across those histories, and each history's own figure
  is its `trips_on_record`.

So `rental_history_count: 2` with `total_trips: 197` means two separate owners
ran the car commercially, 197 trips between them. That is a materially different
story from one owner and 197 trips, and worth saying out loud.

`summary.identity` comes from decoding the VIN against NHTSA data, not from the
listing text. It can disagree with a listing's raw make, model, or trim, because
hosts type those by hand. Prefer the decoded identity when describing the car.

`summary.usage_stats` places the trip total in context: `overall_percentile`
against the whole fleet, `make_percentile` against the same make, each with the
matching median. The block is null when there is not enough comparable data, and
`make_percentile` alone can be null for the same reason. Do not invent a ranking
when the field is null.

`cached: true` means this account had already paid for the VIN, so the call was
free. The data is still freshly aggregated, not a stale copy.

## VIN format

9 to 17 characters, letters and digits only. Modern (1981 and later) VINs are
exactly 17 characters and never contain the letters I, O, or Q, which are
routinely mistyped for 1, 0, and 0. Fix an obvious transcription error before
spending a credit; the API rejects a structurally impossible VIN for free.

## Scope

Coverage is the indexed peer-to-peer rental dataset and the dates it was
crawled. A no-match is evidence of absence within that scope, not proof the
vehicle was never rented anywhere. Say so when reporting a negative result on a
high-value decision.

Full response shapes, every field, and error codes: see `reference.md`.
