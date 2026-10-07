# 02 \u2014 Nanoservice: site

Plain executor over three tables and one config catalog. No processing-object types (nothing waits or converges), no external effects.

## Tables (Directive \u00a72)

### `site`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | minted here (Directive \u00a77: the receiver mints ids) |
| slug | string, unique | `^[a-z0-9-]{1,63}$`, used in public URLs `/s/{slug}/` |
| name | string | |
| theme_name | string | a `theme` config object in namespace `default` |
| created_at | timestamp | |

Invariants: slug unique (`ON CONFLICT` \u2192 `AlreadyExists`, Directive \u00a77: no advisory locks); `theme_name` must resolve in the config repository at write time.

### `site_member`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| site_id | uuid | FK \u2192 site.id (same schema), `ON DELETE CASCADE` |
| identity_id | uuid | Kratos identity id |
| role | string | `owner` \\| `editor` |
| created_at | timestamp | |

Invariants: unique `(site_id, identity_id)`; **every site has \u2265 1 owner** \u2014 the creator is inserted in the same transaction as the site, and remove/demote locks the site's member rows `FOR UPDATE` and refuses to remove the last owner.

Roles: `owner` \u2014 everything, incl. site settings, members, navigation; `editor` \u2014 pages, publish, schedule.

### `nav_item`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| site_id | uuid | FK \u2192 site.id, cascade |
| label | string | |
| page_slug | string | a seam by *name*, not a `content` id (Directive \u00a77) |
| position | int32 | |

`SaveNavigation` replaces the whole menu in one transaction (delete + insert); a dangling `page_slug` is allowed (renders as a link that 404s until the page is published).

### Queries (`repository.rs`)

Insert site + owner (one tx); get site by id / by slug; update name/theme; list sites for an identity; member get/list/insert/delete (last-owner guard); replace/list nav items.

## External reads (not adapters, Directive \u00a76)

`AddSiteMember` takes an email and resolves it to a Kratos identity through the Kratos **admin** API (`GET /admin/identities?credentials_identifier=`). Unknown email \u2192 `NotFound` ("ask them to sign up first"). Called from the request path before the insert transaction (Directive \u00a73.7).

## Config types (Directive \u00a71)

`theme` (prefix `thm`): `layout_html` (template with `{{site_name}}`, `{{title}}`, `{{nav}}`, `{{body}}` slots), `css`. Seeded with `default` and `minimal`. Written only by humans via `config/base/`.

## Messages handled (Directive \u00a77)

- `CreateSiteRequest`, `UpdateSiteRequest`, `GetSiteRequest`, `ListSitesRequest` \u2192 `Site`/`SiteList` \u2014 caller identity from the Kratos session; update requires `owner`.
- `AddSiteMemberRequest`, `RemoveSiteMemberRequest`, `ListSiteMembersRequest` \u2014 add/remove require `owner`; list any member.
- `SaveNavigationRequest \u2192 Navigation` \u2014 `owner`.
- `AuthorizeSiteAccessRequest \u2192 SiteAccess` \u2014 `{site_id, identity_id}` \u2192 `{allowed, role}`; answered from `site_member` only. Sent by `content`.
- `ResolvePublicSiteRequest \u2192 PublicSite` \u2014 `{site_slug}` \u2192 `{site_id, name, layout_html, css, nav[]}`: the *effective* theme values, not the theme name (Directive \u00a77: only derived state crosses). Unauthenticated (public). Sent by `delivery`.

## API

`SiteService`: `CreateSite`, `UpdateSite`, `GetSite`, `ListSites`, `AddSiteMember`, `RemoveSiteMember`, `ListSiteMembers`, `SaveNavigation`.

## Directive sections that bind

\u00a72 for every table; \u00a77 for `AuthorizeSiteAccess` and `ResolvePublicSite`; \u00a73.7 for the Kratos lookup; \u00a710 for `CLAUDE.md` / `flows.md`.

## Tests

Unit: slug validation, menu ordering, template slot substitution inputs. Integration (testkit): create site makes creator owner; duplicate slug rejected; editor cannot update site/members/nav; last owner cannot be removed (concurrent removals of two owners leave one); nav replace atomic; `ResolvePublicSite` returns the theme's effective layout; unknown slug \u2192 NotFound. The Kratos admin lookup is stubbed by a trait in tests.
