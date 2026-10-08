# Reconciliation notes — Month-End Closing

`finances_monthEndClosing` (gateway tool `monthEndClosing`) reconciles a completed calendar
month against its settlements, **one result per account type**. It never mutates anything.

## The core comparison (per account type)
- **LHS — `lhsMonthNetProceeds`:** the month's net proceeds from the monthly summary.
- **RHS — `rhsSettlementSum`:** the trimmed sum of net proceeds of the settlements that overlap
  the month.
- **`difference` = LHS − RHS.** The expected, normal result parses to numeric zero (e.g. `"0"`
  or `"0.00"` — compare the numeric value, not the literal string `"0"`) — a **matched close**. A
  difference that parses to a non-zero value is **not a discrepancy to chase**: it means some transactions for the
  period are **still being processed** and the month hasn't fully settled. The tool never fails
  on it; the skill should tell the seller to **try again after some time**.
- **`beginningBalance` / `endingBalance`:** the account's balance at the month's edges.
- All monetary amounts are **decimal strings** — preserve them; don't round in a way that hides
  a difference.

## Settlement classifications (why a settlement counts partially)
> **Internal only.** These labels are for the agent's reasoning. **Never show `inside`,
> `straddles_start`, `open_at_end`, or `spans_month` to the seller** — translate their effect
> into plain language (e.g. "part of this settlement belongs to the prior month").

- **`inside`** — opened and closed within the month; full (non-negative) net proceeds count.
- **`straddles_start`** — opened before the month, closed within it; the beginning balance
  carried in from the prior month is subtracted.
- **`open_at_end`** — opened within the month, still open at month end; its net proceeds are
  excluded and the ending balance stands in as its month-end contribution.
- **`spans_month`** — opened before and still open at month end; contributes only the balance
  movement (ending − beginning).

Each settlement's `contribution` is the trimmed amount it adds to the RHS. Negative closed-
settlement net proceeds are treated as 0 (a negative settlement produces no payout and carries
into the next settlement).

## Payout status (per settlement)
`PAID` (**Amazon processed the payout — this does NOT prove the deposit reached the bank**) ·
`IN_TRANSIT` / `STARTED` (on the way) · `FAILED` / `CANCELLED` · `NO_PAYOUT` (a closed settlement
that produced no disbursement) · `null` status (no matching payout yet, e.g. a still-open
settlement). `payoutStatusHistory` lists the status transitions with dates and any failure reason.

## The reconciliation trail (how to explain a matched close)
A month's payouts can total *more* than its net proceeds without any error — settlements cross
calendar months. Show the trail from returned values:

```
Settlements closed during the month      Σ netProceeds of settlements whose periodEnd is in-month
− Opening balance carried in             beginningBalance
+ Month-end balance carried forward      endingBalance
──────────────────────────────────────
= Calculated month net proceeds          == lhsMonthNetProceeds  (difference parses to zero, e.g. "0"/"0.00" ✓)
```

## Values the skill derives (sums of returned fields — not fabrication)
- **Settlements closed during the month** = Σ `netProceeds` of settlements whose `periodEnd`
  falls in the month.
- **Payouts awaiting bank confirmation** = the payout amounts of those closed settlements. The
  payout amount **always equals the settlement's `netProceeds`** (confirmed invariant), so use
  `netProceeds`; there is no separate payout-amount field.
- **Settlement periods evaluated** = `len(settlements)`.

## Bank-receipt handoff (no bank integration)
The tool returns Amazon-side data only. To confirm a payout actually landed, the skill asks the
seller to paste deposit **amounts + dates** (a reference helps; never an account number or full
statement) and matches them to payouts by amount/reference:
- Matched → "seller-confirmed."
- No match → "unresolved"; ask the seller to check the first business days of the next month.
  Don't assume a cause.
- Confirmed later → a **bank timing item** (received after month-end), not missing cash.

## How to reconcile
1. Confirm the **year/month** match the books you are closing.
2. For each account, confirm `difference` parses to numeric zero (e.g. `"0"` or `"0.00"` — check
   the numeric value, not literal string `"0"`) — a matched close. If it parses to a non-zero
   value, transactions are still processing — wait and re-run rather than investigating a "gap."
3. Use `classification` to explain why each settlement counts partially, and the trail above to
   explain why payouts can exceed net proceeds.
4. Confirm payout **receipt** via the seller's bank deposits (Amazon `PAID` ≠ received).

## What this skill does NOT do
- It does **not** file or submit taxes, VAT, GST, or any regulated/remittance report.
- It does **not** pay, move, or hold money.
- It does **not** change prices, listings, or inventory.
- It reports the numbers the tool returns; it never invents, estimates, or "fixes" a mismatch.

The seller (or their accountant) reviews the reconciliation, investigates gaps, and performs
any filing or action. This skill is a read-and-explain reconciliation aid only.
