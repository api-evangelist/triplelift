---
name: triplelift-async-report-export
description: >-
  Export a large TripleLift publisher or CTV report asynchronously as CSV, either by
  polling a pre-signed S3 link or by having TripleLift email it, including the
  expiry, size and parsing rules that make the CSV path fail in practice.
api: TripleLift Reporting API
endpoint: https://reporting-api.triplelift.net/graphql
protocol: graphql
operations:
  - asyncDownloadPublisherNetworkReport
  - asyncDownloadCTVPublisherNetworkReport
  - asyncDownloadReportStatus
  - asyncEmailPublisherNetworkReport
  - asyncEmailCTVPublisherNetworkReport
generated: '2026-08-12'
method: generated
source: https://supply-docs.triplelift.com/reference/asynchronous
grounding: >-
  Operation names, argument names, status strings and every limit are taken verbatim
  from TripleLift's asynchronous-endpoints documentation. The schema was not
  introspected — it is credential-gated.
---

# Export a large TripleLift report as CSV

Use this whenever a report could exceed the synchronous **5,000-row** limit: wide date ranges, many dimensions, or any query including the high-cardinality `DOMAIN` dimension. Asynchronous results are returned **within 5 minutes**.

Auth is identical to the synchronous path — `X-API-Key` **and** `Authorization: Bearer <JWT>`, both required.

## Choose a delivery mode

| Mode | Operation | Returns | Use when |
|---|---|---|---|
| Download | `asyncDownloadPublisherNetworkReport` | pre-signed S3 URL | a program consumes the file |
| Download (CTV) | `asyncDownloadCTVPublisherNetworkReport` | pre-signed S3 URL | CTV reporting |
| Email | `asyncEmailPublisherNetworkReport` | boolean | a human wants it in their inbox |
| Email (CTV) | `asyncEmailCTVPublisherNetworkReport` | boolean | CTV, to a human |

All four take the **same arguments as `publisherNetworkReport` minus `cursor` and `size`** — there is no pagination on the async path. The email variants add an `emails` list.

## Option 1 — download and poll

```graphql
query {
  asyncDownloadPublisherNetworkReport(
    sellerMemberId: "YOUR_MEMBER_ID"
    startDate: "2026-06-15"
    endDate: "2026-07-30"
    dimensions: [YMD, PUBLISHER_NAME]
    metrics: [AD_REQUESTS, REVENUE]
  )
}
```

The response is a pre-signed S3 URL — returned **immediately**, before the report exists.

Poll with the link you were given:

```graphql
query {
  asyncDownloadReportStatus(url: "<the pre-signed url>")
}
```

Three possible statuses:

- `READY` — the report is at the download link
- `WAITING` — not ready yet, keep polling
- `ERROR` — the report failed; resubmit the original request

**The pre-signed URL expires 30 minutes after it is returned.** That clock starts when you submit, not when the report is ready, so your polling window and your download window share the same 30 minutes.

TripleLift does not publish a poll interval — their docs defer to the schema's status refresh rate. Remember you are still bound by **1 request per 5 seconds** and **1,000 requests per day**: polling every 5 seconds for the full 5-minute SLA costs 60 requests, 6% of your daily allowance, per report. Poll at 15–30 second intervals instead, and treat a 429 (with `retry-after` in the **body**, not a header) as a signal to back off, not as a failure.

## Option 2 — email delivery

```graphql
query {
  asyncEmailPublisherNetworkReport(
    sellerMemberId: "YOUR_MEMBER_ID"
    startDate: "2026-06-15"
    endDate: "2026-07-30"
    dimensions: [YMD, PUBLISHER_NAME]
    metrics: [AD_REQUESTS, REVENUE]
    filters: []
    emails: ["ops@example.com"]
  )
}
```

Returns `true` if the request was accepted — **not** that the email was delivered. Emails over **10 MB** fail to send, and you will not learn that from the return value. For anything large, use the download path.

## Hard limits

- CSV reports larger than **400 MB** fail outright.
- The pre-signed URL lives **30 minutes**.
- Email payloads over **10 MB** fail silently from the caller's point of view.

If a report is likely to exceed 400 MB, split it — narrow the date range, drop a dimension, or add filters. Filters and fewer dimensions reduce size much more than dropping metrics, because metrics add columns while dimensions add rows.

## Parse the CSV carefully

TripleLift's CSV is not RFC 4180-strict. Their documented rules:

- fields may be **null**
- fields are **unquoted** unless a string contains a quote or a delimiter
- an embedded quote is doubled, and only then is the field quoted

```
example".com   =>   "example"".com"
example,.com   =>   "example,.com"
example\.com   =>   example\.com      (unchanged — backslash is not an escape)
```

A parser configured for always-quoted fields, or one that treats backslash as an escape character, will corrupt domain values.

## Still applies from the synchronous path

- All data is **UTC**, all money is **USD**.
- Any query including `DOMAIN` silently rolls sub-0.1%-of-renders rows into a single `"Other"` row unless you pass `useThreshold`.
- Console labels and API field names differ — Console "Ad Requests" is `IMPRESSIONS`, Console "Filled Impressions" is `RENDERED`.
