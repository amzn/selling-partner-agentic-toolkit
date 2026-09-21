# Special Workflows: Pack Later & India

Two variants of the standard pipeline. Detect them up front and adjust.

## Pack Later (unknown carton contents — LTL only)

**When:** the seller doesn't yet know exact box contents at plan-creation time.
**SPD is NOT available** for Pack Later — LTL (freight) only.

How it differs from pack-first:

| Step | Pack First | Pack Later |
|------|-----------|-----------|
| `setPackingInformation` | full box contents (items per box) | box **dimensions/weight only**, no items |
| `contentInformationSource` | `BOX_CONTENT_PROVIDED` | `BARCODE_2D` or `MANUAL_PROCESS` |
| When items are assigned to boxes | before placement | **after** placement is confirmed |
| Shipping mode | SPD or LTL | **LTL only** |

Sequence: `createInboundPlan` → generate/list/confirm placement →
`setPackingInformation` (dims/weight only) →
generate/list/confirm transportation → **then** the seller provides actual box contents
later (Seller Central scan & pack, Excel upload, or web form).

Tell the seller: this uses the Pack Later (LTL) flow; you'll set boxes with
dimensions/weight now and add item-level contents later in Seller Central.

## India marketplace (`A21TJRUUN4KGV`)

India has extra requirements and one omission:

- **Skip `generatePackingOptions`** — it is **not supported** for India.
- **Compliance must be managed:**
  - `listItemComplianceDetails` — check what's needed per MSKU.
  - `updateItemComplianceDetails` — set the compliance info.
- **Self-ship appointments are required for FC drop-off:**
  - `generateSelfShipAppointmentSlots` — get slots (provide a date range).
  - `getSelfShipAppointmentSlots` — retrieve the generated slots.
  - `scheduleSelfShipAppointment` — book a slot (decision gate — let the seller pick).
  - `cancelSelfShipAppointment` — cancel if needed (requires a reason comment).
- **Delivery challan:** `getDeliveryChallanDocument` for PCP shipments.

Tell the seller: for an India FC we'll check compliance first, skip packing-option
generation, and after transportation is set, book a drop-off appointment.
