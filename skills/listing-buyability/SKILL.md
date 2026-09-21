---
name: listing-buyability
description: >-
  Diagnoses why a seller's listing isn't buyable and relays the specific reason plus
  the next step. Checks the three things that make a listing buyable — a complete
  product, a valid purchasable offer, and available inventory — then fixes the offer
  (preview → approve) or routes inventory elsewhere. Triggers on: not buyable, can't be
  purchased, "why can't customers buy this", no buy box, listing inactive, missing price
  / offer. Do NOT use for: search/discoverability (use listing-searchability),
  fixing reported listing issues (use listing-issues), or inventory replenishment
  (use stockout-prevention).
version: 1.0.0
license: Apache-2.0
metadata:
  author: Amazon Selling Partner
  persona: any non-technical seller managing Amazon listings — category-agnostic
  pattern: Reviewer + Inversion
  tags: [sp-api, listings, buyability, offer, troubleshooting]
  platforms: [claude-desktop, claude-code, aws-quick]
  apis: [listings-items]
---

# Listing Buyability Troubleshooter

## Description

Answers "**why can't customers buy this?**" A listing is buyable only when **all three**
hold: (1) the **product is complete**, (2) there's a **valid purchasable offer**, and
(3) there's **available inventory**. This skill finds which one is missing, **relays the
state and the next step**, and fixes the **offer** with the seller's approval. It does
**not** invent data, and it only edits what the seller controls.

> Not every "not buyable" has a listing *issue*. A listing can be out of stock, or simply
> have no offer, with an empty `issues[]`. So don't treat "no issues" as "fine."

## When to use this skill

Use when a seller asks why a product isn't for sale / buyable / in the Buy Box, or why a
listing is inactive. Don't use for "not showing up in search" (that's the searchability
skill), for fixing reported issues (issue-fixer), or to replenish stock (stockout-prevention).

## Tools this skill orchestrates

| Step | Tool | Purpose |
|------|------|---------|
| Inspect | `listings_searchListingsItems` | Status + offer + attributes (`includedData=summaries,offers,attributes,issues`) |
| Fix offer | `listings_patchListingsItem` | Add/correct the purchasable offer (preview → approve) |

> Needs `entityId` + `marketplaceIds` (e.g. `ATVPDKIKX0DER` US). Ask once, reuse.

## Workflow (diagnose the cause, then relay or fix)

### Step 1 — Inspect
`listings_searchListingsItems` with `includedData=summaries,offers,attributes,issues`.
Read `summaries[].status` (is `BUYABLE` present?), `offers`, and `issues`.

### Step 2 — Find the cause (in this order)
- **Buyable already?** Say so and stop.
- **No valid offer** — `offers` empty or no price → the fix is to add a purchasable offer
  (price + condition). This is in scope (see Step 3).
- **Incomplete product** — a required attribute is missing/invalid (often shows as an
  `ERROR` issue). Relay what's missing. If it's a reported issue, hand to
  **`listing-issues`**. (Fully verifying completeness needs the Product Type
  Definitions tool, which isn't on the gateway yet — so report what the data shows.)
- **No available inventory** — product complete + offer present but still not `BUYABLE` →
  it's almost certainly **out of stock**. Relay that, and point to the next step:
  **FBA → `fba-inbound-management` / `stockout-prevention`**; **MFN → update the available
  quantity**. (This skill relays inventory state and routes; it doesn't manage stock.)

### Step 3 — Whose data is it? (framing for product-data fixes)
If product data is missing/incorrect, identify whether it comes from the **seller's own
contribution**:
- **Their data** → they can fix it (and we can help draft the patch).
- **Shared ASIN, not their data** → they can *attempt* a contribution, but the change may
  need the brand/listing owner or a support case. Say so — don't promise a fix you can't make.

### Step 3b — Compliance gate (before a condition or product-data change)
If the offer fix changes **condition**, or the product-data path touches **claims,
ingredients, category, images, or identifiers**, run the
[listing-compliance](../listing-compliance/SKILL.md) gate now and continue only on the
seller's explicit go. An offer fix that only sets a **price** does not need it. If unsure,
run the gate.

### Step 4 — Fix the offer (preview → approve → verify)
For an offer cause, draft `listings_patchListingsItem` for `purchasable_offer` in
**`mode: VALIDATION_PREVIEW`**, show current → proposed, **wait for approval**, then submit
live. Acceptance is async — offer to re-check that `BUYABLE` returned.

### Step 5 — Summarize
State the single reason it isn't buyable and the one next action (a drafted offer fix, an
inventory hand-off, or a product-data path).

## Guardrails
- **Relay the state, don't guess.** Report exactly what the data shows; if you can't tell
  (e.g. inventory), say "likely out of stock" and route — don't assert.
- **Only edit what the seller controls.** Fix the **offer**; for product content on a
  shared ASIN, frame next steps (Step 3) rather than blindly patching.
- **Inventory is out of scope.** Relay + route to `stockout-prevention` / `fba-inbound-management`;
  don't manage stock here. (MFN quantity edits are a deferred follow-up.)
- **Preview → approve → verify** on any write; never invent a price or attribute value.
- **No pricing guidance.** When an offer is missing a price, ask the seller for the figure
  and use exactly what they give. Never suggest a price level, a range, or a direction, and
  never advise on whether a price will win the Featured Offer — that is the seller's call.
- **Compliance gate before a condition change or product-data change.** If the change touches
  product claims, ingredients, category, condition, images, or identifiers, run the
  [listing-compliance](../listing-compliance/SKILL.md) gate before the offer preview and continue
  only on the seller's explicit go. A price-only offer fix does not need it.
- Stay within [Amazon's selling policies](https://sellercentral.amazon.com/help/hub/reference/GSNV3657R94YP9DZ)
  and the law; the seller is responsible for compliance. No secrets/PII.

## References
- For listings that have explicit `issues[]`, use **`listing-issues`**.
- For "can't be found / optimize for search," use **`listing-searchability`**.
- For a condition change or a product-data change touching claims, ingredients, category,
  images, or identifiers, the pre-flight gate is **`listing-compliance`**.
