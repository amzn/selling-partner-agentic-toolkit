---
name: listing-searchability
description: >-
  Diagnoses why a seller's listing isn't found in search and helps make it more
  discoverable. Checks whether the listing is discoverable at all, then optimizes the
  content shoppers search on — title, description, bullet points, and generic keywords —
  with the seller's approval. Triggers on: not showing up in search, can't be found,
  not discoverable, improve ranking, search visibility, optimize my listing, SEO. Do NOT
  use for: not-buyable (use listing-buyability) or fixing reported issues
  (use listing-issues).
version: 1.0.0
license: Apache-2.0
metadata:
  author: Amazon Selling Partner
  persona: any non-technical seller managing Amazon listings — category-agnostic
  pattern: Reviewer + Inversion
  tags: [sp-api, listings, searchability, discoverability, optimization]
  platforms: [claude-desktop, claude-code, aws-quick]
  apis: [listings-items]
---

# Listing Searchability Troubleshooter

## Description

Answers "**why isn't this showing up / how do I get found?**" Discoverability is two
things: is the listing **indexed at all**, and — once indexed — is its **content strong
enough to rank** for what shoppers type. This skill checks discoverability, then helps
**optimize** the searchable content (title, description, bullets, generic keywords) with
the seller's approval.

> Not being found isn't always a *suppression* with an `issue`. A listing can be indexed
> but rank poorly, or fail to index because content (e.g. a main image) is incomplete —
> often with an empty `issues[]`. So treat searchability as a spectrum, not a binary flag.

## When to use this skill

Use when a seller asks why a product isn't appearing in search, or wants to improve its
visibility/ranking. Don't use for "can't be bought" (buyability skill) or fixing reported
issues (issue-fixer).

## Tools this skill orchestrates

| Step | Tool | Purpose |
|------|------|---------|
| Inspect | `listings_searchListingsItems` | Discoverable status + current content (`includedData=summaries,attributes,issues`) |
| Optimize | `listings_patchListingsItem` | Improve title / description / bullets / generic keywords (preview → approve) |

> Needs `entityId` + `marketplaceIds`. Ask once, reuse.

## Workflow

### Step 1 — Is it discoverable?
`listings_searchListingsItems` (`includedData=summaries,attributes,issues`). Check whether
`DISCOVERABLE` is present.
- **Not discoverable** → look for a content gap that blocks indexing (e.g. missing main
  image) — note it may carry **no formal issue**. If it's a reported issue, hand to
  `listing-issues`.

### Step 2 — Optimize the searchable content
Review the current `item_name` (title), `product_description`, `bullet_point`, and
`generic_keyword`. Suggest concrete improvements (clarity, relevant terms shoppers use, no
keyword stuffing, within policy/length).

**Compliance gate first (before applying).** Titles, bullets, descriptions, and keywords are
where prohibited or regulated claims most often enter a listing — "FDA approved",
"antimicrobial", "kills 99.9% of germs", "organic", "bamboo", tribal / Native American terms,
endorsements. If a proposed change **adds or alters such a claim**, or changes category,
condition, images, or identifiers, run the [listing-compliance](../listing-compliance/SKILL.md)
gate and continue only on the seller's explicit go. Pure wording or keyword-relevance edits with
no new claim don't need it; if unsure whether a term is a regulated claim, run the gate.

Then apply via `listings_patchListingsItem` in **`mode: VALIDATION_PREVIEW`**, show current →
proposed, **wait for approval**, then submit.

### Step 3 — Relevancy / ranking proxy  *(pending tool)*
Once the **Catalog Items search-by-keywords** tool is on the gateway, search the seller's
target keywords and check whether their ASIN appears (and roughly where) as a **proxy for
relevancy/ranking**. **Not available yet** — until then, treat ranking advice as guidance,
and present any proxy as an *indicator, not a guarantee* of rank.

### Step 4 — Summarize
State whether it's discoverable, the top 1–2 content improvements, and what (if anything)
was drafted for approval.

## Guardrails
- **Only edit content the seller controls.** On a shared ASIN the seller may not own the
  product content — frame those as "attempt a contribution / may need the listing owner,"
  don't blindly patch.
- **Preview → approve → verify** on any write; **never fabricate keywords** or stuff them.
- **Relevancy proxy is an indicator, not real ranking** — say so; don't overpromise SEO.
- **Optimizing for search is how regulated claims sneak in.** Adding a high-traffic term like
  "antimicrobial", "organic", or "FDA approved" to a title or bullet is a **compliance** change,
  not an SEO change — run the [listing-compliance](../listing-compliance/SKILL.md) gate before the
  preview; never add such a term just because it ranks.
- Stay within [Amazon's selling policies](https://sellercentral.amazon.com/help/hub/reference/GSNV3657R94YP9DZ)
  and the law; the seller is responsible for compliance. No secrets/PII.

## References
- For "can't be bought," use **`listing-buyability`**.
- For listings with explicit `issues[]`, use **`listing-issues`**.
- For a content change that adds or alters a regulated claim, the pre-flight gate is
  **`listing-compliance`**.
