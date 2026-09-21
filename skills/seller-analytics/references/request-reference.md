# Analytics request & response reference

How to build `analytics_getMetricMetadata` and `analytics_getMetricData` calls correctly.
The single most common failure is a malformed `getMetricData` request — read this before querying.

## `analytics_getMetricMetadata`

Discovers what metrics exist. Call it **first**.

**Input**
- `marketplaceIds` (required) — array with one marketplace ID, e.g. `["ATVPDKIKX0DER"]`.
- `entityId` (required) — the seller's Merchant Token / MCID.

**Reading the response.** The response returns domains, each with columns:

| Field | How to use |
|-------|-----------|
| `metricId` | Use in the `metrics[]` array of `getMetricData` |
| `metricName` | Human-readable name of the metric |
| `businessDefinition` | What the metric measures — use it to confirm intent |
| `dataType` | `DOUBLE`, `STRING`, or `ZONED_TIMESTAMP` |
| `aggregationType` | How values combine — `SUM`, `AVG`, `MAX` |
| `isGroupable` | `true` = dimension (use in `groupBy` / filter); `false` = measure (use in `metrics[]`) |

To find the right metric: browse the domain (`Threep_Inventory`, `Threep_Sales`, `Threep_Traffic`),
look at columns where `isGroupable=false` for measures, and read `businessDefinition` to confirm.

## `analytics_getMetricData`

Retrieves values. Build the request carefully — the API is strict.

**Required parameters**
- `marketplaceIds` — array with one marketplace ID, e.g. `["ATVPDKIKX0DER"]`.
- `entityId` — the seller's merchant customer ID.
- `metrics` — **array of objects** with a `name` field, e.g. `[{"name": "SESSION_CNT"}]`.
  **Not** an array of strings.
- `startDate` — inclusive, `YYYY-MM-DD`.
- `endDate` — inclusive, `YYYY-MM-DD`.
- `groupableColumnFilter` — **must always include a `MARKETPLACE_ID` filter**, even though
  `marketplaceIds` is also passed:
  ```json
  { "operator": "EQUALS", "column": "MARKETPLACE_ID", "value": {"stringValue": "ATVPDKIKX0DER"} }
  ```
- `dateGranularity` — `DAY | WEEK | MONTH | QUARTER | YEAR` (splits results into time buckets).
  **Effectively required:** omitting it returns empty results. The Swagger model lists it as
  optional today, but always send it. (Being made required in the model.)

**Optional parameters**
- `groupBy` — array of groupable column names, e.g. `["ASIN"]` for a per-product breakdown.
- `pageSize` — 1–1000 (default 50).
- `paginationToken` — from the previous response's `pagination.nextToken`.
- `currency` — ISO 4217 code (e.g. `USD`); **required for financial metrics**.

### Multiple metrics in one call
```json
{ "metrics": [
    {"name": "FBA_INV_VIS_ONHAND_PMEAN_DOS"},
    {"name": "FBA_UNITS_SHIPPED_T30"}
] }
```

### Filtering to a specific ASIN — combine with `AND` + `children`
```json
{ "groupableColumnFilter": {
    "operator": "AND",
    "children": [
      {"operator": "EQUALS", "column": "MARKETPLACE_ID", "value": {"stringValue": "ATVPDKIKX0DER"}},
      {"operator": "EQUALS", "column": "ASIN",           "value": {"stringValue": "B0FN71SQ8R"}}
    ]
} }
```

### Response structure
```json
{
  "results": [
    { "startDate": "2026-06-01", "endDate": "2026-06-07",
      "metrics": [
        { "groupByKey": [ {"name": "ASIN", "value": "B0FN71SQ8R"} ],
          "metrics":    [ {"name": "FBA_INV_VIS_ONHAND_PMEAN_DOS", "value": 45.2} ] } ] }
  ],
  "pagination": { "nextToken": "..." }
}
```

## Gotchas (these cause most failures)
- **`MARKETPLACE_ID` filter is mandatory** in `groupableColumnFilter` on every `getMetricData`
  call — required even though `marketplaceIds` is a top-level param.
- **`metrics` must be objects** (`{"name": "..."}`), not strings.
- **`entityId` is not auto-populated** by the gateway — always pass it explicitly.
- **Pagination field names differ:** the request field is `paginationToken`; read its value
  from the response's `pagination.nextToken`.
- **Data is delayed ~2 days** from the current date — don't expect today's or yesterday's numbers.
- **Empty results?** Widen the date range, or drop `groupBy` to get an aggregated total.
- **Discover first.** If unsure of a metric ID, call `analytics_getMetricMetadata` — never guess.

## Worked examples

**"What are my sessions this month by ASIN?"**
```json
{ "marketplaceIds": ["ATVPDKIKX0DER"], "entityId": "<seller_merchant_id>",
  "metrics": [{"name": "SESSION_CNT"}],
  "startDate": "2026-07-01", "endDate": "2026-07-31",
  "dateGranularity": "WEEK", "groupBy": ["ASIN"],
  "groupableColumnFilter": {"operator": "EQUALS", "column": "MARKETPLACE_ID", "value": {"stringValue": "ATVPDKIKX0DER"}} }
```

**"Am I at risk of any stockouts?"** — analysis rule: days of supply < 14 → at risk.
```json
{ "marketplaceIds": ["ATVPDKIKX0DER"], "entityId": "<seller_merchant_id>",
  "metrics": [{"name": "FBA_INV_VIS_ONHAND_PMEAN_DOS"}, {"name": "FBA_UNITS_SHIPPED_T30"}],
  "startDate": "2026-07-01", "endDate": "2026-07-10", "dateGranularity": "DAY", "groupBy": ["ASIN"],
  "groupableColumnFilter": {"operator": "EQUALS", "column": "MARKETPLACE_ID", "value": {"stringValue": "ATVPDKIKX0DER"}} }
```

**"Show me inventory for a specific ASIN"**
```json
{ "marketplaceIds": ["ATVPDKIKX0DER"], "entityId": "<seller_merchant_id>",
  "metrics": [{"name": "FBA_AVAILABLE_QTY"}, {"name": "FBA_INV_VIS_BOUND_QTY"}, {"name": "FBA_INV_INBOUND_TOTAL"}],
  "startDate": "2026-07-01", "endDate": "2026-07-10", "dateGranularity": "DAY", "groupBy": ["ASIN"],
  "groupableColumnFilter": {"operator": "AND", "children": [
    {"operator": "EQUALS", "column": "MARKETPLACE_ID", "value": {"stringValue": "ATVPDKIKX0DER"}},
    {"operator": "EQUALS", "column": "ASIN",           "value": {"stringValue": "B0FN71SQ8R"}}
  ]} }
```
