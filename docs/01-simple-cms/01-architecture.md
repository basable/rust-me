# 01 — Architecture: simple-cms

Two nanoservices in one Rust binary. Neither owns a processing-object type yet: by the Directive §1 question, nothing here so far is a multi-step lifecycle (saving and publishing a page are single transactions). If **scheduled publishing** is included, `content` grows one type, `publication`.

## site

Owns what a website *is* and who may edit it. **Plain tables** (Directive §2): `site` (identity, slug, name, chosen theme), `site_member` (Kratos identity → role per site), `nav_item` (the ordered navigation menu). **Config catalog**: `theme` (layout HTML + CSS, seeded under `config/base/`, edited by a person, loaded at boot — Directive §1). No external systems. Handles `CreateSiteRequest → Site`, `GetSiteRequest → Site`, `ListSitesRequest → SiteList`, `SaveNavigationRequest → Navigation` from the api crate, and `AuthorizeSiteAccessRequest → SiteAccess` from `content` (the only cross-nanoservice message: `content` never reads `site_member`, Directive §2/§7). API: `SiteService`.

## content

Owns the pages and their publication. **Plain tables**: `page` (slug, title, per site), `page_revision` (immutable JSON block body per save), `published_page` (which revision is live per page). No external systems yet (media is later). Handles `SavePageRequest → Page`, `GetPageRequest → Page`, `ListPagesRequest → PageList`, `PublishPageRequest → Page`. Sends `AuthorizeSiteAccessRequest → SiteAccess` to `site` before every mutation — from the request path, outside any transaction (Directive §3.7). API: `ContentService`.

## Deployment shape

One binary (`basable-app`) with both nanoservices, one CloudNativePG cluster (`instances: 1`) with schemas `site` and `content` plus a `kratos` database, Kratos for identity, and a **static** admin frontend (plain HTML/CSS/JS on nginx-unprivileged; the page editor is a form of blocks, not an editor-like surface that justifies React). `/api/*` → binary, `/.ory/*` → Kratos, `/*` → nginx. Replicas: see `05-deployment.md` (1 on the free tier).

## Topology

| Message | Sender(s) | Handler | Response kind |
|---|---|---|---|
| CreateSiteRequest | api | site | Site (1:1) |
| GetSiteRequest | api | site | Site (1:1) |
| ListSitesRequest | api | site | SiteList (1:1) |
| SaveNavigationRequest | api | site | Navigation (1:1) |
| AuthorizeSiteAccessRequest | content | site | SiteAccess (1:1) |
| SavePageRequest | api | content | Page (1:1) |
| GetPageRequest | api | content | Page (1:1) |
| ListPagesRequest | api | content | PageList (1:1) |
| PublishPageRequest | api | content | Page (1:1) |
