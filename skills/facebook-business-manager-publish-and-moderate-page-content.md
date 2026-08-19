---
name: facebook-business-manager-publish-and-moderate-page-content
description: >-
  Publish to a Facebook Page and moderate the conversation underneath — create feed posts, upload photos and
  video, read and reply to comments, delete abuse, and read Page insights. Use when an agent runs organic
  social for a business Page.
generated: '2026-08-13'
method: generated
source: openapi/facebook-business-manager-posts-api-openapi.yml, openapi/facebook-business-manager-comments-api-openapi.yml, openapi/facebook-business-manager-photos-api-openapi.yml, openapi/facebook-business-manager-videos-api-openapi.yml, openapi/facebook-business-manager-pages-api-openapi.yml, openapi/facebook-business-manager-page-insights-api-openapi.yml
api: Facebook Business Manager
baseURL: https://graph.facebook.com/v26.0
operations:
  - getPage
  - getPageFeed
  - createPagePost
  - getPost
  - updatePost
  - deletePost
  - getPostComments
  - createComment
  - deleteComment
  - uploadPagePhoto
  - uploadPageVideo
  - getPageInsights
scopes:
  - pages_show_list
  - pages_read_engagement
  - pages_read_user_content
  - pages_manage_posts
  - pages_manage_engagement
  - publish_video
  - read_insights
---

# Publish and moderate Page content

## Get the right token first

This is the step that fails silently. Page operations need a **Page access token**, not the user
token you got from Login. Read `GET /me/accounts` with `pages_show_list` to list the Pages the user
manages and take the Page's own `access_token` from that response. A user token against a Page
edge returns permission errors that look like the object does not exist.

Confirm with `getPage` (`GET /{page_id}?fields=id,name,category,fan_count`).

Which rate-limit regime applies depends on the token: Pages calls made with a **user or app** token
fall under Platform Rate Limits; calls made with a **page or system user** token fall under
Business Use Case limits. Same endpoint, different ceiling.

## Publish

### Text and link posts — `createPagePost`

`POST /{page_id}/feed` with `message` and optionally `link`.

### Photos — `uploadPagePhoto`

`POST /{page_id}/photos`. To attach a photo to a post rather than publish it standalone, upload
with `published: false` and reference the returned ID in the post's `attached_media`.

### Video — `uploadPageVideo`

`POST /{page_id}/videos`. Requires `publish_video`. Large files use Meta's resumable upload
protocol rather than a single POST.

Post IDs come back composite: `{page_id}_{post_id}`. Store the whole string.

## Read and edit

- `getPageFeed` — `GET /{page_id}/feed?fields=id,message,created_time,permalink_url`
- `getPost` — `GET /{post_id}?fields=...`
- `updatePost` — `POST /{post_id}` (Meta uses POST, not PUT or PATCH, for updates)
- `deletePost` — `DELETE /{post_id}`

## Moderate

- `getPostComments` — `GET /{post_id}/comments?fields=id,message,from,created_time&order=reverse_chronological`
- `createComment` — `POST /{post_id}/comments` to reply publicly. To reply to a specific comment,
  POST to `/{comment_id}/comments` — comments nest.
- `deleteComment` — `DELETE /{comment_id}`

## Measure

`getPageInsights` — `GET /{page_id}/insights?metric=...&period=...`. Metrics are named and
period-scoped; request only the metrics you need.

## Rules that will bite you

**Subscribe to webhooks instead of polling for comments.** `subscribePageApp`
(`POST /{page_id}/subscribed_apps`) plus a `page` object subscription pushes feed and comment
changes to you, signed with `X-Hub-Signature-256`. Polling the comments edge on a busy Page is the
fastest way to hit error code 32. See `asyncapi/facebook-business-manager-webhooks.yml`.

**A webhook subscription with the wrong permission delivers nothing, silently.** Webhooks does not
require App Review, but it does respect permissions — no error, no events.

**Deletes are not idempotent-safe to retry.** There is no idempotency key on this API. A retried
`deleteComment` on an already-deleted comment returns an error, not a no-op.

**HTTP 200 can carry an error.** Check for an `error` object in every response body regardless of
status. Capture `fbtrace_id` when you find one.

**Duplicate posts are rejected.** Error code 506 — consecutive identical posts are blocked. Vary
the content or check the feed before re-posting.

**The 90-day permission decay applies.** A permission unused for 90 days must be re-granted by the
app user. Low-frequency Page automation breaks on this without warning.
