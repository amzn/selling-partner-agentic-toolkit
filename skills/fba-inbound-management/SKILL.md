---
name: fba-inbound-management
description: >-
  Guides a seller end-to-end through sending inventory to Amazon's fulfillment
  network: creating an inbound plan, packing, placement, transportation, delivery
  windows, and shipment confirmation — plus status checks, modifications, and
  cancellation. Drives the sequential CREATE→PACK→PLACE→TRANSPORT→DELIVER→SHIP
  pipeline, polls async operations to completion, and pauses at every decision gate
  for the seller's choice. Triggers on: send inventory to Amazon, create inbound
  plan, ship to FBA, inbound shipment status, cancel inbound plan, add tracking,
  FNSKU labels, prep requirements, schedule FC drop-off. Do NOT use for: stockout /
  replenishment forecasting (use stockout-prevention), listing issues, advertising,
  or orders.
version: 1.0.0
license: Apache-2.0
metadata:
  author: Amazon Selling Partner
  persona: any 3P (seller) account shipping inventory to FBA — category-agnostic
  pattern: Pipeline + Inversion
  tags: [sp-api, fba, inbound, shipments, fulfillment]
  platforms: [claude-desktop, claude-code, aws-quick]
  apis: [fulfillment-inbound]
---

# FBA Inbound Management Skill

## Description

Helps a Seller (3P) create, manage, and execute **inbound plans** to ship inventory
to Amazon's fulfillment network. It runs the full pipeline end-to-end, but it is
**not autonomous**: every async write is polled to confirmed success before moving on,
and at each decision gate (packing, placement, transportation, delivery window,
cancellation) it presents the options and **waits for the seller to choose** —
nothing irreversible happens without an explicit go-ahead.

> Scope: Sellers (3P) only. Vendors (1P) use a separate Direct Fulfillment workflow.
> Only plans created with the v2024-03-20 flow are visible to these tools.

## When to use this skill

Use when the seller wants to send inventory to FBA, check or modify an inbound plan
or shipment, add tracking, get FNSKU labels, see prep requirements, or schedule an FC
drop-off. Do **not** use for stockout/replenishment forecasting, listing content/issues,
advertising, or order management.

## Tools this skill orchestrates

All are SP-API Fulfillment Inbound v2024-03-20 tools on the `amazon-selling-partner` MCP server. Each
pipeline stage follows a **generate → list → confirm** pattern:

| Stage | Generate / Write | List / Read | Confirm |
|-------|------------------|-------------|---------|
| Create | `fbaInbound_createInboundPlan` | `fbaInbound_getInboundPlan` | — |
| Pack info | `fbaInbound_setPackingInformation` | `fbaInbound_listPackingGroupItems` | — |
| Pack | `fbaInbound_generatePackingOptions` | `fbaInbound_listPackingOptions` | `fbaInbound_confirmPackingOption` |
| Place | `fbaInbound_generatePlacementOptions` | `fbaInbound_listPlacementOptions` | `fbaInbound_confirmPlacementOption` |
| Transport | `fbaInbound_generateTransportationOptions` | `fbaInbound_listTransportationOptions` | `fbaInbound_confirmTransportationOptions` |
| Deliver (opt) | `fbaInbound_generateDeliveryWindowOptions` | `fbaInbound_listDeliveryWindowOptions` | `fbaInbound_confirmDeliveryWindowOptions` |

Status & management: `fbaInbound_getInboundOperationStatus` (poll async),
`fbaInbound_listInboundPlans`, `fbaInbound_getShipment`,
`fbaInbound_updateShipmentTrackingDetails`, `fbaInbound_cancelInboundPlan`.
The full tool set (labels, prep, compliance, self-ship, content updates, inspection)
is in [tools-and-additional.md](references/tools-and-additional.md).

> Every call needs `entityId` (merchant id) and a marketplace (e.g. `ATVPDKIKX0DER`
> US, `A21TJRUUN4KGV` India). Ask once and reuse.

## The async pattern (do this after EVERY write)

Almost every write returns an `operationId`, not an immediate result. After each write:
1. Poll `fbaInbound_getInboundOperationStatus` with the `operationId`.
2. `SUCCESS` → proceed. `FAILED` → report the error and advise recovery; do **not**
   proceed. `IN_PROGRESS` → **wait, then poll again with exponential backoff** — give it
   ~10–15s before the first re-check, then roughly double each wait (≈15s → ≈30s → ≈60s).
   Don't hammer the API every couple seconds.
3. **Cap at 3 polls.** If it's still `IN_PROGRESS` after the 3rd check, **stop**: tell the
   seller it's taking longer than usual, give them the `operationId`, and offer to check
   back later (re-poll on a later turn). Never poll forever.
4. Only advance to the next stage after confirmed success.

Tell the seller what you're doing: *"Submitted — checking status… done, moving on."*

## Workflow (Pipeline + Inversion — sequential, ask at every gate)

### Step 1 — Create the inbound plan
Gather source address, destination marketplace, and items (MSKU + quantity, and prep/
label owner). Call `fbaInbound_createInboundPlan`, poll status, note the `inboundPlanId`.

### Step 2 — Set packing information
Gather boxes, what's in each box, dimensions, and weight. Call
`fbaInbound_setPackingInformation` (`BOX_CONTENT_PROVIDED` for pack-first), poll.
> **Ordering matters:** set packing info **before** generating placement options. If you
> change box info **after** placement is generated, you must **regenerate placement
> options** (the split depends on the boxes). Use `fbaInbound_listShipmentItems` to pull a
> pick list for the boxes once shipments exist.

### Step 3 — Pack option (decision gate)
`fbaInbound_generatePackingOptions` → poll → `fbaInbound_listPackingOptions`. Present
the grouping choices and **wait** for the seller's pick, then
`fbaInbound_confirmPackingOption`. (Confirm within 24–72h or the groups expire.)

### Step 4 — Placement option (PERMANENT, fee-charging — HARD GATE)
`fbaInbound_generatePlacementOptions` → poll → `fbaInbound_listPlacementOptions`.
Present each option with its FC split and its placement **fee in dollars** — **show the
fee before asking the seller to choose** (never auto-pick the cheapest). The inbound API
has **no preview/dry-run**, and `fbaInbound_confirmPlacementOption` is **permanent and
charges the placement fee** — so it sits behind a hard gate:
1. The seller **explicitly picks** one option.
2. You **restate** the exact choice + the **exact dollar fee** + that it's permanent.
3. The seller types **`CONFIRM`** — a plain "ok"/"yes" is **not** enough.

Only after the typed `CONFIRM` do you call `fbaInbound_confirmPlacementOption`.

### Step 5 — Transportation option (decision gate)
First **collect the seller's ready-to-ship date and confirm the ship-from address** —
`fbaInbound_generateTransportationOptions` needs the confirmed `placementOptionId`,
`shipmentId`, `readyToShipDate`, and `shipFromAddress`. Then generate → poll →
`fbaInbound_listTransportationOptions`. Present carrier / mode / **quote in dollars**
(Amazon Partnered SPD or LTL, or your own carrier) — **show the cost before asking**.
- **Carrier mixing:** shipments don't have to share one carrier — you *can* mix (e.g.
  SPD + LTL, or PCP + non-PCP) **only when** the selections are on **different shipping
  modes** and **all shipments are PCP-eligible**; otherwise mixing errors with
  `FBA_INB_0354`. (Shipping modes go beyond SPD/LTL — see
  [carriers-and-shipping.md](references/carriers-and-shipping.md).)
- **Non-partnered ordering:** for a non-partnered carrier you must **confirm the delivery
  window (Step 6) BEFORE** `fbaInbound_confirmTransportationOptions`.

`fbaInbound_confirmTransportationOptions` **locks the carrier and the quoted cost** and has
no preview, so apply the **same hard gate** as placement: explicit pick → restate carrier +
**exact dollar quote** → seller types **`CONFIRM`** → only then confirm. See
[carriers-and-shipping.md](references/carriers-and-shipping.md) for post-confirm steps,
void windows, and tracking.

### Step 6 — Delivery window & post-confirmation
Run the delivery-window generate → list → confirm
(`fbaInbound_generateDeliveryWindowOptions` → `…listDeliveryWindowOptions` →
`…confirmDeliveryWindowOptions`). It's **mandatory for non-partnered carriers** (confirm it
**before** booking the FC appointment and **before** confirming transportation), and also
applies when enrolled in the confirmed-delivery-window program. Then guide the
carrier-specific finish: partnered SPD (schedule UPS pickup), partnered LTL (BOL,
freight-ready within 2 weeks), or non-partnered (collect tracking →
`fbaInbound_updateShipmentTrackingDetails`).

### Step 7 — Summarize
Recap the plan: stage reached, shipments and destinations, carrier, and the single
next action the seller must take.

## Special workflows & extras
- **Pack Later (unknown carton)** and **India (IN) marketplace** differ — see
  [special-workflows.md](references/special-workflows.md).
- **Labels, prep, content updates, modify/cancel, inspection** — see
  [tools-and-additional.md](references/tools-and-additional.md).

## Guardrails
- **Ask at every decision gate.** Present packing/placement/transport/delivery-window
  options and **wait for the seller's choice** — never auto-pick.
- **Irreversible / fee-charging confirms sit behind a hard typed gate.** The inbound API
  has **no preview mode**, so for `fbaInbound_confirmPlacementOption` (permanent + placement
  fee) and `fbaInbound_confirmTransportationOptions` (locks carrier + quoted cost): show the
  **exact dollar amount**, require an **explicit pick**, and require the seller to type
  **`CONFIRM`** — never proceed on a vague "ok". Match the gate to the blast radius (a
  permanent, money-charging action gets the strongest gate).
- **Cancellation needs confirmation.** Before `fbaInbound_cancelInboundPlan`, warn that
  it voids all shipments and may incur charges outside the void window (24h SPD / 1h
  LTL); do **not** cancel without an explicit confirmation.
- **Always poll after a write — but bounded.** Treat an `operationId` as pending, not done;
  confirm `SUCCESS` before proceeding, surface `FAILED` honestly, and **cap polling at 3
  attempts with exponential backoff** (don't loop forever — hand off the `operationId` and
  offer to check back).
- **Never invent data.** Use only addresses, quantities, and tracking the seller gives
  you and only options the tools return. No secrets/PII.

## References
- [carriers-and-shipping.md](references/carriers-and-shipping.md) — partnered SPD/LTL vs
  your own carrier, void windows, BOL/pallets, tracking.
- [special-workflows.md](references/special-workflows.md) — Pack Later (LTL only) and the
  India FC workflow (compliance + self-ship appointments).
- [tools-and-additional.md](references/tools-and-additional.md) — labels, prep, delivery
  windows, content updates, modify/cancel, full tool quick-reference, errors & constraints.
