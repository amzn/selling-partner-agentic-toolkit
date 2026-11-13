# Solicitation policy — what makes this skill compliant

This skill automates Amazon's **native** review solicitation. The compliance posture below is
drawn from Amazon's Customer Reviews policy and Communication Guidelines (Seller Central). The
single most important line Amazon draws is **native "Request a Review" action vs. your own bulk
messages** — this skill is firmly on the native side.

## The sanctioned mechanism
- `createProductReviewAndSellerFeedbackSolicitation` is the API equivalent of the Seller Central
  **"Request a Review"** button. It sends **Amazon's own fixed template** — the seller cannot add
  or change wording — so content neutrality is **structural**, not a thing the skill has to police.
- This is categorically different from the "third-party bulk-emailing tools" and "third-party
  review solicitation services" that the Customer Reviews policy explicitly flags as violations.
  Using email or Buyer-Seller Messaging to solicit reviews is unsolicited, opt-out-bound, and
  counts as a second request per order.

## The three limits — enforced for you by the eligibility gate
`getSolicitationActionsForOrder` returns **empty actions** whenever any of these is not satisfied,
so the skill never has to compute them:
1. **Once per order lifetime.** A second solicitation (or free-form proactive message) for the same
   order is refused; the send call returns access-denied.
2. **5–30 days after delivery.** The request is only allowed inside this window — documented in
   Amazon's Solicitations setup guide as **5 days after `EarliestDeliveryDate` through 30 days
   after `LatestDeliveryDate`**. It is **enforced server-side by the gate**, not by the send
   operation's own schema, so do **not** recompute it client-side — trust the gate.
3. **Buyer opt-out respected.** Buyers who opted out of solicitations yield empty actions → no send.

## The lines the seller/skill must not cross (these are on you, not the API)
These are **zero-tolerance** violations under the Customer Reviews policy. Consequences include
permanent loss of selling privileges, fund withholding, removal of all product reviews, product
delisting, legal referral, and public identification of the seller — and the seller is held
responsible **even if a third-party tool did it without their knowledge**.

- **No selective (sentiment-based) solicitation.** Do not send only to buyers predicted to be
  happy, and do not skip buyers who might leave a negative review. **Send to every eligible order
  or none.** "Selectively seeking positive reviews" and "diverting negative reviews" are violations.
- **No incentives.** Never offer money, gift cards, discounts, or refunds for a review, and never
  condition a refund/replacement on a review. (The fixed template can't carry an incentive anyway.)
- **No parallel email / Buyer-Seller Messaging channel.** Don't *also* message buyers to ask for
  reviews — that's the violating channel, and a second request per order.
- **No asking to change/remove a review**, no diverting negative feedback elsewhere, no fake or
  employee/family reviews.

> **The one sanctioned exception — and it's not this skill.** Amazon does provide a legitimate path
> for *seller feedback* (not product reviews): after you have **resolved** a buyer's problem, the
> Messaging API's `createNegativeFeedbackRemoval` operation lets you ask the buyer to remove the
> now-resolved negative **seller feedback**. That is a different operation, a different API, and a
> different situation (a resolved issue — not "please change your rating"). This review-solicitation
> skill never does it. If a seller asks to get a negative review/feedback changed, decline the
> "change it to 5 stars" ask, but you may point them to that sanctioned Seller Central / Messaging
> path for genuinely resolved issues.

## Why neutrality-by-default matters for a shared skill
Because the API hands you a fixed neutral template, a "dumb" send-to-all-eligible loop is compliant
by construction. The risk is never the API — it's any **sentiment selectivity, incentive, or
parallel email channel** layered on top. Keep the skill a plain gate-first, send-to-all-eligible
loop and it stays clean for every seller who runs it. Never add an override that injects custom
messaging or filters the eligible set.

## Visibility
API solicitations do **not** appear in Seller Central Messaging or the buyer's Message Center. That
is why the skill's own run summary (order id → sent / skipped / already-requested) is the seller's
record, and why running it daily (not repeatedly within a day) is the right cadence.
