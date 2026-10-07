# 00 — Scope: simple-cms

## The idea, in the user's words

> i want a cms system for simple websites

## Fit gate

**Fits.** A CMS is a backend with its state in Postgres (sites, members, domains, pages, revisions, publication state, form submissions) driven through a CRUD + workflow API, with an admin UI on top and server-rendered public pages. There is no binary media store (media library excluded), so Postgres is the only source of truth.

## Working interpretation

A user signs in (Kratos, passwordless email code / passkey), creates one or more websites, adds existing accounts as owners or editors, and optionally attaches a custom domain verified through a DNS TXT record. Editors write pages and blog posts as JSON blocks (heading, paragraph, image-by-URL, link, contact form), with SEO metadata. Every save is a revision. They preview a draft exactly as visitors will see it, then publish now or schedule publish/unpublish at a future time, and arrange a navigation menu. Visitors get real HTML pages rendered by the app with the site's theme, a blog index with tag pages, `sitemap.xml` and `robots.txt`, and can send a contact form whose submissions editors read in the admin. "Simple websites" = brochure sites, small business sites, portfolios and small blogs: tens to a few hundred pages per site, a handful of editors.

## Scope ledger

Decisions: `include` / `exclude` / `later` / `open` (not yet asked).

### Sites & access

| Feature | Decision | User note | Rationale |
|---|---|---|---|
| Sites (create, rename, choose theme) | include (baseline) | — | Without a site there is nothing to manage. |
| Editors via Kratos login | include (baseline) | — | Standard auth; an editor is a Kratos identity. |
| Multiple sites per account, per-site members & roles | **include** (round 1) | — | `site_member` with `owner` / `editor`; every editor request is authorized by `site`. |
| Add a member by email of an existing account | include (follows from multi-site) | — | A Kratos admin lookup (a read, not an effect). Emailing invitations to people without an account needs SMTP → not now. |
| Custom domains per site | **include** (round 2) | — | `site` gains the `domain` processing-object type: it polls DNS for a TXT record until verified and keeps re-checking. Serving traffic on the domain also needs the platform to accept the hostname and issue TLS — a platform dependency, see `05-deployment.md`. |
| Themes (layout + CSS as a config catalog) | include (baseline) | — | A seeded `theme` catalog, selected per site. |

### Content

| Feature | Decision | User note | Rationale |
|---|---|---|---|
| Pages with block content (JSON) | include (baseline) | — | The core object. |
| Navigation menu per site | include (baseline) | — | Every simple website has one. |
| Draft / publish with revision history | **include** (round 1) | — | `page_revision` (append-only) + `published_page` pointer; rollback = publish an older revision. |
| Scheduled publish / unpublish | **include** (round 1) | — | The real lifecycle (wait until a time, then drive the live pointer): processing-object type `publication` in `content`. |
| Media library (uploads) | **exclude** (round 1) | — | Images are linked by external URL; no object storage. |
| Draft preview for editors | **include** (round 2) | — | `delivery` renders any revision with the theme, behind login; `content` authorizes it. |
| Blog posts (dated, tagged, listed) | **include** (round 2) | — | `page.kind = post`; post date and tags are revisioned fields; `delivery` renders `/blog` and `/blog/tag/{tag}`, paginated. |
| SEO metadata, sitemap.xml, robots.txt | **include** (round 2) | — | Meta description and social image per revision; `delivery` generates the sitemap from the published index. |

### Visitors

| Feature | Decision | User note | Rationale |
|---|---|---|---|
| Public delivery: server-rendered HTML with the theme | **include** (round 1) | — | `delivery`: a stateless renderer over `site` and `content`, keyed by host + path. |
| Contact forms with stored submissions | **include** (round 2) | — | New `forms` nanoservice: a `contact_form` block, a public POST route, submissions table, per-IP rate limit, honeypot. |
| Email alert on new submission | later (implied by round 2) | — | Needs an SMTP/email provider credential; added later via the agent as a `keyed_replay` adapter in `forms`. |

### Not considered for this product

E-commerce, multi-language content, comments, analytics — out of a "simple websites" CMS unless the user raises them.

## Open candidates not yet asked

- Page hierarchy (nested pages, `/about/team` URLs)
- Redirects when a slug changes (and manual redirects)
- Activity log per site (who published what, when)
- Deleting a site (and everything under it)

## Ready

Not ready — round 3 questions pending.
