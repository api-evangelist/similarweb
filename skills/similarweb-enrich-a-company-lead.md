---
name: Enrich a company lead from its domain
description: Turn a bare company domain into firmographics, digital scale and search demand using the Similarweb lead-enrichment, keywords and rank operations, with credit checks around it.
api: openapi/_original/similarweb-rest-api-openapi.yml
generated: '2026-08-13'
method: generated
operations:
  - getCredits
  - checkCapabilities
  - getLeadEnrichment
  - getWebsiteKeywords
  - getGlobalRank
  - getAppDownloadsAndroid
  - getAppDownloadsIos
---

# Enrich a company lead

Use this when the input is a company domain and the output needs to be a qualified record:
who they are, how big they are digitally, and what demand they capture.

## Steps

1. **Check the budget.** `getCredits` — `GET /v3/batch/credits`. Enrichment endpoints are on
   the expensive end of the credit model; confirm the balance before a bulk pass.
2. **Check entitlement.** `checkCapabilities` —
   `GET /v1/website/{domain_name}/capabilities`. Confirms the plan covers the countries and
   date range you are about to request. Skipping this is how you get application code `103
   Country not available` halfway through a list.
3. **Enrich.** `getLeadEnrichment` —
   `GET /v1/website/{domain_name}/lead-enrichment/all`. This is the one REST response that
   joins firmographics onto web measurement in a single call. `LeadEnrichmentResponse`
   carries: `company_name`, `employee_range`, `estimated_revenue_in_million_usd`,
   `headquarters`, `site_type`, `category`, plus `global_rank`, `visits`, `unique_visitors`,
   `pages_per_visit`, `bounce_rate`, `avg_visit_duration` and `desktop_mobile_share`.
   Prefer this single call over assembling the same picture from six separate endpoints —
   it is fewer calls and fewer credits.
4. **Add search demand (optional).** `getWebsiteKeywords` —
   `GET /v4/website-analysis/keywords`. Returns `keywords_count` plus `keywords[]` of
   `KeywordRecord`: `keyword`, `clicks`, `traffic_share`, `difficulty`, `competition`,
   `primary_intent`, `secondary_intent`, `volume`, `cpc`, `cpc_low_bid`, `cpc_high_bid`,
   `zero_clicks_share`, `position`, `serp_features`. This is a `v4` path, so it supports
   `limit` / `offset` / `sort` / `asc` — **always** page it. Pulling an unbounded keyword
   set is the single largest avoidable credit spend in this workflow.
5. **Add positioning (optional).** `getGlobalRank` —
   `GET /v1/website/{domain_name}/global-rank/global-rank`, if `getLeadEnrichment` did not
   already answer it.
6. **Add mobile presence (optional).** `getAppDownloadsAndroid` —
   `GET /v5/apps/google/downloads` and `getAppDownloadsIos` — `GET /v5/apps/apple/downloads`.
   These take an app-store app id, not a domain, so only run them when you already have one.

## Rules while running

- Normalise the domain: bare, lowercase, no scheme, no path. A malformed domain returns
  HTTP 401 "Data not found", not a 404.
- Throttle to **10 requests per second** per key. Over that is HTTP 429; no rate-limit
  headers are returned, so count client-side. 429s cost no credits.
- 403 means invalid key **or** exhausted credits — check the balance before assuming the key
  is wrong.
- **Cache aggressively.** Firmographic and traffic estimates move on a monthly cadence, not a
  live one. Read `meta.last_updated` on each response and do not re-fetch a record whose
  underlying data has not refreshed. There is no idempotency key here — a repeat call is a
  repeat charge.
- When enriching more than a few hundred domains, stop and use the Batch API instead (see the
  `similarweb-bulk-report-to-cloud-storage` skill). That is what it exists for.

## PII note

Similarweb also publishes contact-search and contact-enrichment endpoints under its Company
Analysis API. Those return personal data about named individuals and are out of scope for
this skill deliberately. Do not fetch, store or forward personal contact records as part of
an automated enrichment loop without an explicit, lawful, human-approved basis.

## Version warning

Paths above span v1, v3, v4 and v5. Legacy REST endpoints sunset **2026-10-06** with no
runtime `Sunset` header. See `lifecycle/similarweb-lifecycle.yml`.
