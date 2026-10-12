# Tool reference — daily-review-requests

Three tools on the `sp-ai` MCP server, chained **discover → gate → send**. Every call requires
`entityId` (merchant MCID) and a single `marketplaceIds` value. `amazonOrderId` is the plain
3-7-7 identifier (e.g. `902-1845936-5435065`) and flows straight from Step 1 into Steps 2 and 3 —
never construct, alter, or fabricate it.

---

## 1. `searchOrders_searchOrders` (read-only)
Returns orders created or updated in a time window.

- **Required:** `entityId`, and **exactly one of** `createdAfter` **or** `lastUpdatedAfter` (ISO-8601).
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

## 2. `solicitations_getSolicitationActionsForOrder` (read — the gate)
The authoritative eligibility check. **Call this for every order before any send.**

- **Required:** `amazonOrderId`, `marketplaceIds` (single), `entityId`.
- **Output:** available solicitation `actions`. Currently only product-review-and-seller-feedback.
- **Interpretation:**
  - **Non-empty actions** → **eligible** (delivered, inside the 5–30 day window, not yet solicited,
    buyer not opted out).
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

- **Required:** `amazonOrderId`, `marketplaceIds` (single), `entityId`.
- **Output:** **empty body on success.**
- **Behavior:** **one per order lifetime** — a second attempt returns **access-denied**. Order must
  be delivered and within the 5–30 day window (guaranteed if Step 2 returned actions). Amazon sends
  a **template-based email the seller cannot customize**. Does **not** appear in Seller Central
  Messaging or the buyer Message Center.
- **`requiresHumanReview: true`** — the connector surfaces its own per-call approval. The skill also
  gets **batch** confirmation from the seller before sending (Step 3).
- **Rate limit:** 1 req/s, burst 5 — pace the batch.

### Result interpretation (do not over-dedupe on 403)
| Result | Meaning | Action |
|--------|---------|--------|
| Empty body (success) | Solicitation sent | Mark **sent** + timestamp |
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
