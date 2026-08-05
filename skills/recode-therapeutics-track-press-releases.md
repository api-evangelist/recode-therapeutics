---
name: recode-therapeutics-track-press-releases
description: Track ReCode Therapeutics press releases, publications and presentations incrementally from the company's WordPress content API, without re-reading the whole archive each run.
api: recode-therapeutics:recode-therapeutics-content-api
generated: '2026-08-05'
method: generated
source: openapi/recode-therapeutics-content-openapi.yml
operations:
  - listPosts
  - getPost
  - listCategories
  - listTags
  - getMediaItem
---

# Track ReCode Therapeutics press releases

ReCode Therapeutics is a clinical-stage genetic medicines company. Its news stream — 82 items at
verification — carries the material a watcher actually wants: clinical trial milestones for RCT1100
(primary ciliary dyskinesia) and RCT2100 (cystic fibrosis), FDA designations, financing rounds,
partnership announcements and peer-reviewed publications.

There is **no API key and no signup**. Every call below is an anonymous GET against
`https://recodetx.com/wp-json`.

## Before you start

- There is no status page, no SLA, no changelog and no support channel. This is the company's
  WordPress install, not a product. Be polite and fault-tolerant.
- The edge is Cloudflare and no rate limit is published. Keep concurrency at 1 and pause between
  pages. A block cannot be appealed — there is nobody to ask.
- Counts and pagination live in **response headers**, not the body. The body is a bare array.

## Step 1 — establish the category map

`listCategories` → `GET /wp/v2/categories`

Seven terms. The ones that matter:

| slug | count | what it is |
|---|---|---|
| `press-releases` | 47 | corporate and clinical announcements |
| `publications` | 9 | peer-reviewed papers (often outbound links to Nature/PNAS/Science) |
| `presentations` | 9 | conference and investor decks |
| `featured` | 14 | overlaps the others — a display flag, not a distinct class |
| `careers` | 3 | hiring notices |

Keep the term `id` for each slug. Filter with the **id**, not the slug — `?categories=` takes IDs.

## Step 2 — first full read

`listPosts` → `GET /wp/v2/posts?orderby=modified&order=asc&per_page=100&_fields=id,slug,title,excerpt,date_gmt,modified_gmt,link,categories,tags,featured_media`

- `per_page` is hard-capped at **100**. Asking for 101 returns `400 rest_invalid_param`.
- Read `X-WP-Total` (82) and `X-WP-TotalPages` from the headers.
- Follow the `Link` header's `rel="next"` until it is absent. Do not compute page numbers yourself.
- The `_fields` projection matters: without it each item carries a multi-kilobyte
  `content.rendered`. Fetch bodies only for the items you decide to keep.

Persist the highest `modified_gmt` you saw. That is your cursor.

## Step 3 — every run after the first

`GET /wp/v2/posts?modified_after={cursor}&orderby=modified&order=asc&per_page=100&_fields=...`

Use **`modified_gmt`**, the UTC field — not `modified`, which is site-local (UTC-7). Mixing them
across a daylight-saving boundary silently skips or replays records.

`modified_after` catches edits to existing releases as well as new ones, which is what you want:
ReCode has edited releases after publication (the most recent one was modified three minutes after
its publish timestamp).

## Step 4 — fetch the body of anything new

`getPost` → `GET /wp/v2/posts/{id}`

`content.rendered` is populated on this site (5.7 KB on the most recent release) — you get the real
prose, not an empty page-builder shell. `excerpt.rendered` is a short summary suitable for a digest.

Some `publications` entries point **off-site** — the linked resource is a Nature, PNAS, Science or
Journal of Cystic Fibrosis article, not a page on recodetx.com. Check the body for an outbound link
before assuming the post is the artifact.

## Step 5 — resolve the featured image, if you need it

`getMediaItem` → `GET /wp/v2/media/{featured_media}` — `source_url` is the direct file URL.
`featured_media: 0` means there is none.

Cheaper alternative: add `_embed` to the Step 3 request and read `_embedded['wp:featuredmedia']`
inline instead of making a second call per post.

## Handling errors

| code | status | what to do |
|---|---|---|
| `rest_invalid_param` | 400 | read `data.params` — almost always `per_page` above 100 |
| `rest_post_invalid_id` | 404 | the ID is not a post; do not retry |
| `rest_no_route` | 404 | the route was removed by a WordPress or plugin change — re-read `GET /wp-json/` |
| non-JSON body | any | a Cloudflare challenge, not a WordPress error. Back off hard. |

Branch on `code`. Never match on `message` — the same English sentence is returned by unrelated
namespaces.

## Do not

- Do not call `/wp/v2/users` to attribute a release. Author records are personal data and are
  deliberately out of scope for this skill. Every press release is corporate; there is no useful
  byline to attach.
- Do not poll faster than the content changes. The archive spans 2022–2026 at roughly two items a
  month. Daily is generous; hourly is abuse.
