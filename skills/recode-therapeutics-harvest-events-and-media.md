---
name: recode-therapeutics-harvest-events-and-media
description: Harvest the ReCode Therapeutics conference and investor event listings and the 305-item media library (posters, decks, logos) from the company's WordPress content API.
api: recode-therapeutics:recode-therapeutics-content-api
generated: '2026-08-05'
method: generated
source: openapi/recode-therapeutics-content-openapi.yml
operations:
  - listEvents
  - getEvent
  - listMedia
  - getMediaItem
  - searchContent
---

# Harvest ReCode Therapeutics events and media

Two collections that a scraper of the rendered site would miss entirely.

Anonymous GET against `https://recodetx.com/wp-json`. No key.

## Events

`listEvents` → `GET /wp/v2/events?per_page=100&_fields=id,slug,title,excerpt,date_gmt,link,featured_media`

Fifty items at verification. `events` is a **site-specific custom post type** registered into
`wp/v2` — it is not WordPress core, and it exists only because this site's theme declares it. It
carries the conference appearances, investor presentations and scientific meetings ReCode
Therapeutics participates in: ATS, NACFC, Gordon Research Conferences, J.P. Morgan Healthcare,
Jefferies, Oppenheimer, Stifel, Morgan Stanley Global Healthcare.

Fetch the body with `getEvent` → `GET /wp/v2/events/{id}` when you need the detail — date, venue and
format live inside `content.rendered` as prose, not as fields. The type registers **no taxonomies**,
so there is nothing to filter on beyond `search` and date ordering.

Because it is a non-core type registered into the core namespace, it can vanish with a theme change
and nothing will announce it. Confirm it still exists with `GET /wp/v2/types` before each run rather
than treating a `404 rest_no_route` as a transient failure.

## Media library

`listMedia` → `GET /wp/v2/media?per_page=100&_fields=id,source_url,mime_type,media_type,alt_text,title,filename,filesize,post`

305 attachments, four pages. The `_fields` projection is not optional in practice — without it every
item drags along the full `media_details.sizes` rendition map and the walk goes from kilobytes to
megabytes.

Filter to what you actually want:

- `?mime_type=application/pdf` — the conference posters and corporate decks. These are the highest
  value items in the library: ATS posters on DNAI1 mRNA optimization and aerosol delivery,
  presented material not otherwise available as text.
- `?media_type=image` — logos, leadership headshots, figure art.

Per item: `source_url` is the direct file URL. `post` back-references the object the attachment is
attached to — but it is **not type-discriminated**, so the ID may belong to a post, a page or a
custom post type. Resolve it by reading the `_links.about` relation rather than guessing a
collection.

`getMediaItem` → `GET /wp/v2/media/{id}` for a single item, including the full
`media_details.sizes` map of generated renditions.

## Cross-referencing

`searchContent` → `GET /wp/v2/search?search={term}&subtype=events` finds events by term across the
whole site in one call, and returns the `subtype` discriminator you need to resolve each hit back to
its owning collection.

## Handling errors and etiquette

| code | status | what to do |
|---|---|---|
| `rest_invalid_param` | 400 | `per_page` above the hard cap of 100 |
| `rest_post_invalid_id` | 404 | ID belongs to a different collection |
| `rest_no_route` | 404 | the custom post type was deregistered — re-read `GET /wp/v2/types` |

No rate limit is published and there are no `RateLimit-*` headers. Traffic passes through
Cloudflare. Serialize requests, pause between pages, and back off hard on any non-JSON body — that
is an edge challenge, not a WordPress error, and there is no support channel to appeal a block.

Downloading 305 files is a different act from reading 305 records. Fetch the index; fetch the
binaries only for the items you actually need.

## Do not

- Do not call `/wp/v2/users`. Leadership headshots are in the media library with `alt_text`; the
  author collection is personal data and is out of scope for this skill.
