# Carriers & Shipping

How transportation works after you confirm a placement option, and what the seller
must do per carrier type. From the Fulfillment Inbound v2024-03-20 guides.

## The carrier choice (at the transportation gate)

`listTransportationOptions` returns options with a `shippingSolution`:

- **`AMAZON_PARTNERED_CARRIER` (PCP)** — Amazon-negotiated rates; quote shown in the option.
  - **SPD (Small Parcel Delivery)** — typically UPS; one tracking per box.
  - **LTL (Less-Than-Truckload / freight)** — palletized; carrier provides a BOL.
- **`USE_YOUR_OWN_CARRIER` (Non-Partnered)** — the seller arranges and pays for shipping.

### Shipping modes (beyond SPD/LTL)
Options can carry more modes than just SPD and LTL: **Ground small parcel (SPD)**,
**LTL freight**, **Full truckload (FTL)** — palletized and non-palletized, **LCL ocean**,
**FCL ocean**, **Air small parcel**, **Air express**. Present whatever the option list
returns; don't assume only SPD/LTL exist.

### Mixing carriers across shipments
It's **not** strictly "one carrier per plan." You *can* mix carrier selections across
shipments (e.g. **SPD + LTL**, or **PCP + non-PCP**) **when** the selections are on
**different shipping modes** *and* **all shipments are PCP-eligible**. Outside those
conditions, mixing returns error `FBA_INB_0354`.

> **Non-partnered carriers require a confirmed delivery window.** For a non-partnered
> shipment you **must confirm the anticipated delivery window before booking the FC
> appointment** — and before `confirmTransportationOptions`. (For partnered carriers it's
> only needed when enrolled in the confirmed-delivery-window program.)

## Amazon Partnered Carrier — SPD

- Quote is in the transportation option.
- **Void window: 24 hours** after confirmation (charges may apply after).
- Amazon does **not** schedule pickup — the **seller schedules pickup with UPS** directly.
- Tracking is provided automatically by the partnered carrier.

Tell the seller after confirm: schedule the UPS pickup; you have 24h to cancel free.

## Amazon Partnered Carrier — LTL

- **Freight-ready date must be within 2 weeks** of workflow creation.
- **Pallet information is required** — check `listInboundPlanPallets` / `listShipmentPallets`.
- The carrier provides a **BOL (Bill of Lading)**.
- **Void window: 1 hour** after confirmation (charges may apply after).
- India PCP: a delivery challan is available via `getDeliveryChallanDocument`.

## Non-Partnered Carrier (your own)

After confirming transportation, the seller ships and provides tracking; then call
`updateShipmentTrackingDetails`:

- **SPD** — a tracking number per box.
- **LTL/FTL** — the freight bill number or BOL number (one per shipment).

Constraints: only works for shipments created via these API tools (not Seller
Central–created), and **one freight bill number per shipment**.

## Cancellation windows (quote before cancelling)

| Carrier | Void window | After the window |
|---------|-------------|------------------|
| Amazon Partnered SPD | 24 hours | Charges apply |
| Amazon Partnered LTL | 1 hour | Charges apply |
| Non-Partnered | Any time | No charges |

## Source address change gotcha

`updateShipmentSourceAddress` **invalidates existing transportation options** — the
seller must regenerate and reconfirm transportation afterward.
