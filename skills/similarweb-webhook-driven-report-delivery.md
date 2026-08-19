---
name: Get notified when a Similarweb report is ready
description: Validate a webhook endpoint, subscribe it, and attach it to a Batch report so report-status changes are pushed instead of polled.
api: openapi/_original/similarweb-batch-api-openapi.yml
generated: '2026-08-13'
method: generated
operations:
  - testWebhook
  - subscribeWebhook
  - listWebhookSubscriptions
  - unsubscribeWebhook
  - requestReport
  - getRequestStatus
---

# Webhook-driven report delivery

Batch reports are asynchronous. Polling `getRequestStatus` works but wastes calls against the
10 rps budget; a webhook removes the poll loop entirely. This is the only event surface
Similarweb publishes — there is no event stream for the REST measurement data.

## Steps

1. **Stand up an endpoint** in your application that accepts an HTTP `POST` with a JSON body.
   It must respond within **5 seconds**; a slower response is treated as a delivery failure.
2. **Validate it before you rely on it.** `testWebhook` —
   `POST /batch/v4/webhooks/test` (the V5 docs show this at `/v3/batch/webhooks/test`) with:
   ```json
   { "webhook_url": "https://example.com/hooks/similarweb" }
   ```
   Similarweb immediately POSTs back:
   ```json
   { "event_type": "test_webhook_endpoint", "payload": "Webhook Integration with Similarweb is successful!" }
   ```
   Confirm in your own system that the `test_webhook_endpoint` event arrived. Do not proceed
   until it does.
3. **Subscribe it.** `subscribeWebhook` — `POST /batch/v4/webhooks/subscribe` with a
   `WebhookSubscribeRequest` (`webhook_url`, `events`, `secret`). Returns a
   `WebhookSubscription` with a `webhook_id`.
4. **Attach it to a job.** Put the validated URL in `delivery_information.webhook_url` on the
   `requestReport` body. Similarweb POSTs when the report status changes.
5. **Handle the states.** Report status is one of `processing`, `complete`, `internal_error`.
   On `complete`, collect the output from the configured S3 / GCS / Snowflake destination. On
   `internal_error`, call `retryRequest` with the existing `report_id`.
6. **Housekeep.** `listWebhookSubscriptions` — `GET /batch/v4/webhooks/list` to audit what is
   registered. `unsubscribeWebhook` — `DELETE /batch/v4/webhooks/unsubscribe` to remove a
   stale endpoint. Leaving a dead subscription registered means silent delivery failures.

## Rules while running

- **You cannot cryptographically verify a delivery.** `subscribeWebhook` accepts a `secret`,
  but Similarweb publishes no signature header and no verification algorithm. Treat the
  webhook as an untrusted *hint*, not as data: on receipt, call `getRequestStatus` with the
  `report_id` to confirm the state before acting on it. Do not accept report content from
  the webhook body.
- **The report-status payload schema is not documented.** Only the `test_webhook_endpoint`
  payload is published. Parse the delivered body defensively and key off the `report_id`
  rather than assuming a field layout.
- **No delivery-retry policy is documented.** Assume at-most-once. Keep a reconciliation
  sweep that polls `getRequestStatus` for any job that has not reported in within its
  expected window — the webhook is an optimisation, not a guarantee.
- Deliveries are not idempotent-keyed. Make your handler idempotent on `report_id` + status.
