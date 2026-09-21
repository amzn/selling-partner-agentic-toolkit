---
name: account-setup
description: >-
  Runs at the very start of an Amazon Selling Partner Connector session to establish which seller account and
  marketplace to work in, so every other skill can reuse it without asking again. Confirms
  the identity the session already carries, or — if it doesn't — asks the seller for
  their Merchant Token (a.k.a. Seller ID / Merchant ID / MCID — used as entityId) and
  Marketplace ID, and helps them pick the marketplace (or account) when they have more than
  one. Remembers the chosen account + marketplace so future sessions confirm instead
  of re-asking. Triggers on: session start, first message, "get me set up", "connect my
  account", "which account", set up Amazon Selling Partner Connector. Do NOT use for: any actual selling task once the
  account context is established.
version: 1.0.0
license: Apache-2.0
metadata:
  author: Amazon Selling Partner
  persona: any seller starting an Amazon Selling Partner Connector session — category-agnostic
  pattern: Setup + Inversion
  tags: [sp-api, session, onboarding, identity, setup]
  platforms: [claude-desktop, claude-code, aws-quick]
  apis: [identity]
---

# Account Setup

## Description

The first thing that runs in an Amazon Selling Partner Connector session. It pins down **which account and
marketplace** the seller is operating in and holds it as the session context, so the
listings, inventory, and inbound skills can just use it (each says "ask once and reuse" —
this is where it's asked). It **prefers** the identity the session already carries and
only asks the seller for what's missing.

## The identifiers

There are really **two** things to establish — an account/entity identifier and a marketplace.

| Field | What it is | Example |
|-------|-----------|---------|
| **Seller ID / Merchant Token / MCID** | the seller's account/entity ID (`A`-format) — Amazon Selling Partner Connector uses it as `entityId`; lives in Account Info → Business Information | `A1WBZMPXK3ADFU` |
| **Marketplace ID** | the specific Amazon store / country — a **fixed, known value**; changes per marketplace | `ATVPDKIKX0DER` (US), `A1F83G8C2ARO7P` (UK) |

How they relate:
- For current accounts, **Seller ID = Merchant ID = Merchant Token = MCID = `entityId`** — it's
  **one `A`-format value**. The label varies across Seller Central, but it's the same identifier,
  and Amazon Selling Partner Connector passes it as `entityId` on every gateway call.
- It is **scoped to a marketplace group** (NA = US/CA/MX, EU, FE). A seller operating in one
  group has a single value; a seller spanning multiple groups (typically an aggregator /
  Professional account) can have a **different MCID per group** — so **re-confirm the entity ID
  when the seller switches marketplace groups**. It is **not** a single global value, and **not**
  strictly one-per-marketplace.
- **Telling the `A`-format IDs apart (important):** the Seller ID / MCID, the Marketplace ID, and
  PA IDs are *all* `A…`. To avoid mixing them up, **match any `A…` value against the known
  Marketplace ID set first** (the table in where-to-find-ids.md); anything that **isn't** a known
  Marketplace ID is the **Seller ID / entity ID**. Don't assume the first `A…` you see is the Seller ID.

See [where-to-find-ids.md](references/where-to-find-ids.md).

> **Naming:** "Seller ID," "Merchant ID," "Merchant Token," and "MCID" are the **same value**;
> Amazon Selling Partner Connector uses it as `entityId`. Use the seller's wording when talking to them; use `entityId`
> for tool calls.

## When to use this skill

Use at the **start of a session**, or whenever the active account/marketplace is unknown
or needs to change. Once the context is set, hand off to the task the seller actually wants.

## Workflow

### Step 1 — Reuse what's already known (don't ask twice)
Before asking the seller anything, check in this order:
1. **The session.** The Amazon Selling Partner Connector connection authenticates **as the seller**, so the session
   often already carries the account identity (Seller ID / entity ID / marketplace). **Prefer
   this — never ask for what's already there.** *How* it gets there is a gateway concern; the
   skill only needs to check whether it's present.
2. **Saved setup from a previous session.** If the agent remembered this seller's account +
   marketplace last time (Step 5), reuse it (with a quick confirmation — see Step 5).

If either gives you the account, **confirm in one line** ("Working as account `A…`, US
marketplace — switch?") and skip to the task. Only collect what's still missing.

### Step 2 — Ask only for what's missing
If the session doesn't already carry the identity, ask the seller — and **always tell them the
exact place to find the Seller ID**, the full path **Seller Central → Settings → Account
Information → Business Information**, then ask **which marketplace**:
> To get started I need your **Seller ID / Merchant Token** — your Amazon account ID, the `A…`
> value in **Seller Central → Settings → Account Information → Business Information** (it's the
> same value Amazon Selling Partner Connector uses as the `entityId` / MCID) — and which **Marketplace** this is for (e.g.
> `ATVPDKIKX0DER` for the US).

Only ask for what you don't already have.

### Step 3 — Multiple marketplaces (and, rarely, multiple accounts)
- **Multiple marketplaces:** ask **which marketplace** this session is for. The Seller ID / MCID
  is the **same within a marketplace group**, but a seller spanning groups (NA / EU / FE) can have
  a **different MCID per group** — so when the seller crosses groups, **re-confirm the entity ID**.
- **Multiple separate seller accounts/entities:** there's a different Seller ID per account — ask
  **which account** as well.

When the seller has more than one, **tell them explicitly** that you'll work in **one account +
one marketplace at a time** (never blend them) and that they can **switch later by re-running
setup**.

### Step 4 — Pin the active account, confirm, and hand off
Echo back the chosen **Seller ID (entity ID) + Marketplace ID** for confirmation, and
**pin that entity ID + marketplace as the session's active context** — every gateway tool call
(search or execute) carries this **`entityId` + `marketplaceId`**. State you'll reuse
it so other skills won't re-ask, then move on.

> **At tool-invocation time:** if the seller has **multiple accounts / marketplaces** and an
> active one isn't already pinned, **ask which entity ID and which marketplace to invoke with
> before calling the tool** — never silently default. One entity ID + one marketplace per call.

### Step 5 — Remember it for next time (so the seller isn't asked again)
After confirmation, **save the chosen account + marketplace to the agent's persistent memory**
(where the host supports it — Claude Desktop/Code, or the Quick profile) so a future session
can **confirm instead of re-collect**. Two rules:
- **Persist the *preference* — especially the marketplace** (the seller's real choice when
  they have several). The account identity is usually re-established by the session each time,
  so you're mainly remembering the marketplace pick.
- **Reconcile only against identity you actually have.** If the session carries identity, it
  wins — if a remembered value disagrees, prefer the session and re-confirm. If the session
  carries **no** identity, there's nothing to auto-check against, so **reuse the remembered
  values with a one-line confirmation** ("Last time you used account `A…`, US — still right?").
  Full automatic reconciliation arrives once a gateway identity tool exists.

### Step 6 — Self-correct on rejection
There's no identity tool to pre-validate these, so the **first real tool call** is the true
check. If a gateway call fails and the error **could be identity-related** (invalid/unknown
account or marketplace), come back here, point the seller at Seller Central → Account Info,
and re-collect.

## Guardrails
- **Don't guess or fabricate identifiers.** If you don't have a value and the session
  didn't supply it, ask — never invent a Seller ID, marketplace, or entity ID.
- **Disambiguate `A`-format IDs.** Seller ID / MCID, Marketplace ID, and PA IDs are all `A…`.
  Match against the known Marketplace ID set first; anything that isn't a known Marketplace ID
  is the Seller ID / entity ID. Don't treat the first `A…` value you see as the Seller ID.
- **Pin one entity ID + marketplace per tool call.** Every gateway search/execute call carries
  the `entityId` + `marketplaceId`. If the seller has multiple and an active one isn't pinned,
  **ask which to use before invoking the tool** — don't default silently or blend.
- **Reconcile remembered identity with the live session** — never reuse a stored Seller ID
  if the session authenticates a different account; re-confirm and re-pin.
- **Confirm before reusing.** Read the chosen IDs back to the seller once.
- **Treat IDs as account context, not secrets** — they're identifiers, but don't surface them
  more than needed, and never log credentials/tokens.

## References
- [where-to-find-ids.md](references/where-to-find-ids.md) — exactly where each ID lives in
  Seller Central, how to tell the `A`-format IDs apart, and notes on multiple accounts/marketplaces.

> **Note (gateway iteration):** the gateway has no "who am I / list my marketplaces" tool
> today, so this skill asks. When a Sellers/identity tool (e.g. marketplace participations)
> is added, Step 1 can fetch and present the seller's accounts automatically and Step 2's
> manual entry goes away.
