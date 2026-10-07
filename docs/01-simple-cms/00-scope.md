# 00 — Scope: simple-cms

## The idea, in the user's words

> i want a cms system for simple websites

## Fit gate

**Fits.** A CMS is a backend with its state in Postgres (sites, pages, revisions, publication state, editors) driven through a CRUD + workflow API, with an admin UI on top. Binary media (images, files) would live in object storage, which is a later, agent-added integration, not the source of truth for content.

## Working interpretation (round 1)

An editor signs in (Kratos, passwordless email code / passkey), creates a website, writes pages in a block-based editor (headings, paragraphs, images, links stored as JSON blocks), arranges a navigation menu, and publishes. Visitors see the published version only. "Simple websites" = brochure sites, small business sites, portfolios, small blogs — tens to a few hundred pages per site, a handful of editors.

## Scope ledger

Decisions: `include` / `exclude` / `later` / `open` (not yet asked).

### Sites & access

| Feature | Decision | User note | Rationale |
|---|---|---|---|
| Sites (create, rename, settings) | include (baseline) | — | Without a site there is nothing to manage. |
| Editors via Kratos login | include (baseline) | — | Standard auth; an editor is a Kratos identity. |
| Multiple sites per account, per-site members & roles | open (round 1) | — | Decides whether `site_member` and role checks exist. |
| Custom domains per site | open | — | Needs TLS/routing per domain on the platform; likely `later`. |
| Themes (layout + CSS as a config catalog) | include (baseline, thin) | — | A seeded `theme` catalog is the cheapest way to style sites. |

### Content

| Feature | Decision | User note | Rationale |
|---|---|---|---|
| Pages with block content (JSON) | include (baseline) | — | The core object. |
| Navigation menu per site | include (baseline) | — | Every simple website has one. |
| Draft / publish with revision history | open (round 1) | — | Decides `page_revision` + `published_page` vs a single page row. |
| Scheduled publishing | open (round 1) | — | The one candidate that is a real lifecycle (a processing-object type). |
| Blog posts (dated, listed, tags) | open | — | Could be a page flag or its own table. |
| Contact forms / form submissions | open | — | Public write path + email (later integration). |
| SEO metadata, sitemap.xml | open | — | Cheap columns + one generated route. |

### Delivery

| Feature | Decision | User note | Rationale |
|---|---|---|---|
| Public delivery: app server-renders published pages to HTML | open (round 1) | — | Versus headless JSON only; decides a `delivery` path. |
| Media library (image upload) | open (round 1) | — | Needs object storage → can only be `later`, via the agent. |

### Not considered for this product

E-commerce, multi-language content, comments, analytics — out of a "simple websites" CMS unless the user raises them.

## Open candidates not yet asked

Custom domains; blog posts; contact forms; SEO metadata & sitemap.

## Ready

Not ready — round 1 questions pending.
