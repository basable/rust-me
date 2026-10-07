# 00 \u2014 Scope: simple-cms

## The idea, in the user's words

> i want a cms system for simple websites

## Fit gate

**Fits.** A CMS is a backend with its state in Postgres (sites, members, domains, pages, revisions, publication state, form submissions) driven through a CRUD + workflow API, with an admin UI on top and server-rendered public pages. There is no binary media store (media library excluded), so Postgres is the only source of truth.

## Working interpretation

A user signs in (Kratos, passwordless email code / passkey), creates one or more websites, adds existing accounts as owners or editors, and optionally attaches a custom domain verified through a DNS TXT record. Editors write pages (nestable, e.g. `/about/team`) and blog posts as JSON blocks (heading, paragraph, image-by-URL, link, contact form), with SEO metadata. Every save is a revision. They preview a draft exactly as visitors will see it, then publish now or schedule publish/unpublish at a future time, and arrange a navigation menu. Visitors get real HTML pages rendered by the app with the site's theme, a blog index with tag pages, `sitemap.xml` and `robots.txt`, and can send a contact form whose submissions editors read in the admin. "Simple websites" = brochure sites, small business sites, portfolios and small blogs: tens to a few hundred pages per site, a handful of editors.

## Scope ledger

Decisions: `include` / `exclude` / `later`.

### Sites & access

| Feature | Decision | User note | Rationale |
|---|---|---|---|
| Sites (create, rename, choose theme) | include (baseline) | \u2014 | Without a site there is nothing to manage. |
| Editors via Kratos login | include (baseline) | \u2014 | Standard auth; an editor is a Kratos identity. |
| Multiple sites per account, per-site members & roles | **include** (round 1) | \u2014 | `site_member` with `owner` / `editor`; every editor request is authorized by `site`. |
| Add a member by email of an existing account | include (follows from multi-site) | \u2014 | A Kratos admin lookup (a read, not an effect). Emailing invitations to people without an account needs SMTP \u2192 not now. |
| Custom domains per site | **include** (round 2) | \u2014 | `site.domain` processing-object type polls DNS for a TXT record until verified and keeps re-checking. Serving traffic also needs the platform to route and certify the hostname \u2014 a platform dependency, see `05-deployment.md`. |
| Themes (layout + CSS as a config catalog) | include (baseline) | \u2014 | A seeded `theme` catalog, selected per site. |
| Delete a whole site | **exclude** (round 3) | \u2014 | Avoids a cross-nanoservice teardown; sites can be renamed and emptied. |
| Activity log per site | **exclude** (round 3) | \u2014 | Revision authors are the only audit trail. |

### Content

| Feature | Decision | User note | Rationale |
|---|---|---|---|
| Pages with block content (JSON) | include (baseline) | \u2014 | The core object. |
| Navigation menu per site | include (baseline) | \u2014 | Every simple website has one; items link by page path. |
| Draft / publish with revision history | **include** (round 1) | \u2014 | `page_revision` (append-only) + `published_page` pointer; rollback = publish an older revision. |
| Scheduled publish / unpublish | **include** (round 1) | \u2014 | The real lifecycle (wait until a time, then drive the live pointer): `content.publication`. |
| Media library (uploads) | **exclude** (round 1) | \u2014 | Images are linked by external URL; no object storage. |
| Draft preview for editors | **include** (round 2) | \u2014 | `delivery` renders any revision with the theme, behind login; `content` authorizes it. |
| Blog posts (dated, tagged, listed) | **include** (round 2) | \u2014 | `page.kind = post`; post date and tags are revisioned; `delivery` renders `/blog` and `/blog/tag/{tag}`. |
| SEO metadata, sitemap.xml, robots.txt | **include** (round 2) | \u2014 | Meta description and social image per revision; sitemap from the published index. |
| Nested pages (`/about/team`) | **include** (round 3) | \u2014 | `page.parent_id` + a materialised `path`, unique per site; moving a page rewrites its subtree's paths in one transaction. Posts stay top-level. |
| Redirects on slug change / manual redirects | **exclude** (round 3) | \u2014 | Old URLs 404 after a move or rename; the editor UI warns before moving a published page. |

### Visitors

| Feature | Decision | User note | Rationale |
|---|---|---|---|
| Public delivery: server-rendered HTML with the theme | **include** (round 1) | \u2014 | `delivery`: a stateless renderer over `site` and `content`, keyed by host + path. |
| Contact forms with stored submissions | **include** (round 2) | \u2014 | `forms` nanoservice: a `contact_form` block, a public POST route, submissions, per-IP rate limit, honeypot. |
| Email alert on new submission | later (implied by round 2) | \u2014 | Needs an email-provider credential; added later via the agent as a `keyed_replay` adapter in `forms`. |

### Not considered for this product

E-commerce, multi-language content, comments, analytics \u2014 out of a "simple websites" CMS unless the user raises them.

## Open candidates not yet asked

None load-bearing. Marginal ideas left for later sessions: deleting a single page (vs unpublishing), page templates, per-site custom CSS overrides, search on the public site.

## Ready

**ready** \u2014 scope settled after round 3. The manifest is complete for four nanoservices (`site`, `content`, `forms`, `delivery`); Implement can render it.
