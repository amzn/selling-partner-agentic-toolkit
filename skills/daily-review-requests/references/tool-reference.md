# Tool reference — daily-review-requests

Three tools on the `sp-ai` MCP server, chained **discover → gate → send**. Every call requires
`entityId` and a single `marketplaceIds` value. `amazonOrderId` is the plain
3-7-7 identifier (e.g. `902-1845936-5435065`) and flows straight from Step 1 into Steps 2 and 3 —
never construct, alter, or fabricate it.

> **MCP-layer vs. native SP-API parameters.** `entityId` (the merchant MCID) is required by the
> **MCP server layer** for authorization context — it is **not** a native SP-API parameter. The
> underlying Solicitations/Orders operations take only `amazonOrderId` + `marketplaceIds` (and the
> Orders time-window filters). Pass `entityId` on every tool call here, but don't expect to see it
> in the raw SP-API OpenAPI specs.

---

## 1. `searchOrders_searchOrders` (read-only)
Returns orders created or updated in a time window.

- **Required:** `entityId` (MCP-layer), and **exactly one of** `createdAfter` **or** `lastUpdatedAfter` (ISO-8601).
- **Useful optional:** `marketplaceIds` (scope to the seller's marketplace), `fulfillmentStatuses`,
  `createdBefore` / `lastUpdatedBefore` (must be ≥ 2 minutes before now), `maxResultsPerPage`
  (1–100, default 100), `paginationToken`, `includedData`.
- **Output:** `orders[]` + `pagination.nextToken` (token **expires 24h** after issuance).
- **Rate limit:** **0.0056 req/s, burst 20** — roughly **1 request every ~3 minutes**. This is the
  slow step. Keep the lookback tight.
  - **Daily run:** `lastUpdatedAfter` ≈ 48h ago.
  - **First-run catch-up:** optionally `createdAfter` ≈ 30d ago once, then revert to daily.
- **PII:** do **not** pass `includedData: [BUYER]` / `[RECIPIENT]` — the skill only needs
  `amazonOrderId`. Omitting it returns basic order data only.

> **Why polling (not notifications).** Amazon's official Solicitations setup guide describes a
> notification-driven discovery path: subscribe to the `ORDER_CHANGE` notification, watch for
> `OrderStatus = Shipped`, read `EarliestDeliveryDate`/`LatestDeliveryDate`, and cache the
> `amazonOrderId`. This skill intentionally uses **polling** (`searchOrders` with a short
> `lastUpdatedAfter`) instead — it needs no SQS/notification-destination setup, which keeps a
> daily-run skill usable by a non-technical seller through the connector. The notification approach
> is a valid (and lower-latency) alternative for a server-side integration; it's just heavier than a
> once-a-day sweep needs.

## 2. `solicitations_getSolicitationActionsForOrder` (read — the gate)
The authoritative eligibility check. **Call this for every order before any send.**

- **Required:** `amazonOrderId`, `marketplaceIds` (single), `entityId`.
- **Output:** available solicitation `actions`. Currently only product-review-and-seller-feedback.
- **Interpretation:**
  - **Non-empty actions** → **eligible** (delivered, inside the solicitation window, not yet
    solicited, buyer not opted out). The window is documented in Amazon's setup guide as **5 days
    after `EarliestDeliveryDate` through 30 days after `LatestDeliveryDate`** — but the gate
    enforces it for you; don't recompute it client-side.
  - **Empty actions** → **ineligible.** The order exists but can't be solicited now (already
    solicited / outside window / not yet delivered / opted out). Do not guess which — mark
    ineligible.
  - **Error** → order not found or access denied → skip and note; do not retry blindly.
- **Rate limit:** 1 req/s, burst 5.

> The create tool takes a plain `amazonOrderId` — there is **no HAL-href path to substitute**. Pass
> the same `amazonOrderId` you gated in Step 2 directly to Step 3. (This avoids the classic
> malformed-path bug of feeding an action `href` into an order-id parameter.)

## 3. `solicitations_createProductReviewAndSellerFeedbackSolicitation` (write, `requiresHumanReview`)
Sends Amazon's standard product-review + seller-feedback solicitation for one order.

- **Required:** `amazonOrderId`, `marketplaceIds` (single), `entityId` (MCP-layer).
- **Output:** **HTTP 201 on success, with an effectively empty body** (the response schema carries
  only an optional `errors` field).
- **Behavior:** **one per order lifetime** — a second attempt returns **access-denied (403)**. Order
  must be delivered and within the solicitation window (5 days after `EarliestDeliveryDate` to 30
  days after `LatestDeliveryDate`) — guaranteed if Step 2 returned actions. Amazon sends
  a **template-based email the seller cannot customize**. Does **not** appear in Seller Central
  Messaging or the buyer Message Center.
- **`requiresHumanReview: true`** — the connector surfaces its own per-call approval. The skill also
  gets **batch** confirmation from the seller before sending (Step 3).
- **Rate limit:** 1 req/s, burst 5 — pace the batch.

### Result interpretation (do not over-dedupe on 403)
| Result | Meaning | Action |
|--------|---------|--------|
| 201, empty body (success) | Solicitation sent | Mark **sent** + timestamp |
| Access-denied / already-solicited | Already requested earlier (once-per-order) | Mark **already requested** — **not** a retry |
| Auth / expired-token / invalid-signature | Credential problem, **not** "already sent" | Re-auth and **resume**; do **NOT** mark processed |
| 429 throttling | Rate limited | Back off (respect burst 5 / 1 rps), re-enqueue |

A 403 is overloaded: it can mean *already-solicited* **or** *auth failure*. Only the first is a
"done." Treating every 403 as done would silently skip real orders during a token outage.

---

## Chain summary
```
searchOrders_searchOrders (recent window)
        │  amazonOrderId[]  (no sentiment/value filtering)
        ▼
getSolicitationActionsForOrder (per order)   ── empty actions ──▶ skip (ineligible)
        │  eligible set
        ▼
[show eligible set + count → seller confirms]  (send-to-all-eligible or none)
        │
        ▼
createProductReviewAndSellerFeedbackSolicitation (per eligible order, 1 rps)
        │
        ▼
run summary: sent / skipped(by reason) / already-requested / errors-to-retry
```
Store only `amazonOrderId` + status + timestamp. Never buyer PII.
