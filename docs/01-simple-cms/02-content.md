# 02 — Nanoservice: content

Owns pages and posts, their revisions, the live pointer, and one lifecycle: the scheduled `publication`.

## Tables (Directive §2)

### `page`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| site_id | uuid | `site`'s id, received over the messenger, never joined |
| kind | string | `page` \| `post`, set at creation, not changeable |
| slug | string | unique index `(site_id, slug)`; `home` is the site root; `blog`, `sitemap.xml`, `robots.txt` and slugs starting with `_` are reserved |
| created_at, updated_at | timestamp | |

### `page_revision`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| page_id | uuid | FK → page.id |
| title | string | |
| body | json | ordered blocks `{kind: heading \| paragraph \| image \| link \| contact_form, ...}`; images by external `https` URL |
| meta_description | string, nullable | SEO; falls back to the first paragraph, truncated |
| social_image_url | string, nullable | `og:image` |
| post_date | timestamp, nullable | required for `post`, ignored for `page` |
| tags | json | array of lowercase slugs, posts only, max 10 |
| author_id | uuid | Kratos identity |
| created_at | timestamp | |

Append-only: a save inserts a revision, never updates one. The latest revision is the draft. SEO fields, post date and tags are revisioned, so what is live is always one consistent snapshot.

### `published_page`
| Column | Type | Notes |
|---|---|---|
| page_id | uuid PK | FK → page.id |
| revision_id | uuid, nullable | the live revision; **null = unpublished** (row kept so `set_at` survives) |
| set_at | timestamp | when the pointer was last set |
| set_by_publication_id | uuid, nullable | the publication that set it, if any |
| first_published_at | timestamp, nullable | set once; feeds the sitemap `lastmod` fallback |

Invariant: `revision_id` belongs to `page_id` (checked in the writing transaction). `set_at` lets a manual action made *after* a schedule was (re)set win over that schedule.

### Queries

Create page (+ first revision); insert revision; get page with draft + live revision; list pages per site (filter by kind) with live state; list revisions; set live pointer (manual: unconditional; scheduled: guarded by `set_at`); get published by `(site_id, slug)`; **published index** by site: live posts ordered by `post_date desc` with optional tag filter (`tags ? $tag`) and keyset pagination, or all live slugs with `updated_at` for the sitemap; get any revision of a page for preview.

## Processing-object type `publication` (Directive §3, §5)

One scheduled change of one page's live pointer.

**Spec** (`publication_spec`)
| Field | Type | Mutable |
|---|---|---|
| page_id | uuid | immutable |
| action | string `publish` \| `unpublish` | immutable |
| revision_id | uuid, nullable | immutable; required for `publish`, pinned at schedule time (what the editor previewed is what goes live) |
| publish_at | timestamp | **mutable** — reschedule = `update_spec`, advancing the generation (§3.2) |

**Status** (`publication_status`)
| Field | Type |
|---|---|
| phase | string `scheduled` \| `applied` \| `superseded` \| `invalid` |
| applied_at | timestamp, nullable |

**Reconcile pass** (level-triggered, on the claim-time snapshot):
1. `meta.deleting()` (cancelled) → nothing external exists; `Delete`.
2. `status.phase` ∈ {`applied`, `superseded`} → `Settled { status: None }`.
3. `now < spec.publish_at` → `Converged { phase: scheduled }.after(publish_at - now)`.
4. Re-check: page exists and (for `publish`) the revision belongs to it; otherwise `Blocked { phase: invalid }`.
5. Apply in one short transaction on content's own tables (no remote I/O, §3.7): upsert `published_page ... WHERE set_at <= meta.generation_changed_at`. One row → `Settled { phase: applied, applied_at: now }`; zero rows (an editor published/unpublished by hand after this schedule was set) → `Settled { phase: superseded }`; a row already showing `set_by_publication_id = this` (replay after an ambiguous completion) → `applied`; database error → `Retry`.

**Known window**: a reschedule landing while an attempt is between steps 3 and 5 may still apply at the old time; bounded by `attempt_timeout` (30 s), so the handler refuses a reschedule whose old `publish_at` is under 60 s away, and refuses `applied`/`superseded` publications (the `update_spec` closure sees status read-only, §8).

**Teardown**: `CancelPublication` → `mark_deleted`; pass returns `Delete` (§3.6, nothing external to confirm). An applied publication cannot be cancelled — undo by publishing/unpublishing by hand.

**Worker policy** (identical on every replica, §4.4): `resync` 1 h, `backoff` 5 s → 5 min, `max_attempts` 10, `attempt_timeout` 30 s.

## Request flow

Every editor request: (1) send `AuthorizeSiteAccessRequest` to `site` (min role `editor`) — remote I/O, outside any transaction (§3.7); (2) on `allowed`, one transaction on content's own tables, or a `TypedStore` call (`create`, `update_spec`, `mark_deleted`) for publications — never `repository.rs` SQL for intent (§4.2). `PublishPage` / `UnpublishPage` set `published_page` unconditionally with `set_at = now()`.

## Messages

Handles (from api): `CreatePageRequest`, `SavePageRequest`, `GetPageRequest` → `Page`; `ListPagesRequest → PageList`; `ListRevisionsRequest → RevisionList`; `PublishPageRequest`, `UnpublishPageRequest` → `Page`; `SchedulePublicationRequest`, `ReschedulePublicationRequest`, `CancelPublicationRequest` → `Publication`; `ListPublicationsRequest → PublicationList`.

Handles (from delivery):
- `GetPublishedPageRequest → PublishedPage` — `{site_id, slug}` → `{kind, title, body, meta_description, social_image_url, post_date, tags, updated_at}`; NotFound if absent or unpublished. Public.
- `ListPublishedPagesRequest → PublishedPageIndex` — `{site_id, mode: posts \| all, tag?, cursor?, limit ≤ 20}` → entries `{slug, kind, title, meta_description, post_date, tags, updated_at}` + next cursor. Public.
- `GetPreviewPageRequest → PublishedPage` — `{page_id, revision_id?, identity_id}`: sends `AuthorizeSiteAccessRequest` first, then returns that revision (default: the latest draft).

Sends: `AuthorizeSiteAccessRequest → SiteAccess` (§7).

## API

`ContentService`: `CreatePage`, `SavePage`, `GetPage`, `ListPages`, `ListRevisions`, `PublishPage`, `UnpublishPage`, `SchedulePublication`, `ReschedulePublication`, `CancelPublication`, `ListPublications`.

## Directive sections that bind

§2 for every table; §3, §4, §5 for `publication`; §7 for `AuthorizeSiteAccess` and the three delivery reads (read-only requests carry no derived intent, so no generation); §10 docs.

## Tests

Unit: block validation (incl. `https`-only URLs, reserved slugs, post requires `post_date`, tag limits); pass decision table. Integration (testkit): save appends a revision; publish points at it and a later save leaves the live version unchanged; rollback by publishing an old revision; unauthorized save and preview rejected (site stubbed in the test messenger); slug unique per site only; posts index ordered by date, filtered by tag, paginated with a stable cursor; unpublished posts never listed; a due publication applies once with two workers running; reschedule advances the generation and moves the wake; cancel deletes; manual publish after scheduling makes the publication `superseded`; a revision of another page → `Blocked`.
