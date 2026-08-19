---
name: facebook-business-manager-build-custom-audience
description: >-
  Create and reuse a Meta Custom Audience on an ad account, then target it from an ad set. Use when an agent
  needs to stand up retargeting or lookalike-source audiences before launching paid media.
generated: '2026-08-13'
method: generated
source: openapi/facebook-business-manager-custom-audiences-api-openapi.yml, openapi/facebook-business-manager-ad-sets-api-openapi.yml
api: Facebook Business Manager
baseURL: https://graph.facebook.com/v26.0
operations:
  - listCustomAudiences
  - createCustomAudience
  - createAdSet
  - updateAdSet
  - listAdSets
scopes:
  - ads_management
  - business_management
---

# Build and target a Custom Audience

## Steps

### 1. Check what already exists — `listCustomAudiences`

`GET /act_{ad_account_id}/customaudiences?fields=id,name,subtype,approximate_count_lower_bound,operation_status`

Do this first, every time. There is no idempotency key on this API, so a retried create makes a
second audience rather than returning the first. Matching by `name` before creating is the only
duplicate protection available.

### 2. Create the audience — `createCustomAudience`

`POST /act_{ad_account_id}/customaudiences`

Set `name`, `description`, `subtype` (`CUSTOM`, `WEBSITE`, `ENGAGEMENT`, `LOOKALIKE`, …) and, for
rule-based audiences, the `rule` describing the pixel/dataset event and retention window.

Keep the returned `id`.

### 3. Wait for it to populate

A new audience is not immediately targetable. Poll the audience with an explicit
`?fields=operation_status,approximate_count_lower_bound` and wait for the status to settle before
attaching it to an ad set. Audiences below Meta's minimum size will not deliver.

### 4. Target it — `createAdSet` / `updateAdSet`

Reference the audience ID in the ad set's `targeting.custom_audiences[]` array. Use
`targeting.excluded_custom_audiences[]` to suppress existing customers.

`createAdSet` — `POST /act_{ad_account_id}/adsets`
`updateAdSet` — `POST /{ad_set_id}`

Verify with `listAdSets` (`GET /act_{ad_account_id}/adsets?fields=id,name,targeting`).

## Rules that will bite you

**Never send raw PII.** Customer-list audiences require SHA-256 hashed, normalized identifiers —
lowercase, trimmed, and formatted to Meta's normalization rules before hashing. Sending plaintext
email or phone is both a policy violation and a data-protection incident.

**Custom Audience has its own rate ceiling.** Standard Access allows
`5000 + 40 * active custom audiences` calls per rolling hour, capped at 700,000; Advanced Access
starts at 190,000. This is a separate Business Use Case bucket from Ads Management, so exhausting
it does not show up in your campaign call budget.

**Audiences are ad-account-scoped.** An audience created on one ad account is not visible to
another unless it is explicitly shared through Business Manager. Cross-account targeting requires
a sharing step, not a copy.

**Advanced Access is required.** Audience creation needs `ads_management` at Advanced Access,
which means App Review plus Business Verification, plus an annual Data Use Checkup and — because
this touches user data — a Data Protection Assessment. See
`lifecycle/facebook-business-manager-lifecycle.yml`.

**Check for an `error` object even on HTTP 200,** and capture `fbtrace_id` when present.
