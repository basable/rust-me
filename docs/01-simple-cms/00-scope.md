# 00 \u2014 Scope: simple-cms

## The idea, in the user's words

> i want a cms system for simple websites

## Fit gate

**Fits.** A CMS is a backend with its state in Postgres (sites, members, pages, revisions, publication state) driven through a CRUD + workflow API, with an admin UI on top and server-rendered public pages. There is no binary media store (media library excluded), so Postgres is the only source of truth.

## Working interpretation

A user signs in (Kratos, passwordless email code / passkey), creates one or more websites, invites existing accounts as owners or editors, writes pages as JSON blocks (heading, paragraph, image-by-URL, link), saves drafts (every save is a revision), publishes now or schedules publish/unpublish at a future time, and arranges a navigation menu. Visitors get real HTML pages rendered by the app with the site's theme. "Simple websites" = brochure sites, small business sites, portfolios: tens to a few hundred pages per site, a handful of editors.

## Scope ledger

Decisions: `include` / `exclude` / `later` / `open` (not yet asked).

### Sites & access

| Feature | Decision | User note | Rationale |
|---|---|---|---|
| Sites (create, rename, choose theme) | include (baseline) | \u2014 | Without a site there is nothing to manage. |
| Editors via Kratos login | include (baseline) | \u2014 | Standard auth; an editor is a Kratos identity. |
| Multiple sites per account, per-site members & roles | **include** (round 1) | \u2014 | `site_member` with `owner` / `editor`; every content mutation is authorized by `site`. |
| Add a member by email of an existing account | include (follows from multi-site) | \u2014 | A Kratos admin lookup (a read, not an effect). Email invitations to people without an account need SMTP \u2192 not now. |
| Custom domains per site | open (round 2) | \u2014 | Needs per-domain TLS/routing on the platform. |
| Themes (layout + CSS as a config catalog) | include (baseline) | \u2014 | A seeded `theme` catalog, selected per site. |

### Content

| Feature | Decision | User note | Rationale |
|---|---|---|---|
| Pages with block content (JSON) | include (baseline) | \u2014 | The core object. |
| Navigation menu per site | include (baseline) | \u2014 | Every simple website has one. |
| Draft / publish with revision history | **include** (round 1) | \u2014 | `page_revision` (append-only) + `published_page` pointer; rollback = publish an older revision. |
| Scheduled publish / unpublish | **include** (round 1) | \u2014 | The one real lifecycle (waiting until a time, then driving the live pointer): processing-object type `publication` in `content`. |
| Media library (uploads) | **exclude** (round 1) | \u2014 | Images are linked by external URL; no object storage. |
| Draft preview for editors | open (round 2) | \u2014 | Render an unpublished revision behind auth. |
| Blog posts (dated, listed, tags) | open (round 2) | \u2014 | A page kind + an index listing, or its own table. |
| Contact forms / submissions | open (round 2) | \u2014 | A public write path; email notification would need SMTP. |
| SEO metadata, sitemap.xml | open (round 2) | \u2014 | Cheap columns + one generated route. |

### Delivery

| Feature | Decision | User note | Rationale |
|---|---|---|---|
| Public delivery: server-rendered HTML with the theme | **include** (round 1) | \u2014 | New `delivery` nanoservice: a stateless renderer over `site` and `content`. |

### Not considered for this product

E-commerce, multi-language content, comments, analytics \u2014 out of a "simple websites" CMS unless the user raises them.

## Open candidates not yet asked

Custom domains; draft preview; blog posts; contact forms; SEO metadata & sitemap.

## Ready

Not ready \u2014 round 2 questions pending.
