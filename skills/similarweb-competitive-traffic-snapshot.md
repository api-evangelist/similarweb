---
name: Competitive traffic snapshot for a domain
description: Pull a comparable traffic, engagement, geography and rank picture for one or more domains from the Similarweb REST API, after confirming the account is entitled to the countries and dates being asked for.
api: openapi/_original/similarweb-rest-api-openapi.yml
generated: '2026-08-13'
method: generated
operations:
  - checkCapabilities
  - getVisitsDesktop
  - getBounceRateDesktop
  - getTrafficSourcesOverview
  - getGeographyDesktop
  - getGlobalRank
  - getSimilarSites
---

# Competitive traffic snapshot

Use this when someone asks how a website is performing, or how two or more websites compare.

## Before you call anything

- Authenticate with the `api-key` request header. The key comes from the Similarweb
  platform (Settings > Account); only account admins can mint one. A query parameter
  `api_key` also works but prefer the header.
- **Normalise the subject domain first.** Send it bare and lowercase: `cnn.com`, never
  `CNN.COM`, never `https://cnn.com`, never `cnn.com/sports`. A malformed domain returns
  HTTP 401 "Data not found" — which is easy to misread as an auth failure, because an
  invalid key returns 401 as well.
- **Call `checkCapabilities` once before a batch of queries.** It reports the countries,
  datasets and date range the subscription actually covers. Requesting a country outside
  the plan returns application code `103 Country not available`, and requesting dates
  outside the plan returns `101 Dates not in range`. Both are avoidable.
- Every call spends **data credits**, priced on domains x endpoint x granularity x country
  filters x history x results. Ask for the narrowest date range and country set that
  answers the question.

## Steps

1. `checkCapabilities` — `GET /v1/website/{domain_name}/capabilities`. Record the available
   start and end dates and the country list; clamp everything below to them.
2. `getVisitsDesktop` — `GET /v1/website/{domain_name}/traffic-and-engagement/visits`. The
   headline volume series.
3. `getBounceRateDesktop` — `GET /v1/website/{domain_name}/traffic-and-engagement/bounce-rate`.
   Quality signal to sit next to volume; a visits number alone is not an answer.
4. `getTrafficSourcesOverview` — `GET /v1/website/{domain_name}/traffic-sources/overview-share`.
   The channel mix: direct, organic, paid, social, referral, mail, display.
5. `getGeographyDesktop` — `GET /v4/website/{domain_name}/geo/traffic-by-country`. Returns
   `records[]` of `GeographyRecord` (country, share, visits, pages_per_visit, average_time,
   bounce_rate, rank). This endpoint is paginated — use `limit`, `offset`, `sort`, `asc`
   rather than pulling every country.
6. `getGlobalRank` — `GET /v1/website/{domain_name}/global-rank/global-rank`. One number for
   positioning.
7. `getSimilarSites` — `GET /v2/website/{domain_name}/similar-sites`. Use this to *discover*
   the competitor set when the user has not named one, then loop steps 2–6 over the results.

For a multi-domain comparison, run steps 2–6 per domain with **identical** `start_date`,
`end_date`, `granularity` and country filters. Mixed parameters produce numbers that look
comparable and are not.

## Rules while running

- **Throttle to 10 requests per second.** That is the documented cap, per API key. Exceeding
  it returns HTTP 429. Similarweb emits no `X-RateLimit-*`, no `RateLimit-*` and no
  `Retry-After` header, so you cannot read your remaining budget off a response — you must
  count client-side. A 429 costs no data credits, so back off and retry rather than
  abandoning the run.
- **Do not retry blindly on 4xx.** 401 may be a bad domain *or* a bad key, 403 is an invalid
  key *or* an exhausted credit balance. Read the body: an invalid key returns the plain-text
  string `invalid API key`. Responses are not RFC 9457 problem+json.
- **There is no idempotency key.** Similarweb documents none. These are read operations, so a
  duplicate call is safe in effect — but it is not free, it spends credits again. Cache.
- Every response carries a `meta` object with `request`, `status` and `last_updated`. Report
  `last_updated` to the user; this is estimated data with a refresh cadence, not live
  telemetry.

## Version warning

The paths above are v1/v2/v4. Similarweb API V5 is live and the legacy REST endpoints have a
published sunset of **2026-10-06** (see `lifecycle/similarweb-lifecycle.yml`). No `Sunset` or
`Deprecation` response header is emitted, so nothing will warn you at runtime. For example,
`getTrafficSourcesOverview` maps to
`GET /v5/website-analysis/websites/traffic-channels/share?web_source=desktop` in V5. Check the
migration table before shipping anything long-lived.
