# Velocity source: `sales_getOrderMetrics` (not the Reports API)

The original design assumed sales velocity would come from the **Reports API**
(`GET_SALES_AND_TRAFFIC_REPORT`), which is **asynchronous**: create report → poll
until done → download. That adds minutes of latency and polling complexity.

When we enumerated the live Amazon Selling Partner Connector tools (`tools/list`), there is **no
`reports` tool** exposed — but there **is** `sales_getOrderMetrics`, a **synchronous**
call that returns aggregated units/orders/sales for an interval, broken down by a
granularity, and filterable by `sku` or `asin`.

**Decision:** use `sales_getOrderMetrics` for this skill.
- Synchronous → meets the <30s sync-skill latency target (no polling).
- Directly gives `unitCount` per interval → trivial to derive daily velocity.

**Usage notes**
- `interval` is an ISO-8601 range like `2026-05-12T00:00:00-07:00--2026-06-11T00:00:00-07:00`.
- `granularity: Total` for a single 30-day number; `Day` for a daily series.
- `sku` and `asin` are **mutually exclusive** — pass only one.
- `marketplaceIds` and `entityId` are required.

daily velocity = `unitCount / days_in_interval`; days_of_cover = `fulfillableQuantity / daily_velocity`.
