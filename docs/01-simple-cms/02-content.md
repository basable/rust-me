# 02 \u2014 Nanoservice: content

Owns pages, revisions, the live pointer, and one lifecycle: the scheduled `publication`.

## Tables (Directive \u00a72)

### `page`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| site_id | uuid | `site`'s id, received over the messenger, never joined |
| slug | string | unique index `(site_id, slug)`; `home` is the site root |
| title | string | |
| created_at, updated_at | timestamp | |

### `page_revision`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| page_id | uuid | FK \u2192 page.id |
| title | string | title as of this revision |
| body | json | ordered blocks `{kind: heading\\|paragraph\\|image\\|link, ...}`; images by external URL |
| author_id | uuid | Kratos identity |
| created_at | timestamp | |

Append-only: a save inserts a revision, never updates one. The latest revision is the draft.

### `published_page`
| Column | Type | Notes |
|---|---|---|
| page_id | uuid PK | FK \u2192 page.id |
| revision_id | uuid, nullable | the live revision; **null = unpublished** (row kept so `set_at` survives) |
| set_at | timestamp | when the pointer was last set (by a person or a publication) |
| set_by_publication_id | uuid, nullable | the publication that set it, if any |

Invariant: `revision_id` belongs to `page_id` (checked in the writing transaction). `set_at` is the guard that lets a manual action made *after* a schedule was (re)set win over that schedule (below).

### Queries

Create page (+ first revision); insert revision; get page with draft + live revision; list pages per site with live state; list revisions; set live pointer (manual: unconditional; scheduled: guarded); get published page by `(site_id, slug)` joining `published_page` \u2192 `page_revision` (same schema).

## Processing-object type `publication` (Directive \u00a73, \u00a75)

One scheduled change of one page's live pointer.

**Spec** (`publication_spec`)
| Field | Type | Mutable |
|---|---|---|
| page_id | uuid | immutable |
| action | string `publish` \\| `unpublish` | immutable |
| revision_id | uuid, nullable | immutable; required for `publish`, pinned at schedule time (what the editor previewed is what goes live) |
| publish_at | timestamp | **mutable** \u2014 reschedule = `update_spec`, which advances the generation (\u00a73.2) |

**Status** (`publication_status`)
| Field | Type |
|---|---|
| phase | string `scheduled` \\| `applied` \\| `superseded` \\| `invalid` |
| applied_at | timestamp, nullable |

**Reconcile pass** (level-triggered, on the claim-time snapshot):
1. `meta.deleting()` (cancelled) \u2192 nothing external to confirm absent; return `Delete`.
2. `status.phase` \u2208 {`applied`, `superseded`} for this generation \u2192 `Settled { status: None }`.
3. `now < spec.publish_at` \u2192 `Converged { phase: scheduled }.after(publish_at - now)` (the known wait).
4. Re-check: page exists and (for `publish`) the revision belongs to it; otherwise \u2192 `Blocked { phase: invalid }` (unsatisfiable for this generation).
5. Apply in one short transaction on content's own tables (no remote I/O, \u00a73.7): `UPDATE/INSERT published_page ... WHERE set_at <= meta.generation_changed_at`. One row affected \u2192 `Settled { phase: applied, applied_at: now }`. Zero rows (an editor published or unpublished by hand after this schedule was set) \u2192 `Settled { phase: superseded }`. Database error \u2192 `Retry`.

**Idempotence of step 5**: re-applying sets the same pointer; a replay after an ambiguous completion finds `set_by_publication_id = this` and treats it as applied.

**Known window**: a reschedule landing while an attempt is between step 3 and step 5 may still apply at the old time; bounded by `attempt_timeout` (30 s), so the handler refuses a reschedule whose old `publish_at` is under 60 s away. `ReschedulePublication` also refuses `applied`/`superseded` publications (the `update_spec` closure sees status read-only, \u00a78).

**Teardown**: `CancelPublication` \u2192 `mark_deleted`; the pass returns `Delete`. Nothing external exists, so absence is trivially confirmed (\u00a73.6). An already-applied publication cannot be cancelled (undo = publish/unpublish by hand).

**Worker policy** (identical on every replica, \u00a74.4): `resync` 1 h, `backoff` 5 s \u2192 5 min, `max_attempts` 10, `attempt_timeout` 30 s.

## Request flow

Every editor request: (1) send `AuthorizeSiteAccessRequest` to `site` \u2014 remote I/O, outside any transaction (\u00a73.7); (2) on `allowed`, one transaction on content's own tables, or a `TypedStore` call (`create`, `update_spec`, `mark_deleted`) for publications \u2014 never `repository.rs` SQL for intent (\u00a74.2). `PublishPage` / `UnpublishPage` set `published_page` unconditionally with `set_at = now()`.

## Messages

Handles: `CreatePageRequest`, `SavePageRequest`, `GetPageRequest` \u2192 `Page`; `ListPagesRequest \u2192 PageList`; `ListRevisionsRequest \u2192 RevisionList`; `PublishPageRequest`, `UnpublishPageRequest` \u2192 `Page`; `SchedulePublicationRequest`, `ReschedulePublicationRequest`, `CancelPublicationRequest` \u2192 `Publication`; `ListPublicationsRequest \u2192 PublicationList`; `GetPublishedPageRequest \u2192 PublishedPage` (public, from `delivery`; `{site_id, slug}` \u2192 `{title, body, published_at}` or NotFound).
Sends: `AuthorizeSiteAccessRequest \u2192 SiteAccess` (\u00a77).

## API

`ContentService`: `CreatePage`, `SavePage`, `GetPage`, `ListPages`, `ListRevisions`, `PublishPage`, `UnpublishPage`, `SchedulePublication`, `ReschedulePublication`, `CancelPublication`, `ListPublications`.

## Directive sections that bind

\u00a72 for every table; \u00a73, \u00a74, \u00a75 for `publication`; \u00a77 for `AuthorizeSiteAccess` and `GetPublishedPage`; \u00a710 docs.

## Tests

Unit: block body validation; pass decision table (deleting, settled, not-yet-due, invalid, apply, superseded). Integration (testkit): save appends a revision; publish points at it and a later save leaves the live version unchanged; rollback by publishing an old revision; unauthorized save rejected (site stubbed in the test messenger); slug unique per site only; a due publication applies once with two workers running; reschedule advances the generation and moves the wake; cancel deletes; manual publish after scheduling makes the publication `superseded`; revision of another page \u2192 `Blocked`.
