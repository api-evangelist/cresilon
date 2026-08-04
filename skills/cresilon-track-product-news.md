---
name: Track Cresilon product and clearance news
description: >-
  Monitor Cresilon's corporate news stream for TRAUMAGEL and VETIGEL announcements — FDA
  clearances, launches, clinical study publications and conference presentations — from the public
  WordPress content API, resolving categories, tags and featured images correctly.
api: openapi/cresilon-posts-api-openapi.yml
operations:
  - listCategories
  - listTags
  - listPosts
  - getPost
  - getMediaItem
generated: '2026-08-04'
method: generated
---

# Track Cresilon product and clearance news

Cresilon is a Brooklyn biotechnology company that makes plant-based hemostatic gels — VETIGEL for
veterinary use and TRAUMAGEL for human trauma care. Its news stream is where FDA clearances,
product launches, distribution agreements and peer-reviewed study publications get announced. All
of it is readable through the public WordPress REST API with **no credential of any kind**.

Base URL: `https://cresilon.com/wp-json`

## Before you start

- **No auth.** Send no `Authorization` header. There is no key to obtain and no signup. The root
  index advertises WordPress Application Passwords — that is an administrative site credential,
  not a developer one. Do not attempt to obtain it.
- **Be polite.** No rate limits are signalled and `robots.txt` sets no crawl delay, but there is
  also no `Cache-Control` or `ETag`, so you cannot revalidate cheaply. This is a marketing site
  for a medical-device company. Poll daily at most.
- **Trim the payload.** Every post carries a multi-kilobyte `yoast_head` string and a full
  `yoast_head_json` graph you almost never want. Always pass `_fields`.
- **This is press content, not clinical guidance.** Nothing you retrieve here is indications,
  dosing or instructions for use. Do not present it as medical advice.

## Step 1 — Learn the taxonomy (`listCategories`, `listTags`)

```
GET /wp/v2/categories?per_page=100&_fields=id,name,slug,count
GET /wp/v2/tags?per_page=100&_fields=id,name,slug,count
```

The taxonomy is tiny and has two traps worth knowing before you filter on it:

- **Category id 1 is named "News" but its slug is `uncategorized`** — it is the WordPress default
  term, renamed. Match on `id` or `name`; never on `slug`.
- **Category 1365 (`Press Releases`) holds 0 posts.** It exists but is unused. Filtering by it
  returns nothing.
- **Tags are effectively unused.** Four tags (`biotech`, `cresilon`, `fda clearance`, `vetigel`),
  two posts each, out of 36 total. Do not build a topic filter on them — 34 posts carry no tag.

The practical consequence: **there is no working server-side topic filter on this surface.** Get
everything and filter client-side on the title and excerpt.

## Step 2 — List posts (`listPosts`)

```
GET /wp/v2/posts?per_page=100&page=1&_fields=id,slug,link,title,excerpt,date,modified,categories,tags,featured_media
```

- 36 posts total at time of writing. `per_page` is capped at 100; sending 999 returns HTTP 400
  `rest_invalid_param`.
- Read `X-WP-Total` and `X-WP-TotalPages` from the response headers, and follow the RFC 8288
  `Link` header's `rel="next"` rather than incrementing `page` blindly.
- For incremental polling use `after` with an ISO 8601 timestamp:
  `?after=2026-01-01T00:00:00&orderby=date&order=desc`.
- `excerpt.rendered` is HTML with entity escapes (`&#8220;`, `&hellip;`). Decode before matching.

Post titles are the signal here. Cresilon's announcement vocabulary is consistent: "Receives FDA
Clearance", "Announces U.S. Nationwide Launch", "Announces Poster Presentation", "Study
Highlights", "Announces International Distribution Agreement".

## Step 3 — Read a post (`getPost`)

```
GET /wp/v2/posts/18107?_fields=id,title,date,link,content,excerpt,featured_media
```

`content.rendered` is full page-builder HTML (this site runs Elementor), so expect heavy wrapper
markup around the prose. Strip tags before summarising.

**Watch the id space.** Posts and pages share one numbering sequence on this site. Id 17999 looks
like a news item — *"When Seconds Matter: Introducing TRAUMAGEL® for Rapid Bleeding Control in
PA"* — but it is a **page**, and `GET /wp/v2/posts/17999` returns 404 `rest_post_invalid_id`. If
you got an id from search, branch on its `subtype`; see the search-and-summarise skill.

## Step 4 — Resolve the featured image (`getMediaItem`)

```
GET /wp/v2/media/18109?_fields=id,source_url,alt_text,mime_type,media_details
```

Only if you actually need it. Cheaper alternative: pass `_embed=true` on step 2 or 3 and read
`_embedded['wp:featuredmedia'][0].source_url`. Cheaper still — the Jetpack plugin flattens the
resolved URL straight onto the post as `jetpack_featured_media_url`, so add that to `_fields` and
skip this step entirely.

`alt_text` is empty across much of this library. Do not rely on it for image description.

## Error handling

Errors are the WordPress envelope, **not** RFC 9457 problem+json:

```json
{"code": "rest_post_invalid_id", "message": "Invalid post ID.", "data": {"status": 404}}
```

- `400 rest_invalid_param` — read `data.params` and `data.details` for the offending parameter.
- `404 rest_post_invalid_id` — wrong id, or the id belongs to a page.
- `401 rest_forbidden` — you hit an administrative route. **No credential will fix this.** Back
  off; do not retry with auth.

Full registry: `errors/cresilon-problem-types.yml`.
