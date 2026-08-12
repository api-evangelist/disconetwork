---
generated: '2026-08-12'
method: generated
name: Pull DiscoBeat channel reporting
description: Read channel-level and per-publisher performance out of the Disco Reporting
  API, respecting the availability window, the 90-day range cap and the paginated-rows
  vs whole-result-summary split.
api: openapi/disconetwork-reporting-api-v2.yml
operations: [getReportingV2Report, getReportingSummary, getReportingPublishers]
source: >-
  operationIds verified verbatim in openapi/disconetwork-reporting-api-v2.yml and
  openapi/disconetwork-reporting-api-v1.yml; constraints read from those specs and
  from https://disconetwork.com/reporting-api.
---

# Pull DiscoBeat channel reporting

Read-only analytics for a DiscoBeat channel partner. Every request is scoped to the
authenticated channel — there is no account or channel parameter to pass or to get wrong.

## Auth
- Management API key in the `x-api-key` header (`ManagementApiKey` scheme, declared as a
  top-level `security` requirement on both reporting specs).
  See `authentication/disconetwork-authentication.yml`.

## Read the freshness contract before trusting a number
Reporting is **not live**. Every response carries `available_window`, `data_freshness`
and `generated_at`. Check them first. A range outside the window is a `400` with code
`DATE_RANGE_OUTSIDE_AVAILABLE_WINDOW` and the window echoed back — use that echo to
re-clamp and retry rather than walking dates blindly.

## Steps — V2 (current)
1. **Build a report** — `getReportingV2Report` (`GET /discobeat/reporting/v2/report/`).
   Required: `from`, `to`, `metrics` (comma-separated, unique, and only metrics enabled
   for your channel — an unentitled metric is a 400, not a null column).
2. **Choose a grain** — `time_grain` is one of `total`, `day`, `week`, `month`, `hour`
   (default `total`). `hour` supports at most **3 days** and at most **one** breakdown.
3. **Break it down** — `group_by` takes up to **three** of `publisher`, `page_type`,
   `widget_type`, `widget_id` (one only when `time_grain=hour`). Filter with
   `publisher_ids`, `page_types`, `widget_types`.
4. **Page the rows** — `offset` / `limit`. The `summary` block is computed across the
   **complete filtered result**, not the page. Never sum page rows to rebuild a total.

## Steps — V1 (still available)
- `getReportingSummary` (`GET /discobeat/reporting/v1/summary/`) for channel totals over
  `from`/`to`.
- `getReportingPublishers` (`GET /discobeat/reporting/v1/publishers/`) for the
  per-publisher breakdown, with `granularity` (`day`|`hour`), `breakdown`
  (`page_type`|`widget_type`|`widget_id`) and `offset`/`limit` (max **50**).
- Disco states V1 remains available for existing integrations. No Sunset or Deprecation
  header and no dated end-of-life has been published — treat it as superseded, not
  deprecated (`lifecycle/disconetwork-lifecycle.yml`).

## Range and edge-period traps
- Maximum span is **90 days** on both versions; over that is a 400
  (`date_range: ["Date range cannot exceed 90 days."]`).
- On V1, `from` defaults to 6 days before `to` (a 7-day window) and `to` defaults to
  today — so an unparameterised call is a last-week report, not an all-time one.
- On V2 with `time_grain=week` or `month`, the first and last rows are **clipped** to the
  requested range. They are partial periods and must not be compared like-for-like
  against interior rows.

## Empty results
A valid request that matches no data returns `200` with `has_data: false` and zeroed
metrics — not a 404 and not an error. Branch on `has_data`, not on the status code.

## Errors
`400` (validation or availability), `401` (missing/invalid key), `405` (wrong method).
Envelopes in `errors/disconetwork-problem-types.yml`. No `application/problem+json`.

## Rate limits
See `rate-limits/disconetwork-rate-limits.yml` — Disco documents exactly one limit, and
it appears in the changelog rather than in the reference or the specification. No
rate-limit response headers are published.
