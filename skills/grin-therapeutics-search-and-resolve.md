---
name: Search GRIN Therapeutics content and resolve results to full records
description: Use the cross-content search endpoint to find GRIN Therapeutics material by term (radiprodil, GRIN-NDD, Beeline, Astroscape, tuberous sclerosis) and resolve each lightweight hit into the full page record, including its author and attached media.
api: openapi/grin-therapeutics-content-openapi.yml
base_url: https://grintherapeutics.com/wp-json
operations:
  - searchContent
  - getPage
  - getUser
  - listTaxonomies
generated: '2026-08-04'
method: generated
---

# Search GRIN Therapeutics content and resolve results

`searchContent` is the fastest way into this surface when you have a term rather than an id. It
returns lightweight projections that you then resolve.

## Steps

### 1. Search

Call `searchContent` (`GET /wp/v2/search`) with `search=<term>`. Useful terms for this provider:
`radiprodil`, `GRIN-NDD`, `GluN2B`, `NMDA`, `Beeline`, `Astroscape`, `Honeycomb`, `tuberous
sclerosis`, `focal cortical dysplasia`, `Neurvati`, `Angelini`.

```
GET /wp/v2/search?search=radiprodil&per_page=20
```

Each result is a projection, **not** a full record:

```json
{
  "id": 11960,
  "title": "Resources for patients and caregivers",
  "url": "https://grintherapeutics.com/resources-for-patients-and-caregivers/",
  "type": "post",
  "subtype": "page",
  "_links": { "self": [ { "embeddable": true, "href": "..." } ] }
}
```

Read `X-WP-Total` and `X-WP-TotalPages` for the result count; page with `page`/`per_page`.

### 2. Route on `subtype`, not `type`

`type` is the search-index class and will read `post` even for pages. **`subtype` is the field that
tells you which collection to resolve against.** On this site `subtype` will be `page` for
essentially every hit, because the `posts` collection is empty (`listPosts` returns `[]`).

- `subtype: page` → resolve with `getPage`
- `subtype: post` → resolve with `getPost` (you will not see this on this deployment)

### 3. Resolve

Call `getPage` (`GET /wp/v2/pages/{id}`) with the id from the search result. Before using
`content.rendered`, run the WPBakery cleanup described in
`skills/grin-therapeutics-read-corporate-content.md` — the search snippet gives you no body, and the
body you get back is builder markup.

Alternatively follow `_links.self[0].href` directly, which is the same resource and keeps you from
hard-coding routes.

### 4. Resolve the surrounding graph (optional)

- `author` (integer) → `getUser` (`GET /wp/v2/users/{id}`). Be aware these are agency accounts
  (`cglife`, `bencrossroad-us`), not named GRIN Therapeutics staff — do not present them as authors
  of the science.
- `featured_media` (integer) → `getMediaItem`.
- Pass `_embed` on the resolve call to inline author, featured media and terms under `_embedded` in
  one request instead of three.

### 5. Do not lean on taxonomy

`listTaxonomies` reports only `category` and `post_tag`, and every term in both returns `count: 0`.
The terms that exist (`Business`, `Creative`) are theme defaults, not GRIN Therapeutics' editorial
taxonomy. **Filtering by category or tag will return nothing.** Search and the page list are the only
real navigation.

## Errors

| Status | code | What to do |
|---|---|---|
| 400 | `rest_invalid_param` | `per_page` must be 1–100; check `data.params` |
| 404 | `rest_post_invalid_id` | The id from search no longer resolves — re-run the search |

## Etiquette

Search is unauthenticated and uncounted, but no rate-limit policy is published and no rate-limit
headers are returned. Batch your terms, cache results for the declared `max-age=604800`, and do not
poll.
