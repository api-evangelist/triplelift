---
name: triplelift-publisher-reporting
description: >-
  Pull a paginated publisher performance report from the TripleLift Reporting API
  (GraphQL), handling the dual-header auth, the 1-request-per-5-seconds throttle,
  cursor pagination and the two silent-truncation traps.
api: TripleLift Reporting API
endpoint: https://reporting-api.triplelift.net/graphql
protocol: graphql
operations:
  - publisherNetworkReport
  - publisherNetworkFilterOptions
generated: '2026-08-12'
method: generated
source: https://supply-docs.triplelift.com/reference/synchronous
grounding: >-
  Operation names, argument names, response field names and every numeric limit
  below are read verbatim from TripleLift's published documentation and from the
  cURL example TripleLift ships. No schema was introspected — introspection on this
  endpoint requires credentials — so argument sets are the documented ones, not the
  complete ones.
---

# Pull a TripleLift publisher performance report

## Before you start

You need three things, all obtained from the [TripleLift Console](https://console.triplelift.com) under **Reporting → Reporting API**:

- an **API key** (`X-API-Key`) — static, never expires, but rotating it invalidates it for every user and program on the member account
- a **JWT** (`Authorization: Bearer`) — expires after **one month**
- your **member ID** — the `sellerMemberId` argument

Both headers are required on every request. Sending only one returns HTTP 401 with `{"message":"X-API-Key header is required"}`.

To refresh the JWT programmatically:

```
POST https://api.triplelift.com/login
Content-Type: application/json

{"username": "...", "password": "..."}
```

The token comes back as `reporting_api_token`.

## Step 1 — decide synchronous or asynchronous

`publisherNetworkReport` has a **hard 5,000-row limit**. If your date range × dimension cardinality could exceed that, do not use this skill — use `triplelift-async-report-export` instead. The synchronous endpoint does **not** error when you exceed the limit; it silently returns the first 5,000 rows.

Row count grows with the number of dimensions and their cardinality, not with the number of metrics. `DOMAIN` is the highest-cardinality dimension and can multiply row counts by 100x or more.

## Step 2 — resolve filter values first (optional)

If you are filtering, call `publisherNetworkFilterOptions` for the dimension and time period first. The IDs it returns are the values `publisherNetworkReport` expects in its `filters` argument. Do not construct filter values by hand.

## Step 3 — issue the query

```graphql
query {
  publisherNetworkReport(
    sellerMemberId: "YOUR_MEMBER_ID"
    startDate: "2026-08-01"
    endDate: "2026-08-07"
    dimensions: [YMD, PUBLISHER_NAME]
    metrics: [IMPRESSIONS, REVENUE]
  ) {
    rows {
      dimensions { name value }
      metrics {
        ... on MetricLong { name long: value }
        ... on MetricDecimal { name decimal: value }
      }
    }
    nextCursor
    totalRows
  }
}
```

`metrics` is a union — you must spread both `MetricLong` and `MetricDecimal` or you will lose values silently.

Send it as a normal GraphQL POST:

```
POST https://reporting-api.triplelift.net/graphql
X-API-Key: <key>
Authorization: Bearer <jwt>
Content-Type: application/json
```

## Step 4 — paginate

Default page size is **50**. Set `size` to raise it (never above the 5,000 row limit).

When `nextCursor` is present, pass it back as `cursor` on the next request — **and resubmit every other argument, in the same order you originally passed them.** TripleLift documents argument order as significant for cursor continuation. If you rebuild the query from a parsed AST or a map, order can change and pagination will break.

Stop when `nextCursor` is absent. Cross-check the total rows you collected against `totalRows`.

## Step 5 — respect the throttle

- **1 request per 5 seconds**, per API key
- **1,000 requests per day**, per API key

Sleep at least 5 seconds between pages. At the daily cap, a naive loop burns the entire allowance in about 83 minutes.

On exhaustion you get HTTP **429** with this body:

```json
{"path":"/graphql","retry-after":2,"error":"Forbidden","message":"Too many requests, please wait before trying again","timestamp":"...","status":429}
```

There is **no `Retry-After` header** and no `X-RateLimit-*` headers. Parse `retry-after` out of the JSON body and sleep that many seconds. Do not branch on the body's `"error":"Forbidden"` string — the request was throttled, not forbidden.

## Step 6 — check for the two silent traps

1. **Row-limit truncation.** If `totalRows` exceeds what you collected and `nextCursor` is gone, you were truncated. Re-run asynchronously.
2. **Domain threshold rollup.** If you asked for `DOMAIN`, every domain contributing under **0.1% of total renders** was collapsed into one row literally named `"Other"`. Look for it. Pass `useThreshold` to opt out — and expect a much larger result if you do.

## Reading the numbers

- All timestamps and dates are **UTC**. All money is **USD**.
- Current-day data is not real time; the first hour of a day is generally available by 10:00 UTC.
- Daily data goes back 15 months; hourly data only to 00:00 UTC yesterday.
- Field names do **not** match the Console labels. Most dangerously: Console "Ad Requests" is API `IMPRESSIONS`, and Console "Filled Impressions" is API `RENDERED`. Never assume a shared vocabulary between the UI and the API.

## When the schema disagrees with this skill

TripleLift's docs repeatedly defer to the GraphQL schema as the source of truth for available dimensions and metrics, and the schema is only browsable through their hosted Altair instance at `https://reporting-api.triplelift.net/altair` with the same auth headers injected. If a dimension here is rejected, check Altair — dimension names have changed before (`dsp_seat_id` became `dsp_seat` in November 2025, with no deprecation window).
