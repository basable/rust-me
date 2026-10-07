# 02 — Nanoservice: site

Plain executor. No processing-object types, no external calls.

## Tables (Directive §2)

### `site`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | minted here (Directive §7: the receiver mints ids) |
| slug | string, unique | url-safe, `^[a-z0-9-]{1,63}$` |
| name | string | |
| theme_name | string | name of a `theme` config object in namespace `default` |
| created_at | timestamp | |

Invariants: slug unique (`ON CONFLICT` → `AlreadyExists`); `theme_name` must resolve in the config repository at write time.

### `site_member`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| site_id | uuid | → site.id (same schema FK) |
| identity_id | uuid | Kratos identity id |
| role | string | `owner` \| `editor` |

Invariants: unique `(site_id, identity_id)`; every site has ≥ 1 `owner` (the creator, inserted in the same transaction as the site). Whether more members/roles exist depends on round 1.

### `nav_item`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| site_id | uuid | |
| label | string | |
| page_slug | string | a seam by name, not a `content` id (Directive §7) |
| position | int32 | |

`SaveNavigation` replaces the whole menu in one transaction (delete + insert), so order is never half-written.

Queries: insert site + owner; get by id/slug; list sites for an identity (join `site_member`, same schema); member lookup for authorization; replace/list nav items.

## Config types (Directive §1)

`theme` (prefix `thm`): `layout_html` (template with `{{title}}`, `{{nav}}`, `{{body}}` slots), `css`. Seeded with one `default` theme. Read by `site` (validation) — and by delivery if server-rendering is included. Written only by humans via `config/base/`.

## Messages handled (Directive §7)

- `CreateSiteRequest → Site` — caller identity becomes owner.
- `GetSiteRequest → Site`, `ListSitesRequest → SiteList`, `SaveNavigationRequest → Navigation` — each checks membership.
- `AuthorizeSiteAccessRequest → SiteAccess` — `{site_id, identity_id}` → `{allowed, role}`; answered from `site_member` only.

## API

`SiteService`: `CreateSite`, `GetSite`, `ListSites`, `SaveNavigation`.

## Tests

Unit: slug validation, menu ordering. Integration (testkit): create site makes owner; duplicate slug rejected; non-member denied; nav replace is atomic.
