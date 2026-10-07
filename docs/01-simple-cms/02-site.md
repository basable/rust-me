# 02 — Nanoservice: site

Owns sites, members, navigation, the theme catalog, and one lifecycle: the custom `domain`.

## Tables (Directive §2)

### `site`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | minted here (Directive §7: the receiver mints ids) |
| slug | string, unique | `^[a-z0-9-]{1,63}$`; public URL `/s/{slug}/` on the platform host |
| name | string | |
| theme_name | string | a `theme` config object in namespace `default` |
| created_at | timestamp | |

Invariants: slug unique (`ON CONFLICT` → `AlreadyExists`; no advisory locks, §7); `theme_name` resolves in the config repository at write time. Sites are never deleted (excluded in round 3).

### `site_member`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| site_id | uuid | FK → site.id (same schema) |
| identity_id | uuid | Kratos identity id |
| role | string | `owner` \| `editor` |
| created_at | timestamp | |

Invariants: unique `(site_id, identity_id)`; **every site has ≥ 1 owner** — the creator is inserted in the site's transaction, and remove/demote locks the site's member rows `FOR UPDATE` and refuses to remove the last owner.

Roles: `owner` — everything, incl. site settings, members, navigation, domains; `editor` — pages, posts, move, publish, schedule, preview, form inbox.

### `nav_item`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| site_id | uuid | FK → site.id |
| label | string | |
| page_path | string | a seam by *name* (§7): the page's full path without leading slash, e.g. `about/team`; `home` and `blog` are valid |
| position | int32 | |

`SaveNavigation` replaces the whole menu in one transaction (delete + insert). A dangling `page_path` is allowed (404 until a page lives there); since redirects are excluded, moving a page does not update the menu — the admin UI compares the menu with `ListPages` and flags broken items.

### Queries (`repository.rs`)

Insert site + owner (one tx); get site by id / slug; update name/theme; list sites for an identity; member get/list/insert/delete with the last-owner guard; replace/list nav items; **resolve host → site**: join `domain_spec` → `domain_status` (own schema) where `phase = 'verified'` and the envelope is not deleting (a read of its own type's tables; status is never *written* here, §4.3).

## Processing-object type `domain` (Directive §3, §5)

One hostname attached to one site, verified by a DNS TXT record the owner creates: `_basable-verify.<hostname>` = `basable-verify=<verification_token>`.

**Spec** (`domain_spec`) — all immutable; changing a hostname is remove + add.
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
1. `meta.deleting()` → nothing external was created (the app never wrote DNS), so absence is trivially confirmed; host resolution already excludes deleting rows → `Delete`.
2. Look up the TXT record (a read through a `DnsResolver` trait, no transaction held, §3.7). Resolver error (SERVFAIL, timeout) → `Retry { last_error }`.
3. Record matches the token → `Converged { phase: verified, verified_at: keep-or-now, consecutive_misses: 0 }`, rescheduled at `resync` (24 h re-check).
4. Record absent and phase was `verified` → `consecutive_misses + 1`; under 3 → `Converged { phase: verified }.after(1h)` (a DNS blip does not take a live site down); at 3 → `Converged { phase: unverified }.after(10m)` (host routing stops).
5. Record absent, never verified: if `now - meta.generation_changed_at` < 7 days → `Converged { phase: pending }.after(5m)`; else → `Blocked { phase: failed, cause: "TXT record not found after 7 days" }`. `VerifyDomain` ("check again") is a `nudge`, which re-arms a blocked object.

**Teardown**: `RemoveDomain` → `mark_deleted` → pass returns `Delete`. **Worker policy** (identical on every replica, §4.4): `resync` 24 h, `backoff` 30 s → 30 min, `max_attempts` 0 (resolver failures are transient; the 7-day give-up is the level rule in step 5), `attempt_timeout` 30 s.

**Platform dependency**: verification makes the app *willing* to serve the host. Traffic only arrives once the platform's gateway routes that hostname to this app with a TLS certificate. See `05-deployment.md`; until confirmed, the UI shows the platform URL `/s/{slug}/` beside each domain.

## External reads (not adapters, Directive §6)

- Kratos **admin** API `GET /admin/identities?credentials_identifier=<email>` for `AddSiteMember` — unknown email → `NotFound` ("ask them to sign up first").
- DNS TXT lookups for `domain` (hickory resolver behind a trait, stubbed in tests).

Both run outside any transaction (§3.7).

## Config types (Directive §1)

`theme` (prefix `thm`): `layout_html` (slots `{{head}}`, `{{site_name}}`, `{{title}}`, `{{nav}}`, `{{breadcrumbs}}`, `{{body}}`), `css`. Seeded `default` and `minimal`. Written only by humans via `config/base/`.

## Messages handled (Directive §7)

- `CreateSiteRequest`, `UpdateSiteRequest`, `GetSiteRequest` → `Site`; `ListSitesRequest → SiteList` — identity from the Kratos session; update requires `owner`.
- `AddSiteMemberRequest`, `RemoveSiteMemberRequest` → `SiteMember` (owner); `ListSiteMembersRequest → SiteMemberList` (any member).
- `SaveNavigationRequest → Navigation` (owner).
- `AddDomainRequest → Domain` (owner; `TypedStore::create`, returns the TXT record to set), `RemoveDomainRequest → Domain` (owner; `mark_deleted`), `ListDomainsRequest → DomainList`, `VerifyDomainRequest → Domain` (owner; `nudge`).
- `AuthorizeSiteAccessRequest → SiteAccess` — `{site_id, identity_id, min_role}` → `{allowed, role}`; answered from `site_member` only. Sent by `content`, `forms`.
- `ResolvePublicSiteRequest → PublicSite` — exactly one of `{host, slug, site_id}` → `{site_id, slug, name, canonical_base_url, layout_html, css, nav[]}`: the *effective* theme values and the canonical URL (verified domain if any, else the platform URL) — derived state only (§7). Unauthenticated. Sent by `delivery`, `forms`.

## API

`SiteService`: `CreateSite`, `UpdateSite`, `GetSite`, `ListSites`, `AddSiteMember`, `RemoveSiteMember`, `ListSiteMembers`, `SaveNavigation`, `AddDomain`, `RemoveDomain`, `ListDomains`, `VerifyDomain`.

## Directive sections that bind

§2 for every table; §3, §4, §5 for `domain`; §6 (reads are not adapters) for the Kratos and DNS lookups; §7 for `AuthorizeSiteAccess` and `ResolvePublicSite`; §10 for `CLAUDE.md` / `flows.md`.

## Tests

Unit: slug and hostname validation; menu ordering; nav path validation; the `domain` pass decision table (deleting, resolver error, match, blip under/at 3 misses, pending, 7-day give-up). Integration (testkit): create site makes creator owner; duplicate slug rejected; editor cannot change site/members/nav/domains; last owner cannot be removed (two concurrent removals leave one owner); nav replace atomic; `ResolvePublicSite` by slug / by verified host / unverified host → NotFound; duplicate hostname across sites rejected; remove domain deletes it; nudge re-arms a blocked domain. Kratos admin and DNS are stubbed by traits.
