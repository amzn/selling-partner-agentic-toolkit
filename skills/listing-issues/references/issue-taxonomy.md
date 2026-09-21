# Listing Issue Taxonomy

Reference detail for reading a listing's `issues[]` and `status` from
`listings_searchListingsItems`. This describes what the tool **returns** (not its inputs —
those are in the MCP tool schema), and it's **loaded on demand**, so SKILL.md stays lean.
Grounded in the SP-API **Manage Listings Issues** guide.

## Listing status (`summaries[].status`)

A listing's status is an **array** of independent states:

| Status | Question it answers | If missing |
|--------|---------------------|-----------|
| `BUYABLE` | Can customers buy it right now? | Not purchasable — usually fully suppressed or out of stock |
| `DISCOVERABLE` | Does it appear in search? | Search-suppressed — buyable by direct link only |

A listing can be `BUYABLE` but **not** `DISCOVERABLE` (search-suppressed), or neither
(fully suppressed). `[BUYABLE, DISCOVERABLE]` with no `ERROR` issues = healthy.

## Issue object fields

Each entry in `issues[]` carries:

- **`code`** — Amazon's error code (e.g. `90220`, `100708`).
- **`message`** — human-readable description with the fix.
- **`severity`** — `ERROR` or `WARNING` (see below).
- **`attributeName` / `attributeNames`** — the attribute(s) at fault — *this is what you fix*.
- **`categories`** — issue category, e.g. `MISSING_ATTRIBUTE`, `INVALID_ATTRIBUTE`, `INVALID_PRICE`.
- **`enforcements.actions[].action`** — what Amazon did to the listing (see below).

## Severity

| Severity | Meaning | Blocks the listing? |
|----------|---------|---------------------|
| `ERROR` | Prevents acceptance, or makes the listing non-buyable / search-suppressed | **Yes** — fix first |
| `WARNING` | Quality problem; the listing stays live | No — surface calmly, don't alarm |

> Example: a pricing issue with code `100708` / category `INVALID_PRICE` is a `WARNING`
> ("not eligible to be the Featured Offer due to uncompetitive price") — the update is
> accepted and the listing stays active; it just may not win the Featured Offer.

## Enforcement actions (`enforcements.actions`)

What Amazon has done to the listing because of the issue. Priority order for fixing:

| Action | Effect | Urgency |
|--------|--------|---------|
| `LISTING_SUPPRESSED` | Entire listing not buyable | **Highest** — lost sales now |
| `SEARCH_SUPPRESSED` | Buyable but not in search results | High — customers can't find it |
| `ATTRIBUTE_SUPPRESSED` | One attribute value hidden | Medium |
| `CATALOG_ITEM_REMOVED` | Item removed from the catalog | Investigate — may need support |

(no enforcement action + `WARNING`) = a quality nudge, not a blocker.

## Priority rule

Same order as SKILL.md Step 3 (`LISTING_SUPPRESSED` → `SEARCH_SUPPRESSED` →
`ATTRIBUTE_SUPPRESSED` → `WARNING`; `ERROR` before `WARNING`) — lead with the issue that
costs the seller the most sales.

## Where issues come from

> On this gateway the only tools are `searchListingsItems` and `patchListingsItem`. The
> other SP-API operations named below are mentioned for accuracy but are **not available
> here** — do not use them.

- **Synchronous** — returned directly in a `patchListingsItem` response that failed
  validation. (SP-API documents the same for `putListingsItem`, but that operation is
  **not available** on this gateway.)
- **Asynchronous** — surfaced via `searchListingsItems` with `includedData=issues`. An
  `ACCEPTED` submission can still produce issues during downstream catalog processing, so
  re-scan to catch them. (SP-API also exposes `getListingsItem` and a
  `LISTINGS_ITEM_ISSUES_CHANGE` notification for this; **neither is on this gateway today** —
  hence the re-scan-on-demand approach.)
