---
name: recode-therapeutics-read-corporate-content
description: Read the ReCode Therapeutics corporate site — purpose, SORT LNP platform, drug pipeline, patient programs, leadership and company values — as structured content instead of scraping rendered HTML.
api: recode-therapeutics:recode-therapeutics-content-api
generated: '2026-08-05'
method: generated
source: openapi/recode-therapeutics-content-openapi.yml
operations:
  - listPages
  - getPage
  - listValues
  - getValue
  - searchContent
  - listTypes
---

# Read the ReCode Therapeutics corporate site as data

The whole recodetx.com site is retrievable as JSON. Eighteen pages, with `content.rendered`
populated — so you get the actual prose about the SORT lipid nanoparticle platform, the pipeline
and the patient programs, not an empty shell.

Anonymous GET against `https://recodetx.com/wp-json`. No key.

## Step 1 — get the site tree

`listPages` → `GET /wp/v2/pages?per_page=100&_fields=id,slug,title,parent,menu_order,link,modified_gmt`

Eighteen pages, one request. `parent` and `menu_order` give you the hierarchy: `parent: 0` is
top-level, `menu_order` orders siblings. That reconstructs the navigation without touching
`/wp/v2/menu-items`, which returns `401 rest_cannot_view` anonymously.

The pages worth knowing by slug:

| slug | what is on it |
|---|---|
| `about` | Our Purpose — company mission |
| `leadership` | Who We Are — the executive and board roster |
| `careers-culture` | Careers & Culture |
| `science` | the SORT LNP delivery platform |
| `pipeline` | the drug pipeline and development stages |
| `patients` | Who We Serve |
| `cf` | Cystic Fibrosis program |
| `pcd` | Primary Ciliary Dyskinesia program |
| `partnering` | business development |
| `news` | the news index |
| `terms-conditions`, `privacy-policy`, `cookie-policy` | legal |

## Step 2 — pull the pages you want

`getPage` → `GET /wp/v2/pages/{id}`

Or fetch by slug in one call, which avoids hard-coding numeric IDs that are not stable across a
site rebuild:

`GET /wp/v2/pages?slug=pipeline`

Returns a single-element array. `content.rendered` is the page HTML — the home page alone is 53 KB,
so request bodies deliberately, one page at a time.

**What you will not get:** the pipeline table, the trial phases and the leadership roster are
rendered as page-builder HTML inside `content.rendered`. There is no structured pipeline object, no
`Drug` or `MedicalTrial` schema.org node, no ACF payload (the `acf` key is present but empty
anonymously). You will be parsing HTML for the fine structure — the API gets you the document, not
the fields inside it. For authoritative trial structure, go to ClinicalTrials.gov instead.

## Step 3 — read the company values

`listValues` → `GET /wp/v2/values?per_page=100`

Six items. A site-specific custom post type registered into `wp/v2` — each is one stated company
value with a title and a body. These render on the About and Careers & Culture pages but are only
available as discrete records through this collection.

## Step 4 — search across everything

`searchContent` → `GET /wp/v2/search?search={term}&per_page=100`

Searches every REST-exposed post type at once. A search for `mrna` returned 79 hits spanning the
`post` and `events` subtypes.

Each hit is lightweight — `id`, `title`, `url`, `type`, `subtype`. **`subtype` is the
discriminator**: it tells you whether to resolve `id` against `/wp/v2/posts`, `/wp/v2/pages`,
`/wp/v2/events` or `/wp/v2/values`. A bare id is ambiguous; the same integer sequence is shared
across post types and `GET /wp/v2/pages/{postId}` returns `404 rest_post_invalid_id`.

Constrain with `&subtype=pages` when you only want corporate pages.

## Step 5 — discover what else exists

`listTypes` → `GET /wp/v2/types`

Twenty-one registered post types, self-describing. Use this rather than assuming: the site-specific
types (`events`, `values`, `portfolio`) are registered into `wp/v2` alongside core, so they can
appear or disappear with a theme or plugin change and there is no versioning or changelog that
would announce it. `portfolio` is registered but empty (0 items).

Types whose collections return `401` anonymously — `elementor_library`, `wp_template`,
`wp_navigation` internals — are builder infrastructure, not content. Ignore them.

## Notes

- `content.rendered` HTML contains inline styles and builder wrapper divs. Strip to text before
  feeding a model.
- Pages change rarely. The home page's schema.org graph reports `dateModified` of 2025-08-15.
  Re-read weekly at most; use `?modified_after=` to check cheaply.
- Do not call `/wp/v2/users` to attribute page authorship. Author records are personal data and are
  out of scope for this skill.
