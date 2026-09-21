# Where to find the account identifiers

There are really **two** things: the account/entity identifier (one value, several names)
and the Marketplace ID. Point the seller here when the session didn't already supply them.

## Seller ID / Merchant Token / MCID  (used as `entityId`)
- **Seller Central → Settings (gear) → Account Information → Business Information → Merchant token.**
- Looks like `AXXXXXXXXXXXXX` (starts with `A`, ~13–14 chars).
- **"Seller ID," "Merchant ID," "Merchant Token," and "MCID" are the same value** — the
  label just varies in different parts of Seller Central. Amazon Selling Partner Connector uses it as `entityId` on
  every gateway call.
- It is **scoped to a marketplace group** (NA = US/CA/MX, EU, FE). A seller operating in one
  group has a **single** value; a seller spanning multiple groups (e.g. an aggregator /
  Professional account) can have a **different MCID per group** — **re-confirm when switching
  groups.** It is **not** a single global value.

## Marketplace ID
- The marketplace the seller is operating in — a **fixed, known set**. Common IDs:

| Marketplace | ID |
|-------------|-----|
| US | `ATVPDKIKX0DER` |
| Canada | `A2EUQ1WTGCTBG2` |
| Mexico | `A1AM78C64UM0Y8` |
| UK | `A1F83G8C2ARO7P` |
| Germany | `A1PA6795UKMFR9` |
| Japan | `A1VC38T7YXB528` |
| India | `A21TJRUUN4KGV` |

- A seller can be active in several marketplaces — ask which one this session is for.

## Telling the `A`-format IDs apart
The Seller ID / MCID, the Marketplace ID, and PA IDs are **all** `A…`, so it's easy to mix
them up (Seller Assistant historically did). To disambiguate:
- **Match any `A…` value against the known Marketplace ID set above first.** If it matches,
  it's the **Marketplace ID**.
- Anything `A…` that **isn't** a known Marketplace ID is the **Seller ID / entity ID**.
- **Don't assume the first `A…` value you see is the Seller ID** — check it against the
  marketplace list before deciding.

## Multiple marketplaces / accounts
- **Multiple marketplaces, one account:** pick the **marketplace** for the session; the Seller
  ID / MCID is the same **within a marketplace group**, but **re-confirm the entity ID** when the
  seller crosses groups (NA → EU → FE).
- **Separate seller accounts/entities:** there's a different Seller ID per account — pick which
  **account** as well.
- Work in **one account + one marketplace** per request; re-run setup to switch.

> No API call returns these today (the gateway has no marketplace-participations / identity
> tool yet), so they're either carried by the session or collected from the seller. When that
> tool lands, the skill can list the seller's accounts and marketplaces for them to pick
> instead of asking them to copy them.
