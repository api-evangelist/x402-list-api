---
name: list-a-service-in-the-x402-directory
description: >-
  Submit an x402 service or facilitator to the x402 List directory, or correct
  an existing listing through the domain-proof owner-update flow — including
  what it costs, what cannot be changed, and what cannot be taken back.
api: x402 List API
base_url: https://x402-list.com/api/v1
operations:
  - submitService
  - requestServiceUpdate
  - verifyServiceOwnership
  - reissueOwnershipToken
generated: '2026-09-02'
method: generated
source: >-
  Grounded in the operationIds and response codes of
  https://x402-list.com/api/v1/openapi.json and the submission and owner-update
  prose at https://x402-list.com/api.
---

# List or correct a service in the x402 directory

This is the write half of the API. **Read the reversibility section first** —
nothing here can be undone through the API.

## Submitting

`submitService` — `POST /submit`

Omit `type` (or set anything other than `facilitator`) for a service; set
`"type": "facilitator"` for a settlement operator. Service submissions are
automatically probed for valid HTTP 402 responses on the endpoints you give; a
failed or idle probe does not block the submission. Facilitator submissions are
checked for recent on-chain USDC settlement from the declared settler addresses
instead. **Every submission of either type is reviewed by a human before it
appears.**

The 201 body carries `submission_id`, `status: pending`, and a `probe_result`
(`null` if the probe timed out).

### What it costs

Free from your own domain. The endpoint answers **402** in three cases:

| Case | Price |
|---|---|
| Service URL on a free compute host (vercel.app, workers.dev, pages.dev, netlify.app, onrender.com, railway.app, fly.dev, herokuapp.com, replit.app, glitch.me, deno.dev, pythonanywhere.com, firebaseapp.com, web.app…) | $1.00 |
| Resubmitting one rejected less than 14 days ago | $0.50 |
| Both at once | $1.50, as a single payment |

All in USDC on Base. Read `accepts[0].amount` and `resource.description` from the
`PaymentRequired` body, pay it, and retry with a `PAYMENT-SIGNATURE` header. The
app-level code for the second case is `resubmission_fee_required`.

The free-host fee is **non-refundable and never expires**: it buys a place in the
human review queue, not a listing. After 14 days a resubmission is free again.

Static-only hosting (github.io, gitlab.io, surge.sh) and dev tunnels (ngrok,
trycloudflare.com, localhost.run, serveo.net) are rejected with **400** at any
price.

### Cooldown

7 days per submitter email, tracked separately per submission type — a pending
service submission does not block a facilitator submission from the same
address. Exceeding it returns **429** with `Retry-After`.

## Correcting a listing you own

Three operations, in order.

### 1. `requestServiceUpdate` — `POST /services/{slug}/request-update`

Requires `email` plus at least one changed field. Changeable: `name`,
`description`, `website_url`, `category`, `contact_email`, `endpoints_add` (each
optionally prefixed with its HTTP method), and `base_url` (an identity change,
reviewed with extra scrutiny).

**Measured fields are read-only for everyone**: `verified`, `uptime`, `pricing`,
`status` and `payTo` cannot be changed through this channel by anybody,
including the owner. Fields are compared server-side and only real changes are
recorded — submitting with no effective change returns **400**.

The response carries a **one-time ownership token, returned once and never
stored in clear**. Capture it on the first read.

Cooldown: one request per (email, service) every 7 days, **429** with
`Retry-After`. Only a live or approved request holds the window, so a rejected
one can be corrected and resent immediately.

### 2. Publish the proof

Write the token as a line of a plain-text file at
`{base_url origin}/.well-known/x402list.txt` **on the currently listed domain**.
The token expires after **72 hours**.

### 3. `verifyServiceOwnership` — `POST /services/{slug}/verify-ownership`

Returns **410 Gone** if the token has expired — call `reissueOwnershipToken`
(`POST /services/{slug}/reissue-token`) for a fresh one — and **409** if the
request is not in a verifiable state.

After verification the request **always** goes to manual review; you are
notified by email. Nothing about a pending request is public.

## Reversibility — read before you write

There is **no** DELETE, cancel, withdraw, undo or rollback anywhere in this API.
Once a submission, an update request, an ownership verification or a paid
suggestion is accepted, there is no programmatic way to take it back, and the
x402 fees attached to them are stated to be non-refundable.

What exists instead is decay, not reversal:

- a submission still pending after the 7-day review window is auto-rejected,
  with an email notification;
- an unverified update request lapses when its 72-hour token expires.

Delisting is handled **out of band** — email `info@x402-list.com`, or use the
owner-update flow at `/services/{slug}/update`.

There is also **no idempotency key**. If `POST /submit` times out, you cannot
tell a duplicate from a first attempt except by hitting the cooldown. Record
your own `submission_id` on the first success and check before retrying.

## Provenance you cannot set

Each listing carries a `source` recording how it entered the directory:
`submitted`, or `imported:bazaar` / `imported:x402scan` for auto-imports from
public x402 indexes. An imported listing is not an operator endorsement — claim
and correct it through the owner-update flow above.
