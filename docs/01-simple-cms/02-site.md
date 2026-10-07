# 02 \u2014 Nanoservice: site

Owns sites, members, navigation, the theme catalog, and one lifecycle: the custom `domain`.

## Tables (Directive \u00a72)

### `site`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | minted here (Directive \u00a77: the receiver mints ids) |
| slug | string, unique | `^[a-z0-9-]{1,63}$`; public URL `/s/{slug}/` on the platform host |
| name | string | |
| theme_name | string | a `theme` config object in namespace `default` |
| created_at | timestamp | |

Invariants: slug unique (`ON CONFLICT` \u2192 `AlreadyExists`; no advisory locks, \u00a77); `theme_name` resolves in the config repository at write time. Sites are never deleted (excluded in round 3).

### `site_member`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| site_id | uuid | FK \u2192 site.id (same schema) |
| identity_id | uuid | Kratos identity id |
| role | string | `owner` \| `editor` |
| created_at | timestamp | |

Invariants: unique `(site_id, identity_id)`; **every site has \u2265 1 owner** \u2014 the creator is inserted in the site's transaction, and remove/demote locks the site's member rows `FOR UPDATE` and refuses to remove the last owner.

Roles: `owner` \u2014 everything, incl. site settings, members, navigation, domains; `editor` \u2014 pages, posts, move, publish, schedule, preview, form inbox.

### `nav_item`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| site_id | uuid | FK \u2192 site.id |
| label | string | |
| page_path | string | a seam by *name* (\u00a77): the page's full path without leading slash, e.g. `about/team`; `home` and `blog` are valid |
| position | int32 | |

`SaveNavigation` replaces the whole menu in one transaction (delete + insert). A dangling `page_path` is allowed (404 until a page lives there); since redirects are excluded, moving a page does not update the menu \u2014 the admin UI compares the menu with `ListPages` and flags broken items.

### Queries (`repository.rs`)

Insert site + owner (one tx); get site by id / slug; update name/theme; list sites for an identity; member get/list/insert/delete with the last-owner guard; replace/list nav items; **resolve host \u2192 site**: join `domain_spec` \u2192 `domain_status` (own schema) where `phase = 'verified'` and the envelope is not deleting (a read of its own type's tables; status is never *written* here, \u00a74.3).

## Processing-object type `domain` (Directive \u00a73, \u00a75)

One hostname attached to one site, verified by a DNS TXT record the owner creates: `_basable-verify.<hostname>` = `basable-verify=<verification_token>`.

**Spec** (`domain_spec`) \u2014 all immutable; changing a hostname is remove + add.
| Field | Type | Notes |
|---|---|---|
| site_id | uuid | immutable |
| hostname | string | immutable, lowercased, IDNA-normalised; must not end in `basable.com`; **unique index on `hostname`** (added by the implementing session in the migration; a second site cannot claim a hostname still held, including while it is being torn down) |
| verification_token | string | immutable, 32 random bytes base32, minted by the `AddDomain` handler before `TypedStore::create` |

**Status** (`domain_status`)
| Field | Type |
|---|---|
| phase | string `pending` \| `verified` \| `unverified` \| `failed` |
| verified_at | timestamp, nullable |
| last_checked_at | timestamp, nullable |
| consecutive_misses | int32 |
| last_error | string, nullable |

**Reconcile pass** (level-triggered, one `Outcome`):
1. `meta.deleting()` \u2192 nothing external was created (the app never wrote DNS), so absence is trivially confirmed; host resolution already excludes deleting rows \u2192 `Delete`.
2. Look up the TXT record (a read through a `DnsResolver` trait, no transaction held, \u00a73.7). Resolver error (SERVFAIL, timeout) \u2192 `Retry { last_error }`.
3. Record matches the token \u2192 `Converged { phase: verified, verified_at: keep-or-now, consecutive_misses: 0 }`, rescheduled at `resync` (24 h re-check).
4. Record absent and phase was `verified` \u2192 `consecutive_misses + 1`; under 3 \u2192 `Converged { phase: verified }.after(1h)` (a DNS blip does not take a live site down); at 3 \u2192 `Converged { phase: unverified }.after(10m)` (host routing stops).
5. Record absent, never verified: if `now - meta.generation_changed_at` < 7 days \u2192 `Converged { phase: pending }.after(5m)`; else \u2192 `Blocked { phase: failed, cause: "TXT record not found after 7 days" }`. `VerifyDomain` ("check again") is a `nudge`, which re-arms a blocked object.

**Teardown**: `RemoveDomain` \u2192 `mark_deleted` \u2192 pass returns `Delete`. **Worker policy** (identical on every replica, \u00a74.4): `resync` 24 h, `backoff` 30 s \u2192 30 min, `max_attempts` 0 (resolver failures are transient; the 7-day give-up is the level rule in step 5), `attempt_timeout` 30 s.

**Platform dependency**: verification makes the app *willing* to serve the host. Traffic only arrives once the platform's gateway routes that hostname to this app with a TLS certificate. See `05-deployment.md`; until confirmed, the UI shows the platform URL `/s/{slug}/` beside each domain.

## External reads (not adapters, Directive \u00a76)

- Kratos **admin** API `GET /admin/identities?credentials_identifier=<email>` for `AddSiteMember` \u2014 unknown email \u2192 `NotFound` ("ask them to sign up first").
- DNS TXT lookups for `domain` (hickory resolver behind a trait, stubbed in tests).

Both run outside any transaction (\u00a73.7).

## Config types (Directive \u00a71)

`theme` (prefix `thm`): `layout_html` (slots `{{head}}`, `{{site_name}}`, `{{title}}`, `{{nav}}`, `{{breadcrumbs}}`, `{{body}}`), `css`. Seeded `default` and `minimal`. Written only by humans via `config/base/`.

## Messages handled (Directive \u00a77)

- `CreateSiteRequest`, `UpdateSiteRequest`, `GetSiteRequest` \u2192 `Site`; `ListSitesRequest \u2192 SiteList` \u2014 identity from the Kratos session; update requires `owner`.
- `AddSiteMemberRequest`, `RemoveSiteMemberRequest` \u2192 `SiteMember` (owner); `ListSiteMembersRequest \u2192 SiteMemberList` (any member).
- `SaveNavigationRequest \u2192 Navigation` (owner).
- `AddDomainRequest \u2192 Domain` (owner; `TypedStore::create`, returns the TXT record to set), `RemoveDomainRequest \u2192 Domain` (owner; `mark_deleted`), `ListDomainsRequest \u2192 DomainList`, `VerifyDomainRequest \u2192 Domain` (owner; `nudge`).
- `AuthorizeSiteAccessRequest \u2192 SiteAccess` \u2014 `{site_id, identity_id, min_role}` \u2192 `{allowed, role}`; answered from `site_member` only. Sent by `content`, `forms`.
- `ResolvePublicSiteRequest \u2192 PublicSite` \u2014 exactly one of `{host, slug, site_id}` \u2192 `{site_id, slug, name, canonical_base_url, layout_html, css, nav[]}`: the *effective* theme values and the canonical URL (verified domain if any, else the platform URL) \u2014 derived state only (\u00a77). Unauthenticated. Sent by `delivery`, `forms`.

## API

`SiteService`: `CreateSite`, `UpdateSite`, `GetSite`, `ListSites`, `AddSiteMember`, `RemoveSiteMember`, `ListSiteMembers`, `SaveNavigation`, `AddDomain`, `RemoveDomain`, `ListDomains`, `VerifyDomain`.

## Directive sections that bind

\u00a72 for every table; \u00a73, \u00a74, \u00a75 for `domain`; \u00a76 (reads are not adapters) for the Kratos and DNS lookups; \u00a77 for `AuthorizeSiteAccess` and `ResolvePublicSite`; \u00a710 for `CLAUDE.md` / `flows.md`.

## Tests

Unit: slug and hostname validation; menu ordering; nav path validation; the `domain` pass decision table (deleting, resolver error, match, blip under/at 3 misses, pending, 7-day give-up). Integration (testkit): create site makes creator owner; duplicate slug rejected; editor cannot change site/members/nav/domains; last owner cannot be removed (two concurrent removals leave one owner); nav replace atomic; `ResolvePublicSite` by slug / by verified host / unverified host \u2192 NotFound; duplicate hostname across sites rejected; remove domain deletes it; nudge re-arms a blocked domain. Kratos admin and DNS are stubbed by traits.
