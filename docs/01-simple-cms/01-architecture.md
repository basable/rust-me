# 01 \u2014 Architecture: simple-cms

Three nanoservices in one Rust binary. Exactly one lifecycle passes the Directive \u00a71 question ("waiting \u2026 scheduled work \u2026 driving another system to match"): a **scheduled publication** waits until its time and then drives the page's live pointer. Everything else (saving, publishing now, membership, navigation, rendering) is a single transaction or a pure read, so it is plain tables or no state at all.

## site

Owns what a website *is* and who may touch it. **Plain tables** (Directive \u00a72): `site`, `site_member` (Kratos identity \u2192 `owner` | `editor` per site), `nav_item` (the ordered menu). **Config catalog**: `theme` (layout HTML + CSS, seeded under `config/base/`, Directive \u00a71). **No external effects**: adding a member by email does a Kratos admin *lookup* (a read, Directive \u00a76: reads are not adapters). Handles the site/member/nav API requests, plus two internal requests: `AuthorizeSiteAccessRequest \u2192 SiteAccess` (from `content`) and `ResolvePublicSiteRequest \u2192 PublicSite` (from `delivery`: the site, its effective theme layout/CSS and its menu \u2014 derived state only, Directive \u00a77). API: `SiteService`.

## content

Owns pages and what is live. **Plain tables**: `page`, `page_revision` (append-only JSON bodies), `published_page` (the live pointer per page; `revision_id` null = unpublished). **Processing-object type** `publication` (prefix `pub`): one scheduled publish or unpublish of one page at `publish_at`. It needs a type because it *waits* (possibly days), must survive restarts and replicas, can be rescheduled or cancelled, and must fire exactly once on whichever replica claims it \u2014 Directive \u00a73, \u00a75 bind. Sends `AuthorizeSiteAccessRequest \u2192 SiteAccess` to `site` before every editor request, from the request path and outside any transaction (Directive \u00a73.7). Handles the page API and `GetPublishedPageRequest \u2192 PublishedPage` from `delivery`. API: `ContentService`.

## delivery

Renders the public website. **Owns nothing** \u2014 no tables, no types, no external effects, so no schema and no pool (Directive \u00a79). Handles `RenderPageRequest \u2192 RenderedPage`: sends `ResolvePublicSiteRequest` to `site` and `GetPublishedPageRequest` to `content`, then fills the theme's layout with title, menu and the escaped block HTML. Exposed to visitors through a raw `GET /s/{site_slug}/{page_slug}` route on the api crate (and `/s/{site_slug}/` \u2192 page slug `home`) which makes ONE messenger send; also as `PublicService.RenderPage` for the editor UI. No caching in v1 (a cache would be in-memory state, Directive \u00a79).

## Deployment shape

One binary (`basable-app`) with all three nanoservices; one CloudNativePG cluster (`instances: 1`) with schemas `site` and `content` plus a `kratos` database; Kratos for editor identity; a **static** admin frontend (plain HTML/CSS/JS on nginx-unprivileged \u2014 the block editor is a form of ordered blocks, not an editor-like surface that justifies React). Routing: `/api/*` and `/s/*` \u2192 binary, `/.ory/*` \u2192 Kratos, `/*` \u2192 nginx (the `/s/*` match is a deviation, see `05-deployment.md`). App replicas: 1 on the free tier (`05-deployment.md`).

## Topology

| Message | Sender(s) | Handler | Response kind |
|---|---|---|---|
| CreateSiteRequest | api | site | Site (1:1) |
| UpdateSiteRequest | api | site | Site (1:1) |
| GetSiteRequest | api | site | Site (1:1) |
| ListSitesRequest | api | site | SiteList (1:1) |
| AddSiteMemberRequest | api | site | SiteMember (1:1) |
| RemoveSiteMemberRequest | api | site | SiteMember (1:1) |
| ListSiteMembersRequest | api | site | SiteMemberList (1:1) |
| SaveNavigationRequest | api | site | Navigation (1:1) |
| AuthorizeSiteAccessRequest | content | site | SiteAccess (1:1) |
| ResolvePublicSiteRequest | delivery | site | PublicSite (1:1) |
| CreatePageRequest | api | content | Page (1:1) |
| SavePageRequest | api | content | Page (1:1) |
| GetPageRequest | api | content | Page (1:1) |
| ListPagesRequest | api | content | PageList (1:1) |
| ListRevisionsRequest | api | content | RevisionList (1:1) |
| PublishPageRequest | api | content | Page (1:1) |
| UnpublishPageRequest | api | content | Page (1:1) |
| SchedulePublicationRequest | api | content | Publication (1:1) |
| ReschedulePublicationRequest | api | content | Publication (1:1) |
| CancelPublicationRequest | api | content | Publication (1:1) |
| ListPublicationsRequest | api | content | PublicationList (1:1) |
| GetPublishedPageRequest | delivery | content | PublishedPage (1:1) |
| RenderPageRequest | api (raw route + Connect) | delivery | RenderedPage (1:1) |

No events and no fan-out: nothing downstream must happen when a page is published (delivery reads on demand).
