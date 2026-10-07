# 02 — Nanoservice: content

Owns pages and posts, the page tree, their revisions, the live pointer, and one lifecycle: the scheduled `publication`.

## Tables (Directive §2)

### `page`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| site_id | uuid | `site`'s id, received over the messenger, never joined |
| kind | string | `page` \| `post`, set at creation, not changeable |
| parent_id | uuid, nullable | FK → page.id (same table); null = top level; **always null for posts** |
| slug | string | one path segment, `^[a-z0-9-]{1,63}$` |
| path | string | materialised full path, e.g. `about/team`; **unique index `(site_id, path)`**; `home` is the site root |
| depth | int32 | 0 for top level; max 4 (five levels) |
| created_at, updated_at | timestamp | |

Invariants:
- `path = parent.path + '/' + slug` (or `slug` at top level), maintained only by `CreatePage` and `MovePage`.
- Parent is a `page` of the same site, never a post, never the page itself or one of its descendants (no cycles).
- Reserved first segments: `blog`, `sitemap.xml`, `robots.txt`, anything starting with `_`. `home` may not have children (it is `/`).
- No hard delete of pages (not in scope); unpublish hides one.

### `page_revision`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| page_id | uuid | FK → page.id |
| title | string | |
| body | json | ordered blocks `{kind: heading \| paragraph \| image \| link \| contact_form, ...}`; images by external `https` URL; internal links by page path |
| meta_description | string, nullable | SEO; falls back to the first paragraph, truncated |
| social_image_url | string, nullable | `og:image` |
| post_date | timestamp, nullable | required for `post`, ignored for `page` |
| tags | json | array of lowercase slugs, posts only, max 10 |
| author_id | uuid | Kratos identity |
| created_at | timestamp | |

Append-only: a save inserts a revision, never updates one. The latest revision is the draft. Slug and parent are *not* revisioned — they are the page's address, changed only by `MovePage` and effective immediately.

### `published_page`
| Column | Type | Notes |
|---|---|---|
| page_id | uuid PK | FK → page.id |
| revision_id | uuid, nullable | the live revision; **null = unpublished** (row kept so `set_at` survives) |
| set_at | timestamp | when the pointer was last set |
| set_by_publication_id | uuid, nullable | the publication that set it, if any |
| first_published_at | timestamp, nullable | set once |

Invariant: `revision_id` belongs to `page_id` (checked in the writing transaction). `set_at` lets a manual action made *after* a schedule was (re)set win over that schedule.

A published child under an unpublished parent stays reachable at its path; breadcrumbs simply show the parent's title unlinked. Visibility is per page, never inherited, so publishing one page never changes another.

### Moving a page — `MovePageRequest → Page`

`{page_id, new_parent_id?, new_slug}`, one transaction on content's own tables:
1. Lock the moved page and every descendant `FOR UPDATE` (`WHERE site_id = $1 AND (id = $2 OR path LIKE $old_path || '/%')`, ordered by id).
2. Validate: new parent exists in the site, is a `page`, is not in the locked subtree; resulting depth of the deepest descendant ≤ 4; reserved segments.
3. Rewrite `path` and `depth` for the page and each descendant (`$new_path || substr(path, len($old_path)+1)`).
4. The unique index on `(site_id, path)` rejects a collision → roll back, `AlreadyExists` with the colliding path.

No redirects (excluded): old URLs 404 after the move. The admin UI warns when the subtree contains published pages and lists the URLs that will change.

### Queries

Create page (+ first revision) with parent path lookup; insert revision; get page with draft + live revision; list pages per site as a tree (ordered by `path`) with live state, or posts only; list revisions; move subtree (above); set live pointer (manual: unconditional; scheduled: guarded by `set_at`); get published by `(site_id, path)` plus its ancestors' titles for breadcrumbs (`path` prefixes, one query); **published index**: live posts ordered by `post_date desc` with optional tag filter (`tags ? $tag`) and keyset pagination, or all live paths with `updated_at` for the sitemap; get any revision of a page for preview.

## Processing-object type `publication` (Directive §3, §5)

One scheduled change of one page's live pointer. Keyed on `page_id`, so a move does not affect it.

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

Handles (from api): `CreatePageRequest` (with optional `parent_id`), `SavePageRequest`, `MovePageRequest`, `GetPageRequest` → `Page`; `ListPagesRequest → PageList`; `ListRevisionsRequest → RevisionList`; `PublishPageRequest`, `UnpublishPageRequest` → `Page`; `SchedulePublicationRequest`, `ReschedulePublicationRequest`, `CancelPublicationRequest` → `Publication`; `ListPublicationsRequest → PublicationList`.

Handles (from delivery):
- `GetPublishedPageRequest → PublishedPage` — `{site_id, path}` → `{kind, path, title, body, meta_description, social_image_url, post_date, tags, updated_at, breadcrumbs[{path, title, published}]}`; NotFound if absent or unpublished. Public.
- `ListPublishedPagesRequest → PublishedPageIndex` — `{site_id, mode: posts \| all, tag?, cursor?, limit ≤ 20}` → entries `{path, kind, title, meta_description, post_date, tags, updated_at}` + next cursor. Public.
- `GetPreviewPageRequest → PublishedPage` — `{page_id, revision_id?, identity_id}`: sends `AuthorizeSiteAccessRequest` first, then returns that revision (default: the latest draft).

Sends: `AuthorizeSiteAccessRequest → SiteAccess` (§7).

## API

`ContentService`: `CreatePage`, `SavePage`, `MovePage`, `GetPage`, `ListPages`, `ListRevisions`, `PublishPage`, `UnpublishPage`, `SchedulePublication`, `ReschedulePublication`, `CancelPublication`, `ListPublications`.

## Directive sections that bind

§2 for every table; §3, §4, §5 for `publication`; §7 for `AuthorizeSiteAccess` and the three delivery reads (read-only requests carry no derived intent, so no generation); §10 docs.

## Tests

Unit: block validation (`https`-only URLs, reserved segments, post requires `post_date`, tag limits); path computation and move validation (cycle, depth, posts cannot nest, `home` has no children); pass decision table. Integration (testkit): save appends a revision; publish points at it and a later save leaves the live version unchanged; rollback by publishing an old revision; unauthorized save and preview rejected (site stubbed in the test messenger); path unique per site only; create nested page computes its path; move rewrites a three-level subtree atomically; move into own descendant rejected; move colliding with an existing path rolls back everything; a published page is served at its new path after a move and 404s at the old one; posts index ordered, filtered and paginated; a due publication applies once with two workers running; reschedule advances the generation; cancel deletes; manual publish after scheduling makes it `superseded`; a revision of another page → `Blocked`.
