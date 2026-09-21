# Additional Functionality, Tool Reference, Errors & Constraints

## Additional functionality

**Labels (FNSKU):** `createMarketplaceItemLabels` — ask for marketplace, MSKUs +
quantities, label type, page type, locale. For shipment/box labels via `getLabels`, the
`shipmentId` you pass **must be the `shipmentConfirmationId`** (returned after
`confirmPlacementOption`) — **not** the pre-confirm v2024-03-20 `shipmentId`.

**Prep:** `listPrepDetails` (read prep owner SELLER/AMAZON + prep types per MSKU) and
`setPrepDetails` (set who handles prep/labeling). **US note:** from **Jan 1, 2026** sellers
must **prep and label all products themselves** for the US marketplace — set
`prepOwner` / `labelOwner` to SELLER accordingly.

**Delivery windows:** `generateDeliveryWindowOptions` → `listDeliveryWindowOptions` →
`confirmDeliveryWindowOptions` (decision gate). **Mandatory for non-partnered carriers** —
confirm **before** booking the FC appointment and **before** `confirmTransportationOptions`.
For partnered carriers, only when enrolled in the CONFIRMED_DELIVERY_WINDOW program.

**Shipment content updates** (modify box contents AFTER transportation is confirmed):
`generateShipmentContentUpdatePreviews` → `getShipmentContentUpdatePreview` /
`listShipmentContentUpdatePreviews` (shows item changes + cost impact) →
`confirmShipmentContentUpdatePreview` (accepts changes and any added cost — decision gate).

## Modify & cancel operations

| Action | Tool | Notes |
|--------|------|-------|
| Rename plan | `updateInboundPlanName` | any time |
| Rename shipment | `updateShipmentName` | any time |
| Update source address | `updateShipmentSourceAddress` | **invalidates transportation options** — regenerate |
| Update box identifiers | `updateBoxIdentifiers` | custom IDs shown on box labels |
| Update tracking | `updateShipmentTrackingDetails` | non-partnered only, after transport confirmed |
| Cancel plan | `cancelInboundPlan` | voids all shipments; warn about charges/void window first |
| Cancel appointment | `cancelSelfShipAppointment` | India only; requires a reason comment |

## Inspection / reading (read-only)

`getInboundPlan` (top-level status), `listInboundPlans` (all plans), `getShipment`
(full shipment incl. transportation selection), `getInboundOperationStatus` (async
status). Contents: `listInboundPlanItems`/`Boxes`/`Pallets`,
`listShipmentItems`/`Boxes`/`Pallets`, `listPackingGroupItems`/`Boxes`. India:
`listItemComplianceDetails`, `getSelfShipAppointmentSlots`, `getDeliveryChallanDocument`.
Content updates: `getShipmentContentUpdatePreview`, `listShipmentContentUpdatePreviews`.
Use `listShipmentItems` to generate a **pick list** per shipment once placement is confirmed.

## Key constraints

| Constraint | Detail |
|-----------|--------|
| Carrier mixing | Allowed across shipments only on **different shipping modes** with **all shipments PCP-eligible**; otherwise → `FBA_INB_0354` |
| Packing before placement | Set packing info first; **editing box info after placement → must regenerate placement options** |
| Transportation inputs | `generateTransportationOptions` needs `readyToShipDate` + `shipFromAddress` — collect from the seller |
| Delivery window (non-partnered) | **Mandatory** — confirm before the FC appointment and before confirming transportation |
| `getLabels` shipment id | Use the `shipmentConfirmationId`, **not** the pre-confirm `shipmentId` |
| Multiple expiration dates per SKU | **Not supported** on one inbound plan — use separate plans |
| Placement is permanent | Once confirmed, cannot be changed for the plan |
| Confirm packing in 24–72h | Packing groups expire otherwise |
| 500K units per SKU | Max per inbound plan |
| Case-packed not supported | Use Send to Amazon (STA) in Seller Central |
| V0 shipments not accessible | Only v2024-03-20 plans are visible |
| Pack Later = LTL only | SPD not available for unknown-carton flows |
| PCP-LTL freight-ready | Within 2 weeks of workflow creation |
| US prep/label (Jan 1, 2026) | Sellers must self-prep/label for US — set prepOwner/labelOwner = SELLER |
| Source address update | Invalidates transportation options |
| Non-partnered tracking | API-created shipments only; one freight bill per shipment |

## Common errors

| Error | Cause | Resolution |
|-------|-------|-----------|
| `FBA_INB_0354` | Incompatible carrier mix | Mixing is allowed only on different shipping modes with all shipments PCP-eligible; otherwise use one carrier |
| Operation `FAILED` | Async processing error | Read the error message, fix input, retry |
| Pack options not generating | India marketplace | Skip — not supported for IN |
| Cannot update tracking | Shipment created via Seller Central | Only API-created shipments supported |
| Transportation options empty | Source address changed | Regenerate transportation options |
| Placement confirmation fails | Invalid/expired option | Regenerate placement options |

## Seller vs. Vendor

This workflow is for **Sellers (3P) only**. Vendors (1P) use a separate Direct
Fulfillment workflow.
