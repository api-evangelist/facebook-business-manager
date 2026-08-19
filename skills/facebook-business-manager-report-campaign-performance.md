---
name: facebook-business-manager-report-campaign-performance
description: >-
  Pull Meta ads performance — account-level and campaign-level insights with breakdowns, paged safely to
  completion. Use when an agent needs spend, impressions, clicks, reach or conversion metrics out of an ad
  account.
generated: '2026-08-13'
method: generated
source: openapi/facebook-business-manager-insights-api-openapi.yml, openapi/facebook-business-manager-campaigns-api-openapi.yml
api: Facebook Business Manager
baseURL: https://graph.facebook.com/v26.0
operations:
  - getAdAccountInsights
  - getCampaignInsights
  - listCampaigns
  - listAdSets
  - listAds
scopes:
  - ads_read
  - ads_management
---

# Report campaign performance

Insights are not a node — they are a computed report keyed by object and time window. The same
`/insights` edge hangs off the ad account, the campaign, the ad set and the ad.

## Steps

### 1. Enumerate what you are reporting on — `listCampaigns`

`GET /act_{ad_account_id}/campaigns?fields=id,name,objective,status`

The Graph API returns a minimal default field set, so `fields` is mandatory, not optional.

### 2. Pull account-level insights — `getAdAccountInsights`

`GET /act_{ad_account_id}/insights`

Set `fields` to the metrics you need (impressions, clicks, spend, reach, cpc, ctr, actions),
a `level` (`account` | `campaign` | `adset` | `ad`), a `time_range` or `date_preset`, and any
`breakdowns`.

### 3. Pull campaign-level insights — `getCampaignInsights`

`GET /{campaign_id}/insights` — same parameter shape, scoped to one campaign.

### 4. Page to completion

Every list response is `{data: [...], paging: {...}}`.

- Prefer **cursor** pagination: follow `paging.next` until it is absent.
- **Stop on the absence of `paging.next` — never on a short page.** A page can return fewer rows
  than `limit`, or zero rows, and still have more pages behind it.
- **Do not store cursors.** They are invalidated as soon as the item they point at moves or is
  deleted.
- If you page too deep you get error code 100: "The After Cursor specified exceeds the max limit
  supported by this endpoint." That is a signal to narrow the query, not to retry.

See `conventions/facebook-business-manager-conventions.yml`.

## Rules that will bite you

**Insights is the most rate-limited surface on the platform.** Ads Insights on Standard Access
allows `600 + 400 * active ads - 0.001 * user errors` calls per rolling hour; Advanced Access
raises the floor to 190,000. Note the term subtracting your own errors — a failing loop lowers
your ceiling.

**You can be throttled on complexity, not just count.** Meta separately throttles on
`backend_qps` and `complexity_score`. Wide date ranges, many object IDs, many metrics and many
breakdowns in a single query all raise complexity. The documented remedy is to split one large
query into several smaller ones spaced out over time — the opposite of the usual advice.

**Async is the right tool for large pulls.** For anything wide, submit the report as an async job
(`POST /act_{ad_account_id}/insights`) and poll the returned `report_run_id`, rather than paging a
synchronous query.

**Watch `X-Business-Use-Case-Usage`** after every call and stop at 100 on any of `call_count`,
`total_cputime`, `total_time`. There is no `Retry-After`; back off on
`estimated_time_to_regain_access`.

**Attribution windows change the numbers.** Two pulls of the same date range taken days apart will
not match, because conversions attribute backwards. Record the pull timestamp alongside the data.
