---
name: pick-and-verify-an-x402-service
description: >-
  Choose one x402-payable API to call for a stated need, then verify it is up
  and that its payment terms have not drifted, before writing any payment code.
api: x402 List API
base_url: https://x402-list.com/api/v1
operations:
  - getBestServices
  - listServices
  - getService
  - getServiceChecks
  - listServiceChanges
generated: '2026-09-02'
method: generated
source: >-
  Grounded in the operationIds of https://x402-list.com/api/v1/openapi.json and
  the parameter semantics documented at https://x402-list.com/api. No credential
  is required for any step below.
---

# Pick and verify an x402 service

Use this before an agent commits to paying a third-party API. Every call here is
free and keyless up to 2,000 GET requests per IP per UTC day.

## 1. Rank the directory for the need

`getBestServices` — `GET /best`

Pass the need in `q`, and narrow with `category`, `network`, `max_price_usd`,
`require_verified` and `prefer`. This is the deterministic ranking engine, not a
language model.

    GET /best?q=web+search&network=BSE&max_price_usd=0.05&limit=5

Read `meta.ranking_version` off the response and store it beside any score you
cache. It is currently `3`; scores from an earlier generation are declared
non-comparable, so a cached score without its generation is not reusable.

If you would rather filter than rank, use `listServices` — `GET /services` —
with `status`, `category`, `network`, `verified`, `payment_ready`, `source`,
`signable`, `sort`, `page` and `per_page`.

## 2. Understand the two tier flags before you trust either

They are not synonyms and they decay differently:

- `verified` (FORTE) — x402 List paid a real x402 call and the service
  delivered. No time decay; revoked on payTo or schema drift. This tier starts
  empty and grows only with paid probes, so `verified=true` can legitimately
  return zero rows.
- `payment_ready` (BASE) — the endpoint answered a valid 402 handshake and is
  alive within the decay window (a probe success in the last 7 days). Discovery,
  not endorsement.

`signable=true` is the one to reach for before writing signing code: it means no
EVM route was observed missing the EIP-712 domain parameters (`extra.name`,
`extra.version`) a standard x402 client needs in order to sign. A service whose
latest assessment has not measured that check matches neither `true` nor
`false`, so omit the parameter if you want those included.

Every one of these filters is applied server-side, so `meta.total` counts the
filtered set — and every one returns **400 Bad Request** on an unrecognised
value rather than silently ignoring it. Read the closed sets from `GET
/networks` and `GET /categories`; full names like `base` or `solana` are not
matched, only the abbreviations (BSE, SOL, POL, ARB, BSP, AVX).

## 3. Read the full record

`getService` — `GET /services/{slug}`

Returns every active endpoint with its per-network pricing (`pay_to`, `price`
as atomic USDC, `price_usd`, `network_caip2`, `max_timeout_seconds`), the uptime
rollups, and the `assessment` block (`null` until first assessed).

Inside `assessment`, keep measured and generated apart: fields wrapped as
`AiMarkedField` carry `value`/`confidence`/`source` and are model output, while
`traction` is measured on-chain. Treat `traction` as a conservative undercount,
and if `shared_payout` is true, do **not** sum the per-day `/volume` and
`/buyers` series across the services sharing that address — those series are
address-level.

## 4. Confirm it is up right now

`getServiceChecks` — `GET /services/{slug}/checks`

Each entry carries `checked_at`, `response_time_ms`, `status_code`, `is_up`,
`error_message` and `endpoints_found`. Checks run every 5-15 minutes.

## 5. Confirm the payment terms have not moved

`listServiceChanges` — `GET /changes?service={slug}`

This is the step most integrations skip and the one that costs money. The feed
diffs the whole `accepts[]` payment envelope, not just the price, so it catches
a **payTo rotation at an unchanged price** — an address change invisible to
anyone watching prices only. Filter with `service`, `type`, `days`, `page`,
`per_page`.

If a `payTo` rotation is newer than the address you cached, re-read
`getService` and pay the new address. Never pay a cached address.

## Runtime rules that apply to every step

- **No authentication.** No key, no OAuth, no account.
- **Rate limit:** 200 req/min per IP. Above 100 req/min the response carries
  `X-RateLimit-Throttled: true` while still succeeding — back off there rather
  than waiting for the 429, which arrives with `Retry-After`.
- **Metering:** past 2,000 GET/day per IP the same endpoints return **402**, not
  429. Read `accepts[0]` from the `PaymentRequired` body (or base64-decode the
  `PAYMENT-REQUIRED` header), pay $0.01 USDC on Base, and retry the identical
  request with a `PAYMENT-SIGNATURE` header. Watch `X-Meter-Remaining` and
  `X-Meter-Reset` to see it coming.
- **Attribution:** every 2xx carries a top-level `provenance` block with the
  CC BY 4.0 licence, the exact attribution string to reproduce, and a `cite_as`
  URL. If you republish any of this data, use that string.
