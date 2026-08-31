---
name: rennsic-vin-history
description: Look up a vehicle's commercial rental history by VIN, using the Rennsic API. Use when the user asks whether a car or VIN was a rental or fleet vehicle, asks for a rental history or VIN background check, or is evaluating a used car whose conventional vehicle history report looks clean. Requires a Rennsic API token in RENNSIC_API_TOKEN.
---

# Rennsic VIN history

Rennsic answers one question: was this specific vehicle rented commercially on
a peer-to-peer rental platform, and how hard. Conventional vehicle history
reports do not carry that.

## Prefer the MCP server when it is connected

If this session has the Rennsic MCP server connected (tools named `vin_lookup`,
`search_history`, `credit_usage`, `request_report`), use those tools instead of
the curl recipes below: they carry the same gates and answer with the same
payloads, and the client handles auth. The server also provides two prompts a
user can invoke by name, `check_vin` and `account_status`. The Claude Code
plugin bundles this server, so it is normally already registered; if its tools
are absent, the user likely declined the connection, and the recipes below are
the fallback. This skill's shell recipes are for scripts, CI, and environments
without MCP. The reading rules in this file apply either way; the payloads are
identical.

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

Request the branded PDF report for a VIN this account has already looked up,
free:

```bash
curl -s -X POST "${RENNSIC_BASE_URL:-https://rennsic.com}/api/vin/1HGBH41JXMN109186/report/" \
  -H "Authorization: Token $RENNSIC_API_TOKEN"
```

Download the PDF once its status is `ready`, free:

```bash
curl -s "${RENNSIC_BASE_URL:-https://rennsic.com}/api/vin/1HGBH41JXMN109186/report/download/" \
  -H "Authorization: Token $RENNSIC_API_TOKEN" -o report.pdf
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

## PDF reports

The report is the same branded PDF the web console produces, a second rendering
of an answer already paid for, so requesting and downloading it never costs a
credit. The flow is asynchronous: POST the request, poll until the `report`
block's status is `ready` or `failed`, then fetch `download_url`. Poll by
re-POSTing or by re-running the free lookup; repeat requests dedupe onto the
generation already running, so polling is safe. A 404 means the account never
paid for the VIN or it aged out of the plan's retention window; run the lookup
first. A 409 means the VIN has no listings on record, so there is nothing to
build a report from. Both are verdicts, not transient errors: do not retry
them.

## Reading the result

`found: false` is a verdict, not a failure. It means no commercial rental
history is on record for that VIN in the dataset. The credit is spent and the
answer is delivered. Do not retry the VIN, and do not report it to the user as
an error. Report it as what it is: no commercial rental history found. The
response still carries the decoded `identity` at the top level, so read the
year, make, and model back to the user; if it names a different car than they
meant, the VIN was mistyped.

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
against the whole fleet, `make_percentile` against the same make, and
`model_percentile` against the same make and model, each with the matching
median. The model figure is the sharpest comparison, so lead with it when it is
present. The block is null when there is not enough comparable data, and
`make_percentile` or `model_percentile` can each be null on their own for the
same reason. Do not invent a ranking when a field is null.

`review_risk` carries renter-reported condition evidence when Rennsic has
indexed reviews for the VIN: a 0-100 score, a coverage note naming what it
rests on, and up to four quoted findings, worst first. It is null for almost
every VIN, and null means no review data is indexed, never that no problems
were reported. Never present a null as a clean bill of health.

`record_url` is the VIN's record page on the web console. Hand it to the user
for the full visual record; it needs their signed-in browser session, so do not
try to fetch it with the API token.

`report` is the account's PDF report state for the VIN (see PDF reports above).
Its `download_url`, unlike `record_url`, works with the API token.

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
