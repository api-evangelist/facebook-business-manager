---
name: facebook-business-manager-launch-ad-campaign
description: >-
  Launch a Meta ad end to end on the Marketing API — create the campaign, create the ad set with budget,
  schedule and targeting, upload the image, build the creative, then create the ad. Use when an agent needs
  to stand up new paid media on a Facebook/Instagram ad account.
generated: '2026-08-13'
method: generated
source: openapi/facebook-business-manager-campaigns-api-openapi.yml, openapi/facebook-business-manager-ad-sets-api-openapi.yml, openapi/facebook-business-manager-ad-images-api-openapi.yml, openapi/facebook-business-manager-ad-creatives-api-openapi.yml, openapi/facebook-business-manager-ads-api-openapi.yml
api: Facebook Business Manager
baseURL: https://graph.facebook.com/v26.0
operations:
  - createCampaign
  - createAdSet
  - uploadAdImage
  - createAdCreative
  - createAd
  - getCampaign
  - getAdSet
  - getAd
scopes:
  - ads_management
  - business_management
---

# Launch an ad campaign

Meta's advertising objects are a strict four-level hierarchy. Build it top down; every level below
needs the ID returned by the level above.

    AdAccount -> Campaign -> AdSet -> Ad -> AdCreative

## Before you start

- You need an ad account ID. The URL prefix is `act_` (`/act_123456789/campaigns`) but the
  `account_id` field itself is unprefixed. Getting this wrong is the most common 400 on this API.
- Node IDs are opaque numeric **strings**. Never parse them as integers — they exceed 2^53.
- Every write below requires the `ads_management` permission with Advanced Access, which requires
  App Review plus Business Verification. See `scopes/facebook-business-manager-scopes.yml`.

## Steps

### 1. Create the campaign — `createCampaign`

`POST /act_{ad_account_id}/campaigns`

Set the objective and `special_ad_categories` (send `[]` explicitly if none — it is required).
Create it with `status: PAUSED` so nothing spends while you build the rest.

Keep the returned `id` as `{campaign_id}`.

### 2. Create the ad set — `createAdSet`

`POST /act_{ad_account_id}/adsets`

Requires `campaign_id` from step 1, plus budget (`daily_budget` or `lifetime_budget` in **minor
units** — 1000 = $10.00), `billing_event`, `optimization_goal`, schedule, and the `targeting` spec.
To target a saved audience, reference it in `targeting.custom_audiences[]` — build it first with
the `facebook-business-manager-build-custom-audience` skill.

Keep the returned `id` as `{ad_set_id}`.

### 3. Upload the image — `uploadAdImage`

`POST /act_{ad_account_id}/adimages`

Returns an `image_hash`. This is what the creative references — not a URL.

### 4. Build the creative — `createAdCreative`

`POST /act_{ad_account_id}/adcreatives`

Assemble the `object_story_spec` with the Page ID, the `image_hash` from step 3, the link, message
and call to action.

Keep the returned `id` as `{creative_id}`.

### 5. Create the ad — `createAd`

`POST /act_{ad_account_id}/ads`

Requires `adset_id` from step 2 and the creative from step 4. Create with `status: PAUSED`.

### 6. Verify, then go live

Read back with `getCampaign` (`GET /{campaign_id}`), `getAdSet` (`GET /{ad_set_id}`) and `getAd`
(`GET /{ad_id}`), each with an explicit `?fields=` list — the Graph API returns a minimal default
field set. Only then flip `status` to `ACTIVE` with `updateCampaign` / `updateAdSet`.

## Rules that will bite you

**There is no idempotency key.** The Graph API publishes no `Idempotency-Key` header and no
safe-retry contract for POST. If a create times out, do NOT blindly retry — read back the parent's
edge (`listCampaigns`, `listAdSets`, `listAds`) and check whether the object already exists. See
`conventions/facebook-business-manager-conventions.yml`.

**HTTP 200 does not mean success.** The Graph API frequently returns 200 with an `error` object in
the body. Dispatch on the presence of `error`, not on the status code.

**Rate limits are formula-based, not fixed.** Ads Management on Standard Access allows
`300 + 40 * active ads` calls per rolling hour. Read `X-Business-Use-Case-Usage` after every call;
when `call_count`, `total_cputime` or `total_time` approaches 100, stop. On throttle, back off on
`estimated_time_to_regain_access` — there is no `Retry-After` header. Error codes 4, 17, 32, 613
all mean throttled. See `rate-limits/facebook-business-manager-rate-limits.yml`.

**Capture `fbtrace_id` on every error.** It is the only Meta support identifier, it appears only in
error bodies, and it expires quickly.

**Pin your version.** Call `https://graph.facebook.com/v26.0/...` explicitly. An unversioned call
uses whatever version is set in the App Dashboard, and an expired version silently downgrades to
the next oldest usable one rather than failing.
