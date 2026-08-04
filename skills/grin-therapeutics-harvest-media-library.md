---
name: Harvest the GRIN Therapeutics media library
description: Page through the 174-item WordPress media library behind grintherapeutics.com to retrieve scientific figures, corporate imagery and logos with their source URLs, MIME types, alt text and size variants.
api: openapi/grin-therapeutics-content-openapi.yml
base_url: https://grintherapeutics.com/wp-json
operations:
  - listMedia
  - getMediaItem
  - getPage
generated: '2026-08-04'
method: generated
---

# Harvest the GRIN Therapeutics media library

The media library is the largest addressable collection on this surface — **174 attachments** at
harvest time, versus 10 pages and 0 posts. It holds the scientific figures used across the Our
Science and Disease State pages (for example `NMDA_Receptor_Scaled.png`), corporate imagery, and the
company logo lockups.

## Steps

### 1. Page through the collection

Call `listMedia` (`GET /wp/v2/media`) with `per_page=100`.

```
GET /wp/v2/media?per_page=100&orderby=date&order=desc
```

- `per_page` is capped at **100**; asking for more returns `400 rest_invalid_param` with
  `data.details.per_page.code = rest_out_of_bounds`.
- Read `X-WP-Total` and `X-WP-TotalPages` to size the job — do not assume 174 is still current.
- Follow the `Link: rel="next"` header rather than incrementing `page` blindly.

### 2. Narrow before you fetch

The route index exposes real filters — use them instead of downloading everything:

- `media_type=image` — restricts to images (also `video`, `audio`, `text`, `application`).
- `mime_type=image/png` — exact MIME match.
- `search=<term>` — matches filename and title; e.g. `search=NMDA`, `search=logo`.
- `after=` / `before=` on published date, `modified_after=` / `modified_before=` on modified date,
  all ISO 8601 — the right way to run an incremental harvest on a second pass.
- `_fields=id,slug,title,alt_text,mime_type,source_url,media_details` keeps the payload to what you
  actually need; the full `media_details` block is large.

### 3. Read the fields that matter

| Field | Use |
|---|---|
| `source_url` | The direct file URL under `/wp-content/uploads/` — this is what you fetch |
| `mime_type`, `media_type` | Format routing |
| `alt_text`, `caption.rendered`, `title.rendered` | Descriptive text. Often empty on this site — do not depend on it |
| `media_details.sizes` | Pre-rendered size variants (thumbnail, medium, large, and theme-specific crops) with their own `source_url`s. **Prefer a size variant over the full-size original** unless you specifically need full resolution |
| `post` | The page this attachment is attached to, or `null`. Most attachments on this site are unattached, so this is a weak signal |
| `date`, `modified` | Freshness. The most recent item observed was dated `2025-11-12T13:31:02` |

### 4. Fetch a single item

Call `getMediaItem` (`GET /wp/v2/media/{id}`) when you already have an id — from a page's
`featured_media`, from a `listMedia` page, or from `_embedded` after passing `_embed` on a page
request.

### 5. Attribute correctly

These are GRIN Therapeutics' copyrighted corporate and scientific assets, published for the
company's own site. The catalog records their existence and location; it does not license them.
Anything you retrieve should be attributed to GRIN Therapeutics and linked back to the page it
appears on. Do not re-host.

## Errors

| Status | code | What to do |
|---|---|---|
| 400 | `rest_invalid_param` | `per_page` outside 1–100, or a bad `media_type`/`mime_type` value |
| 404 | `rest_post_invalid_id` | The attachment id does not exist |

## Etiquette

JSON responses carry `Cache-Control: public, max-age=604800` and are Fastly-cached, but there is no
`ETag` or `Last-Modified`, so conditional GET is not available — you must track `modified` yourself
and use `modified_after` for incremental runs. No rate-limit policy is published; harvest once,
sequentially, and cache.
