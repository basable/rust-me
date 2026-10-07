# 02 \u2014 Nanoservice: delivery

Renders the public website. **Owns nothing**: no tables, no processing-object types, no config, no external effects \u2014 so no schema, no role and no pool (Directive \u00a79 makes a pool for it a compile error). It is the cheapest shape that fits (\u00a71): a stateless renderer over two messenger requests.

## Flow of `RenderPageRequest \u2192 RenderedPage`

`{site_slug, page_slug}` \u2192
1. send `ResolvePublicSiteRequest{site_slug}` to `site` \u2192 `{site_id, name, layout_html, css, nav[]}`; NotFound \u2192 404.
2. send `GetPublishedPageRequest{site_id, slug}` to `content` \u2192 `{title, body}`; NotFound or unpublished \u2192 404 rendered inside the site's layout.
3. Render each block to HTML with **all text escaped** and URLs restricted to `http(s):` (no `javascript:`); substitute `{{site_name}}`, `{{title}}`, `{{nav}}`, `{{body}}`; inline the CSS.

Both sends are remote I/O with no transaction held (\u00a73.7). Response: `{status_code, html}`.

## Public route

A raw axum route on the api crate beside the Connect router: `GET /s/{site_slug}/` (page slug `home`) and `GET /s/{site_slug}/{page_slug}`, unauthenticated, makes ONE messenger send (`RenderPageRequest`) and writes `text/html` with `Cache-Control: public, max-age=60`. The HTTPRoute must send `/s/*` to the binary (deviation, `05-deployment.md`). Not a manifest `webhook` (no signature): the implementing session adds the route by hand in the api crate.

## API

`PublicService.RenderPage` \u2014 the same request, for the admin UI's "view live" frame.

## Directive sections that bind

\u00a77 for both sends (no generation needed: read-only requests carry no derived intent); \u00a79 for "no in-memory state" (no render cache in v1); \u00a710 docs.

## Tests

Unit: block rendering and escaping (script tags, `javascript:` URLs), template substitution, nav rendering with the current page marked. Integration: with site and content registered, a published page renders; an unpublished page 404s; an unknown site 404s; a draft saved after publishing does not show.
