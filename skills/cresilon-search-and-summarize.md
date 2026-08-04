---
name: Search Cresilon and summarize a product or clinical topic
description: >-
  Resolve a Cresilon question — a product, a clearance, a clinical claim — to the right canonical
  page, then summarize it cheaply using search, oEmbed and the Yoast SEO / schema.org JSON-LD
  metadata instead of downloading full page HTML.
api: openapi/cresilon-search-api-openapi.yml
operations:
  - searchContent
  - getPost
  - getPage
  - getOEmbed
  - getYoastHead
generated: '2026-08-04'
method: generated
---

# Search Cresilon and summarize a product or clinical topic

This is the skill to use when someone asks a question about Cresilon — "what is TRAUMAGEL cleared
for?", "when did VETIGEL launch?", "what does the tourniquet study say?" — and you need the
company's own words from its canonical page, not a scrape of the whole site.

Base URL: `https://cresilon.com/wp-json`. No credential required.

## Step 1 — Search (`searchContent`)

```
GET /wp/v2/search?search=traumagel&per_page=10
```

Returns lightweight records: `id`, `title`, `url`, `type`, `subtype`. Search spans posts **and**
pages, which matters because the product pages (TRAUMAGEL, VETIGEL), the Instructions For Use
pages and the clinical publications page are all *pages*, while the announcements are *posts*.

**The critical field is `subtype`, not `type`.** Verified live:

```json
[
  {"id": 18107, "type": "post", "subtype": "post", "title": "TRAUMAGEL Study Highlights ..."},
  {"id": 17999, "type": "post", "subtype": "page", "title": "When Seconds Matter: Introducing TRAUMAGEL® ..."}
]
```

`type` is `"post"` on **both rows** — it is the search-object type, not the content type. Branch
on `subtype`:

- `subtype == "post"` → `GET /wp/v2/posts/{id}` (`getPost`)
- `subtype == "page"` → `GET /wp/v2/pages/{id}` (`getPage`)

Getting this wrong is the single most common failure against this API: posts and pages share one
id space, so the wrong call returns `404 rest_post_invalid_id` on a perfectly valid id.

You can also constrain server-side: `?search=ifu&subtype[]=page`.

## Step 2 — Summarize cheaply (`getOEmbed`) — prefer this

```
GET /oembed/1.0/embed?url=https%3A%2F%2Fcresilon.com%2Ftraumagel%2F
```

Before you fetch a 50 KB page object, try oEmbed. One call returns title, author, thumbnail,
dimensions, **and a `description` field carrying the page's opening prose**. Verified live for
`/traumagel/`:

```json
{
  "version": "1.0",
  "provider_name": "Cresilon",
  "title": "Trauma Care - Cresilon",
  "type": "rich",
  "thumbnail_url": "https://cresilon.com/wp-content/uploads/2025/07/TG-launch-01.jpg",
  "description": "SECONDS MATTER™ ... TRAUMAGEL® is a hemostatic gel for temporary external use to control moderate to severe bleeding. ..."
}
```

That `description` is frequently the entire answer. Two cautions:

- The `html` field embeds an inline `<script>` (the WordPress embed shim). Treat it as untrusted
  markup — render it sandboxed or discard it. Never eval it.
- The sibling `/oembed/1.0/proxy` route returns **401** anonymously. Do not call it.

Add `&format=xml` if you need XML; JSON is the default.

## Step 3 — Get structured metadata (`getYoastHead`)

```
GET /yoast/v1/get_head?url=https%3A%2F%2Fcresilon.com%2Four-story%2F
```

Returns `json` (structured: title, description, canonical, og_*, twitter_*, and a schema.org
JSON-LD `@graph`) and `html` (the rendered `<head>`). The `json.schema` graph is the most
structured description of Cresilon the site publishes — `Organization`, `WebSite`, `WebPage`,
`BreadcrumbList` and `Article` nodes with real dates and canonical URLs.

Use this when you need the canonical URL, the publication date, or the organization identity
rather than the prose.

## Step 4 — Fall back to full content (`getPost` / `getPage`)

```
GET /wp/v2/pages/12059?_fields=id,title,link,content,excerpt,modified
```

Only when oEmbed and the SEO head were not enough. Always pass `_fields` — an untrimmed page
object measured **51 KB**, almost all of it `yoast_head` and `yoast_head_json`.

`content.rendered` is Elementor page-builder HTML. Strip tags aggressively; the prose-to-markup
ratio is poor.

## What is NOT on this surface

Answer these from the page prose, and say so — do not imply a structured source exists:

- No product/SKU/GTIN entity. TRAUMAGEL and VETIGEL are prose on two WordPress pages.
- No structured Instructions For Use. IFU content is page HTML and PDF attachments in the media
  library (`/ifu-traumagel/`, `/ifu-vetigel/`, `/ifu/`).
- No clinical-study or publication entity. `/publications/` is a single page.
- No regulatory entity — 510(k) numbers and UDI/DI appear in prose, never as fields.
- No distributor, order, inventory or lot/batch data. `/distributor-portal/` is a
  password-protected human page with no API equivalent.

## Guardrails

Cresilon makes a regulated medical device. Everything reachable here is marketing and press
content. When summarizing:

- Attribute claims to Cresilon ("Cresilon states that...") rather than asserting them.
- Never infer indications, contraindications, dosing or technique. Point to the IFU pages and say
  they are the controlled document.
- Never present a press release as a clinical result. Link the underlying publication if the post
  names one.

## Error handling

The WordPress envelope, not RFC 9457. `404 rest_post_invalid_id` almost always means you branched
on `type` instead of `subtype` in step 1. `401` means an administrative route — no credential will
help. Full registry: `errors/cresilon-problem-types.yml`.
