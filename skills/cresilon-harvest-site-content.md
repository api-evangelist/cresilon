---
name: Harvest the Cresilon site content archive
description: >-
  Mirror the Cresilon corporate site — every page, every news post, the media library and the
  taxonomy — driven by the discovery routes rather than a hardcoded content model, so the harvest
  survives a WordPress or plugin change.
api: openapi/cresilon-discovery-api-openapi.yml
operations:
  - getRouteIndex
  - listTypes
  - listTaxonomies
  - listStatuses
  - listUsers
  - listPages
  - getPage
  - listPosts
  - listMedia
generated: '2026-08-04'
method: generated
---

# Harvest the Cresilon site content archive

Use this when you need a complete, repeatable mirror of everything cresilon.com publishes —
53 pages, 36 news posts, 1,307 media items, 6 taxonomy terms — rather than an answer to one
question. Base URL: `https://cresilon.com/wp-json`. No credential required.

## Why drive it from discovery

Cresilon runs no API program. This surface is whatever WordPress and its plugins happen to expose
this week — the route index advertises **45 namespaces and 1,021 routes**, most of them
plugin-internal. A core or plugin upgrade can change it with no notice, no changelog and no
deprecation window. Hardcoding `/wp/v2/posts` and `/wp/v2/pages` works today; asking the site what
it has works after the next upgrade too.

## Step 1 — Read the route index (`getRouteIndex`)

```
GET /wp-json/
```

Returns site identity and the full route table. The fields that matter:

- `name` (`"Cresilon"`), `description` (`"Stops bleeding in seconds"`), `url`, `gmt_offset` (`-4`
  observed — **post `date` values are local time, not UTC**; use `date_gmt` when comparing).
- `site_logo` — a media attachment id (`16942` observed).
- `namespaces` — 45 at capture, including `wp/v2`, `oembed/1.0`, `yoast/v1`, `jetpack/v4`,
  `wpdm` / `wpdmpp/v1` (WordPress Download Manager), `elementor/v1`, `astra/v1`,
  `redirection/v1`, `akismet/v1`, `monsterinsights/v1`, `code-snippets/v1`.
- `authentication` — advertises `application-passwords` with an authorization endpoint at
  `/wp-admin/authorize-application.php`. **Note it and move on.** That is an administrative site
  credential, not a developer one. Do not attempt to obtain or use it.
- `routes` — every registered route with its methods and args.

The response is large (~1.1 MB). Fetch it once per run, not per request. Narrow it with
`?namespace=wp/v2` if you only need the core surface.

**Most of those 1,021 routes are not yours.** Fourteen administrative read routes return **401**
anonymously — `settings`, `themes`, `plugins`, `menus`, `menu-items`, `block-types`, `templates`,
`template-parts`, `pattern-directory/patterns`, `font-families`, `posts/{id}/revisions`,
`oembed/1.0/proxy`, `jetpack/v4/site`, `wp-abilities/v1/abilities`. So do all write methods.
Filter the route table to `GET` and skip anything outside `wp/v2`, `oembed/1.0` and `yoast/v1`
before you start crawling; the full list is in
`authentication/cresilon-authentication.yml`.

## Step 2 — Enumerate the content model (`listTypes`, `listTaxonomies`, `listStatuses`)

```
GET /wp/v2/types
GET /wp/v2/taxonomies
GET /wp/v2/statuses
```

`listTypes` gives you each post type's `rest_base` — that is the collection path to crawl, and
reading it beats assuming `posts`/`pages`/`media`.

`listStatuses` returns **two** statuses anonymously, not one: `publish` and `acf-disabled` (an
Advanced Custom Fields internal status, queryable but holding no reader-facing content). Harvest
`publish` only.

## Step 3 — Record the author (`listUsers`)

```
GET /wp/v2/users?per_page=100&_fields=id,name,slug,link
```

Unusually for a WordPress site in this catalog, this returns **200**, not 401. There is exactly
one author (`cresilon`, id `211412379`) and every post resolves to it, so store it once and stop
resolving the `author` field per post.

## Step 4 — Crawl the collections (`listPages`, `listPosts`, `listMedia`)

```
GET /wp/v2/pages?per_page=100&page=1&_fields=id,slug,link,title,content,excerpt,parent,menu_order,date,modified
GET /wp/v2/posts?per_page=100&page=1&_fields=id,slug,link,title,content,excerpt,date,date_gmt,modified,categories,tags,featured_media
GET /wp/v2/media?per_page=100&page=1&_fields=id,slug,title,source_url,mime_type,media_type,filesize,post,alt_text,media_details
```

- `per_page` max is **100**; 999 returns HTTP 400 `rest_invalid_param`.
- Drive pagination from the RFC 8288 `Link` header's `rel="next"`, and sanity-check against
  `X-WP-Total` / `X-WP-TotalPages`.
- Media is the long pole — 1,307 items, 14 pages at `per_page=100`. Most are images; the IFU and
  clinical-publication PDFs are in here too, so filter with `mime_type=application/pdf` if that is
  what you are after.
- There is **no `Cache-Control` and no `ETag`** on any response, so you cannot revalidate. Use
  `modified` / `after` for incremental runs instead of conditional requests.

## Step 5 — Reconcile against the sitemaps

Cross-check the harvest against the three sitemaps declared in `/robots.txt`:
`/sitemap.xml` (Jetpack: page, image and video sitemaps), `/sitemap_index.xml` (Yoast) and
`/news-sitemap.xml`. If a URL is in a sitemap but not in your crawl, it is a content type you did
not enumerate in step 2 — go back rather than special-casing it.

The RSS feed at `/feed/` is a fourth view of the same 36 posts and a cheap way to detect new
publications without paging the API.

## Pacing and etiquette

No rate limits are signalled, `robots.txt` allows everything with no `Crawl-delay`, and there is
no caching contract. That is permission, not an invitation. This is a small marketing site for a
medical-device manufacturer: serialize your requests, insert a delay between pages, and run a full
harvest no more than daily. Responses carry `X-Robots-Tag: noindex` — the operator does not intend
this output to be republished as-is.

## Error handling

- `400 rate_invalid_param` — `per_page` over 100, or a bad `page`/`order` value.
- `404 rest_no_route` — the route pattern is not registered. Re-read the index (step 1) rather
  than retrying; the plugin surface may have changed under you.
- `401` on an administrative route — expected, not a failure. Skip it and continue. Never retry
  with a credential.
- `invalid_user_permission_view_admin` — the one observed code that does **not** use the `rest_*`
  prefix (Jetpack). If you match errors on `rest_`, you will miss it.

Full registry: `errors/cresilon-problem-types.yml`. Entity graph and traversal costs:
`data-model/cresilon-data-model.yml`.
