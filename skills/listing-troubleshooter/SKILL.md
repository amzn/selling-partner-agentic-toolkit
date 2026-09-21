---
name: listing-troubleshooter
description: >-
  Top-level entry point for any "something's wrong with my listing" question. Does a
  quick scan to read the listing's status and issues, identifies the symptom, and routes
  the seller to the right specialist skill — issue-fixing, buyability, searchability, or
  compliance. Triggers on: something's wrong with my listing, troubleshoot my listing, my
  product has a problem, listing not working, where do I start, "can I list this", "what do I
  need to sell X". Routes to: listing-issues, listing-buyability, listing-searchability,
  listing-compliance.
version: 1.0.0
license: Apache-2.0
metadata:
  author: Amazon Selling Partner
  persona: any non-technical seller managing Amazon listings — category-agnostic
  pattern: Inversion (router)
  tags: [sp-api, listings, troubleshooting, router]
  platforms: [claude-desktop, claude-code, aws-quick]
  apis: [listings-items]
---

# Listing Troubleshooter (router)

## Description

The front door for listing problems. A seller rarely knows whether their issue is an
"issue," a buyability problem, or a searchability problem. This skill does a **quick scan**,
reads the symptom, and **routes** to the right specialist skill — it diagnoses and directs;
it does **not** make changes itself (the specialist skills own the fixes, each behind its
own approval gate).

## When to use this skill

Use when the seller's complaint is **non-specific** ("something's wrong," "my product
isn't working," "where do I start"). If they already name the symptom (can't be bought /
can't be found / has errors), you can go straight to that specialist skill.

## Tools this skill orchestrates

| Step | Tool | Purpose |
|------|------|---------|
| Scan | `listings_searchListingsItems` | One read for status + issues (`includedData=summaries,issues`) to classify the symptom |

> Routing only — this skill performs no writes.

## Workflow (scan → classify → route)

### Step 1 — Quick scan *(skip when there's nothing to scan yet)*
`listings_searchListingsItems` with `includedData=summaries,issues` for the SKU(s) in
question. Read `summaries[].status` and `issues[]`.

**Skip the scan** when the seller is asking about a product they haven't listed yet — "can I
list this", "what do I need to sell X", or any new product with no SKU or ASIN. There is
nothing to scan, and an empty result is not a diagnosis. Route straight to
[listing-compliance](../listing-compliance/SKILL.md) instead.

### Step 2 — Classify the symptom and route
Pick the **most impactful** matching path (a listing can have more than one):

| What you see / hear | Route to |
|---------------------|----------|
| `issues[]` present (errors/warnings to fix) | **`listing-issues`** |
| Not `BUYABLE` — out of stock, no offer, incomplete product | **`listing-buyability`** |
| Not `DISCOVERABLE`, "can't be found," or "improve ranking/visibility" | **`listing-searchability`** |
| Inventory / "running out" / replenishment | hand to **`stockout-prevention`** |
| "Can I list this" / "what do I need to sell X" / "is this compliant" / a new product with no listing yet | **`listing-compliance`** |

Explain *why* you're routing ("you're not buyable and there's no offer — let's use the
buyability troubleshooter"), then continue in that skill.

### Step 3 — If multiple problems
Lead with the one that costs the most sales (not buyable > not found > quality warning),
handle it, then offer to address the next.

## Guardrails
- **Diagnose and route only** — never write here; each specialist skill owns its own
  preview → approve gate.
- **Don't over-claim** — "no issues" doesn't mean healthy; check buyability and
  searchability before declaring a listing fine.
- No secrets/PII.

## References
- **`listing-issues`** — fix problems when `issues[]` are present.
- **`listing-buyability`** — why it can't be bought (offer / completeness / inventory).
- **`listing-searchability`** — why it can't be found + search optimization.
- **`listing-compliance`** — pre-flight compliance gate for "can I list this" and for any
  compliance-sensitive write the specialists are about to make.
