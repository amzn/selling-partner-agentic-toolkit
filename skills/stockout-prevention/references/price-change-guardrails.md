# Price-write mechanics

A price write is the only **write** action in this skill. It must follow the
Inversion pattern: the LLM proposes, the human disposes.

> **This skill gives no pricing guidance, in either direction.** It never suggests raising
> or lowering a price, never offers a price change as an option, and never comments on
> whether a price is wise, safe, or advisable. These mechanics apply **only** when the
> seller has named a specific price themselves. If the seller asks whether they should
> change price, say that pricing is their decision and this skill doesn't advise on it.

## Rules
1. **Preview first.** Always call `listings_patchListingsItem` with
   `mode: VALIDATION_PREVIEW`. This validates the patch without making a live change.
2. **Show the diff.** Present the SKU and current → the price the seller specified. Nothing
   else — no projected effect, no revert date, no commentary.
3. **Use the seller's number exactly.** Never round, adjust, or suggest a different figure.
   If the price is ambiguous, ask them to restate it; never infer one.
4. **Explicit approval.** Do not run a live patch until the seller types "approve"
   (or supplies a modified price). Silence is not approval.
5. **One change at a time.** Draft each SKU's change as its own numbered item; never
   bundle multiple live writes into one approval.
6. **No unsolicited follow-up.** After the write, confirm what was submitted. Do not
   propose a revert, a further change, or a review of the result.

## `listings_patchListingsItem` essentials
- Required: `sellerId`, `sku`, `marketplaceIds`, `productType`, `patches`, `entityId`.
- `patches` is a JSON Patch array (op/path/value). Only top-level listing attributes
  can be patched.
- An `ACCEPTED` response means the patch was **submitted**, not that the change is
  live yet — confirm with the seller and verify afterward.
