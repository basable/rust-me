# 01 \u2014 Architecture: simple-cms

Four nanoservices in one Rust binary. Two lifecycles pass the Directive \u00a71 question ("waiting, polling a third party until it finishes, scheduled work, driving another system to match"):

- a **scheduled publication** waits until its time, then drives a page's live pointer \u2192 `content.publication`;
- a **custom domain** polls DNS until its TXT record appears and keeps re-checking it \u2192 `site.domain`.

Everything else (saving, moving, publishing now, membership, navigation, form submissions, rendering) is a single transaction or a pure read, so it is plain tables or no state at all.

## site

Owns what a website *is*, who may touch it, and where it is served. **Plain tables** (Directive \u00a72): `site`, `site_member` (Kratos identity \u2192 `owner` | `editor`), `nav_item` (ordered menu, linking pages by *path*). **Processing-object type** `domain` (prefix `dom`): one hostname attached to one site; it needs a type because verification *waits on a third party* (the owner's DNS) for minutes to days, must be re-checked periodically, and must survive restarts and replicas (Directive \u00a73, \u00a75). **Config catalog** `theme`. **No external effects**: the Kratos admin lookup (add member by email) and DNS TXT lookups are reads (Directive \u00a76). Handles the site / member / nav / domain API requests plus two internal requests: `AuthorizeSiteAccessRequest \u2192 SiteAccess` (from `content` and `forms`) and `ResolvePublicSiteRequest \u2192 PublicSite` (from `delivery` and `forms`: by host, slug or id \u2192 the site, its effective theme, its menu; derived state only, Directive \u00a77). API: `SiteService`.

## content

Owns pages, posts and what is live. **Plain tables**: `page` (kind `page` | `post`, `parent_id`, materialised `path` unique per site), `page_revision` (append-only: title, blocks, SEO fields, post date, tags), `published_page` (the live pointer). **Processing-object type** `publication` (prefix `pub`): one scheduled publish or unpublish; it waits (possibly days), can be rescheduled or cancelled, and must fire once on whichever replica claims it (Directive \u00a73, \u00a75). Sends `AuthorizeSiteAccessRequest` to `site` before every editor request (request path, outside any transaction, \u00a73.7). Handles the page API (incl. `MovePageRequest`) plus three read requests from `delivery`: `GetPublishedPageRequest \u2192 PublishedPage` (by path), `ListPublishedPagesRequest \u2192 PublishedPageIndex` (posts list, tag pages, sitemap) and `GetPreviewPageRequest \u2192 PublishedPage`. API: `ContentService`.

## forms

Owns what visitors send. **Plain tables**: `form_submission`, `form_rate_limit`. **Schedule** `purge_rate_limits` (hourly ticker). No lifecycle (a submission is stored in one transaction). No external effects yet \u2014 the email alert is `later` and will be a `keyed_replay` adapter here. Handles `SubmitContactFormRequest \u2192 FormSubmissionReceipt` from the public POST route, and the editor inbox API. Sends `ResolvePublicSiteRequest` and `AuthorizeSiteAccessRequest` to `site`. API: `FormsService`.

## delivery

Renders the public website. **Owns nothing** \u2014 no tables, no types, no effects, so no schema and no pool (Directive \u00a79). Handles `RenderPathRequest \u2192 RenderedDocument` ({host, path} \u2192 page by full path, `/blog`, `/blog/tag/{tag}`, `/sitemap.xml`, `/robots.txt`, 404) and `PreviewPageRequest \u2192 RenderedDocument`. Sends `ResolvePublicSiteRequest` to `site`, and `GetPublishedPageRequest`, `ListPublishedPagesRequest`, `GetPreviewPageRequest` to `content`. Visitors reach it through raw routes on the api crate (one messenger send each); editors through `PublicService`. No render cache in v1 (\u00a79).

## Deployment shape

One binary (`basable-app`) with all four nanoservices; one CloudNativePG cluster (`instances: 1`) with schemas `site`, `content` and `forms` plus a `kratos` database; Kratos for editor identity; a **static** admin frontend (plain HTML/CSS/JS on nginx-unprivileged \u2014 the block editor is an ordered form of blocks with a preview iframe and a page tree, not an editor-like surface that justifies React). Routing on the platform host: `/api/*` and `/s/*` \u2192 binary, `/.ory/*` \u2192 Kratos, `/*` \u2192 nginx; on a verified custom domain every path \u2192 binary (platform dependency). App replicas: 1 on the free tier. Details in `05-deployment.md`.

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
| AddDomainRequest | api | site | Domain (1:1) |
| RemoveDomainRequest | api | site | Domain (1:1) |
| ListDomainsRequest | api | site | DomainList (1:1) |
| VerifyDomainRequest | api | site | Domain (1:1) |
| AuthorizeSiteAccessRequest | content, forms | site | SiteAccess (1:1) |
| ResolvePublicSiteRequest | delivery, forms | site | PublicSite (1:1) |
| CreatePageRequest | api | content | Page (1:1) |
| SavePageRequest | api | content | Page (1:1) |
| MovePageRequest | api | content | Page (1:1) |
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
| ListPublishedPagesRequest | delivery | content | PublishedPageIndex (1:1) |
| GetPreviewPageRequest | delivery | content | PublishedPage (1:1) |
| SubmitContactFormRequest | api (raw route) | forms | FormSubmissionReceipt (1:1) |
| ListFormSubmissionsRequest | api | forms | FormSubmissionList (1:1) |
| MarkFormSubmissionReadRequest | api | forms | FormSubmission (1:1) |
| DeleteFormSubmissionRequest | api | forms | FormSubmission (1:1) |
| RenderPathRequest | api (raw route + Connect) | delivery | RenderedDocument (1:1) |
| PreviewPageRequest | api | delivery | RenderedDocument (1:1) |

No events and no fan-out: nothing downstream must happen when a page is published, moved or a form is submitted (delivery reads on demand; a moved page's nav links are the owner's to fix \u2014 the admin flags nav items whose path no longer resolves).
