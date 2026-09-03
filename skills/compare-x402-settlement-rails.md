---
name: compare-x402-settlement-rails
description: >-
  Compare x402 facilitators by measured on-chain settlement volume rather than
  self-reported claims, and trace which listed services actually settle through
  each one.
api: x402 List API
base_url: https://x402-list.com/api/v1
operations:
  - listFacilitators
  - getFacilitator
  - getStats
  - getNetworks
generated: '2026-09-02'
method: generated
source: >-
  Grounded in the operationIds and Facilitator* component schemas of
  https://x402-list.com/api/v1/openapi.json and the measurement caveats
  documented at https://x402-list.com/methodology and in
  https://x402-list.com/llms.txt.
---

# Compare x402 settlement rails

Use this when the question is about the **rail** rather than the service — which
facilitator actually settles money, on which chains, for whom.

## 1. Rank the facilitators

`listFacilitators` — `GET /facilitators`

Every facilitator carries `volume_usd_24h/7d/30d/all`, `tx_count_*` over the
same windows, `settler_count`, `verification`, `first_activity_at` /
`last_activity_at`, a `chains[]` breakdown and a `timeseries[]`.

The volume is read directly from each facilitator's own settler addresses
on-chain. It is **never self-reported**.

## 2. Open one

`getFacilitator` — `GET /facilitators/{id}`

Adds `settlers[]` (the actual on-chain addresses the volume is read from, with
network, CAIP-2 id, token name and decimals, and whether each is enabled),
`buyers`, `avg_settlement_usd_30d`, `trend_7d_vs_prev_7d`, a daily `series[]`
(`period=all` for full history), and `listed_services` — the bridge back to the
directory services that settle through this facilitator.

Toggle the heavier parts with `include_chains` and `include_timeseries`.

## 3. Read the caveats the response hands you

`FacilitatorDetail` carries its own `method` and `caveat` strings. Honour them:

- **USDC only, and a conservative undercount.** Only settlements the monitor
  observed are counted. Never present a figure as an ecosystem total.
- **Per-chain distinct buyers are an upper bound** — they are not de-duplicated
  across chains, so do not sum them.
- Volume is a record of settlement observed on the wire. It is **not** an
  endorsement or a quality ranking of a facilitator.

## 4. Frame it against the directory

`getStats` — `GET /stats` gives directory-wide aggregates (service and endpoint
counts, network counts, average uptime and response time, checks per hour, and
the price distribution: average, min, max, median).

`getNetworks` — `GET /networks` gives the closed network set with both the
abbreviation and the CAIP-2 id (`eip155:8453` for Base), `chain_type`,
`is_mainnet`, `explorer_url`, `service_count` and `avg_uptime`. Testnets are
present and flagged by `is_mainnet`, so filter before quoting a network count.

## Runtime rules

Free and keyless up to 2,000 GET/day per IP; 200 req/min. Past the daily quota
these endpoints return **402**, payable at $0.01/request in USDC on Base with a
`PAYMENT-SIGNATURE` header. Every 2xx carries the CC BY 4.0 `provenance` block —
reproduce its `attribution` string if you republish any of these figures.
