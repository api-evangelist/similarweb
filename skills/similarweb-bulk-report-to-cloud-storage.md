---
name: Run a bulk Similarweb report into cloud storage
description: Price, submit, track and collect a Similarweb Batch API report of up to one million domains, delivered to Amazon S3, Google Cloud Storage or Snowflake.
api: openapi/_original/similarweb-batch-api-openapi.yml
generated: '2026-08-13'
method: generated
operations:
  - describeTables
  - getBatchCredits
  - createS3Integration
  - createGcsIntegration
  - getAllIntegrations
  - validateRequest
  - requestReport
  - getRequestStatus
  - getReportHistory
  - retryRequest
---

# Bulk report to cloud storage

Use this instead of looping the REST API whenever the job covers more than a few hundred
domains. The Batch API takes up to **one million domains per request** and delivers
asynchronously; a REST loop over the same list would burn the 10 rps limit and cost more.

## Order of operations

1. **Find the dataset.** `describeTables` — `GET /batch/v4/tables-describe`. Returns
   `TableDescription` rows: `vtable` (the dataset key), `description`, available `metrics`,
   supported `filters`, `min_date` and `granularities`. **`vtable` is the discovery key for
   this whole surface** — do not guess dataset names, read them from here, and check
   `min_date` before choosing a start date.
2. **Check the balance.** `getBatchCredits` — `GET /v3/batch/credits`. Batch jobs are large;
   confirm the credits exist before submitting.
3. **Set up the destination once.** Reports do not deliver to a URL you pass at submit time —
   they deliver to a named, pre-registered integration.
   - `getAllIntegrations` — `GET /batch/v4/integrations` to see what already exists.
   - `createS3Integration` — `POST /batch/v4/s3-integration` with `integration_name`,
     `bucket_name`, `region`, `prefix`.
   - `createGcsIntegration` — `POST /batch/v4/gcs-integration` with `integration_name`,
     `bucket_name`, `prefix`.
   - Snowflake integrations are also supported; see the provider's Snowflake guide.
   Reuse the `integration_name` in the report's `delivery_information.delivery_method_params`.
4. **Price it before you run it.** `validateRequest` — `POST /batch/v4/request-validate` with
   the same body you intend to submit. It returns `ValidateResponse` with `valid`,
   `estimated_cost` (**in data credits, not currency**) and `errors[]`. Skipping this step is
   the most expensive mistake available on this API. Always run it on a job of this size, and
   surface `estimated_cost` to the human before submitting.
5. **Submit.** `requestReport` — `POST /batch/v4/request-report`. Body is a `ReportRequest`:
   - `report_query.tables[]` — one `TableQuery` per dataset: `vtable`, `granularity`,
     `start_date`, `end_date`, optional `latest` / `all_history` / `window_size`, `filters`,
     `metrics`, and `paging` (`limit`, `offset`, `sort`, `sort_asc`).
   - `delivery_information` — `delivery_method`, `response_format` (e.g. `csv`),
     `webhook_url` (optional), and `delivery_method_params` (`integration_name`,
     `table_name`, `retention_days`, `num_of_files`, `write_mode`).
   Returns `ReportSubmitResponse` with a `report_id`. **Persist the `report_id`** — it is the
   only handle you have on the job.
6. **Track it.** `getRequestStatus` — `GET /batch/v4/request-status`. Status is one of
   `processing`, `complete`, `internal_error`. Poll with backoff, or skip polling entirely by
   attaching a webhook (see the `similarweb-webhook-driven-report-delivery` skill).
   `getReportHistory` — `GET /batch/v4/report-history` lists past jobs if the id was lost.
7. **On `internal_error`, retry server-side.** `retryRequest` — `POST /batch/v4/retry-request`
   resubmits an existing job. Prefer it over rebuilding and resubmitting a fresh request:
   it is the provider's own retry path and avoids duplicating a large job.
8. **Collect** the output from the S3 bucket / GCS bucket / Snowflake share you registered in
   step 3, honouring the `retention_days` you set.

## Rules while running

- **The Batch 429 is not the REST 429.** On Batch, HTTP 429 means you exceeded the number of
  allowed *pending* jobs — it is a concurrency limit, not a per-second limit. Do not fix it by
  slowing your request rate; fix it by letting jobs finish.
- **400 is usually a date.** The most common Batch 400 is a `start_date` earlier than the
  dataset's history. The error message names the earliest available period — read it and
  clamp, and cross-check against `min_date` from `describeTables`.
- **There is no idempotency key.** Similarweb documents none. A resubmitted `requestReport`
  is a *new* job that spends credits again. Deduplicate on your side by keying on the query
  you sent, and use `retryRequest` with the existing `report_id` rather than resubmitting.
- Errors are not RFC 9457 problem+json; parse defensively.

## Version warning

These paths are `/batch/v4/*` and `/v3/batch/*`. API V5 is live, uses a single API key across
REST and Batch, and publishes `/v5/batch/*` and `/batch/v5/*` paths. Legacy REST endpoints
sunset **2026-10-06** and no runtime `Sunset` header is emitted. See
`lifecycle/similarweb-lifecycle.yml`.
