# 02 — Nanoservice: delivery

Renders the public website and editor previews. **Owns nothing**: no tables, no processing-object types, no config, no external effects — so no schema, no role and no pool (Directive §9 makes a pool for it a compile error). The cheapest shape that fits (§1): a stateless renderer over messenger requests.

## `RenderPathRequest → RenderedDocument`

`{host, site_slug?, path}` → `{status_code, content_type, body, cache_control}`.

1. Send `ResolvePublicSiteRequest` to `site` — by `site_slug` on the platform host, else by `host`. NotFound → plain 404.
2. Normalise the path (lowercase, strip leading/trailing `/`, reject `..` and empty segments), then route it:
   - empty (root) → page path `home`.
   - `blog`, `blog?after=<cursor>` → `ListPublishedPagesRequest{mode: posts}`; `blog/tag/{tag}` → same with `tag`.
   - `sitemap.xml` → `ListPublishedPagesRequest{mode: all}` (paged until done; a few hundred entries), absolute URLs on `canonical_base_url`, `lastmod` = `updated_at`.
   - `robots.txt` → `User-agent: *`, `Allow: /`, `Sitemap: <canonical_base_url>/sitemap.xml`.
   - anything else, any depth (`about/team`) → `GetPublishedPageRequest{site_id, path}` to `content`.
   - unpublished / unknown → 404 rendered inside the site's layout.
3. Render: each block to HTML with **all text escaped** and URLs restricted to `http(s):` or internal paths; a `contact_form` block becomes the plain POST form (see `02-forms.md`) with a thank-you or error notice from `?sent=1` / `?error=`; posts show date and tag links; `{{breadcrumbs}}` from the page's ancestors (unpublished ancestors shown unlinked). `{{head}}` gets `<title>`, `meta description`, `og:title`, `og:description`, `og:image`, `<link rel=canonical>` on `canonical_base_url`; `{{nav}}` marks the current page and its ancestors.

All sends are remote I/O with no transaction held (§3.7); read-only requests carry no generation (§7).

## `PreviewPageRequest → RenderedDocument`

`{page_id, revision_id?, identity_id}` (identity from the Kratos session in the api crate). Sends `GetPreviewPageRequest` to `content` (which authorizes against `site`), then `ResolvePublicSiteRequest{site_id}`, then renders exactly as above with `X-Robots-Tag: noindex` and `Cache-Control: no-store`. The admin UI shows it in a sandboxed `iframe srcdoc`.

## Public routes (raw routes on the api crate, one messenger send each)

- Platform host: `GET /s/{site_slug}/` and `GET /s/{site_slug}/{*path}`.
- Custom domain (any host that is not the platform host): `GET /{*path}`.

Response `text/html` (or `application/xml`, `text/plain`) with `Cache-Control: public, max-age=60`. These are not manifest `webhooks` (no signature): the implementing session adds them by hand in the api crate, next to the forms POST route.

## API

`PublicService`: `RenderPath` (the admin's "view live"), `PreviewPage`.

## Directive sections that bind

§7 for every send; §9 (no in-memory render cache in v1); §10 docs.

## Tests

Unit: block rendering and escaping (`<script>`, `javascript:` URLs), head/meta generation, breadcrumbs, sitemap XML, robots, path normalisation and routing table (`..`, double slashes, nested paths, reserved prefixes), nav current-page marking. Integration (all four nanoservices registered): published page renders; nested page renders at `/about/team` with breadcrumbs; unpublished 404s; unknown site 404s; a draft saved after publishing does not show but does in preview; preview by a non-member is denied; blog index pages and tag filter; sitemap lists only live pages with canonical URLs.
