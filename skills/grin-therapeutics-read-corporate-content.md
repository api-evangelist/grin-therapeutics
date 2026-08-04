---
name: Read GRIN Therapeutics corporate and science content
description: Retrieve GRIN Therapeutics' published pages — About, Our Science, Disease State, Resources for patients and caregivers, News, Careers, Contact, Privacy and Cookie Policy — over the anonymous WordPress REST content API, and clean the WPBakery markup so the text is usable.
api: openapi/grin-therapeutics-content-openapi.yml
base_url: https://grintherapeutics.com/wp-json
operations:
  - listPages
  - getPage
  - listTypes
generated: '2026-08-04'
method: generated
---

# Read GRIN Therapeutics corporate and science content

GRIN Therapeutics is a clinical-stage biotechnology company developing radiprodil for GRIN-related
neurodevelopmental disorder, tuberous sclerosis complex and focal cortical dysplasia type II. Its
substantive public content lives in **ten WordPress pages**, all readable with no credentials.

## Before you start

- **No authentication.** Every operation here returns data with no Authorization header, no cookie
  and no nonce. Do not attempt to obtain a credential — GRIN Therapeutics issues none to third
  parties, and the credentialed routes (`/wp/v2/settings`, `wp-abilities/v1`, `contact-form-7/v1`)
  will return `401 rest_forbidden` or `403`.
- **This is a CMS content API, not a product API.** There is no versioning policy, no deprecation
  policy, no SLA, no status page and no rate-limit policy. Treat it as best-effort.
- **Read-only.** The `Allow` response header on the collections reads exactly `GET`. There is no
  idempotency contract because there is nothing to write.

## Steps

### 1. Confirm the surface is still shaped as expected

Call `listTypes` (`GET /wp/v2/types`) and check that `page.rest_base` is still `pages`. This is a
one-request sanity check: the site runs on WordPress core's release train (it reported WordPress
7.0.2 on 2026-08-04) and GRIN Therapeutics announces no contract changes, so verify rather than
assume.

### 2. List the pages

Call `listPages` (`GET /wp/v2/pages`) with `per_page=100`. There were 10 published pages at harvest
time, so one request is enough — but read `X-WP-Total` and `X-WP-TotalPages` rather than assuming,
and follow the `Link: rel="next"` header if it appears.

Use `_fields` to keep the payload small when you only need the index:

```
GET /wp/v2/pages?per_page=100&_fields=id,slug,title,link,modified
```

The pages you should expect, by slug:

| slug | what it holds |
|---|---|
| `home` | Company positioning |
| `about-grin-therapeutics` | Mission, founding story, leadership team, board of directors |
| `our-science` | Radiprodil mechanism of action, trial portfolio, regulatory designations |
| `disease-state` | GRIN-NDD, TSC and FCD type II background |
| `resources-for-patients-and-caregivers` | Patient and caregiver resources |
| `news` | Press releases and corporate events — see the caveat below |
| `careers` | Open roles |
| `contact` | Contact routes (titled "Working with Grin Therapeutics") |
| `privacy` | Privacy Policy |
| `cookie-policy` | Cookie Policy |

### 3. Fetch a page body

Call `getPage` (`GET /wp/v2/pages/{id}`) with the id from step 2. The body arrives in
`content.rendered`.

### 4. Clean the markup before using the text

**This is the step that matters.** `content.rendered` is **WPBakery page-builder shortcode markup**,
not semantic HTML. A raw extraction will be dominated by layout noise. Strip, in this order:

1. Shortcodes — `[vc_row ...]`, `[vc_column ...]`, `[vc_column_inner ...]`, `[vc_row_inner ...]` and
   their closing forms.
2. Inline builder CSS — `css=".vc_custom_1700241508945{padding-top: 80px !important; ...}"` blocks,
   including the `css_tablet_portrait` and `css_mobile` variants.
3. HTML tags.
4. HTML entities, then normalise typographic quotes (`”` `“` `’`) to ASCII — the shortcode attributes
   use curly quotes, which break naive attribute parsers.

Only after all four passes does the prose survive.

### 5. Handle the News caveat

The News page (`id: 6`) is where GRIN Therapeutics' press releases and corporate-event summaries
live — **as hand-authored markup inside that one page**, not as posts. Consequences you must plan
for:

- `listPosts` returns `[]` (`X-WP-Total: 0`). Do not treat this as an error or a fetch failure.
- `/feed/` is a valid RSS 2.0 channel with **no items**.
- There is no per-press-release resource, no stable id per release, no published date field, and no
  way to diff releases between polls except by diffing the page body.
- Dates appear only inside the prose (e.g. "September 2, 2025 – GRIN Therapeutics, Inc., ...").

If you need release-level structure, parse it out of the page body and key it on the release title —
and record that the date and title came from prose, not from a field.

## Errors you will actually see

| Status | code | Meaning | What to do |
|---|---|---|---|
| 400 | `rest_invalid_param` | A query parameter failed validation, e.g. `per_page` outside 1–100 | Read `data.params` and `data.details[param].code`; fix and retry |
| 404 | `rest_post_invalid_id` | The id does not exist or is not published | Resolve ids from `listPages` or `searchContent`, never construct them |
| 401 | `rest_forbidden` | The route needs a credential | Stop — this route is not available to you |

Full catalog: `errors/grin-therapeutics-problem-types.yml`. Note these are **not** RFC 9457
`application/problem+json`; the envelope is `{code, message, data:{status, params?, details?}}`.

## Etiquette

Responses carry `Cache-Control: public, max-age=604800` and are served through Fastly. There is no
`ETag` or `Last-Modified`, so conditional GET is unavailable — cache the response yourself for the
declared week rather than re-polling. No rate-limit headers are published; keep request volume low.
