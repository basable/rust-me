# 02 — Nanoservice: content

Plain executor today. Grows a `publication` processing-object type only if scheduled publishing is included (round 1).

## Tables (Directive §2)

### `page`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| site_id | uuid | `site`'s id, received over the messenger, never joined |
| slug | string | unique per site: unique index `(site_id, slug)` |
| title | string | |
| created_at | timestamp | |
| updated_at | timestamp | |

### `page_revision`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| page_id | uuid | |
| body | json | ordered list of blocks `{kind: heading\|paragraph\|image\|link, ...}` |
| author_id | uuid | Kratos identity |
| created_at | timestamp | |

Rows are append-only: a save inserts a revision, never updates one.

### `published_page`
| Column | Type | Notes |
|---|---|---|
| page_id | uuid PK | one live revision per page |
| revision_id | uuid | must belong to `page_id` |
| published_at | timestamp | |

Publish = upsert this row in one transaction. Unpublish = delete it.

## Request flow

`SavePage`/`PublishPage`: (1) send `AuthorizeSiteAccessRequest` to `site` — remote I/O, outside any transaction (Directive §3.7); (2) on `allowed`, one transaction on `content`'s own tables.

## Messages

Handles `SavePageRequest → Page`, `GetPageRequest → Page`, `ListPagesRequest → PageList`, `PublishPageRequest → Page`. Sends `AuthorizeSiteAccessRequest → SiteAccess` (Directive §7).

## API

`ContentService`: `SavePage`, `GetPage`, `ListPages`, `PublishPage`.

## Tests

Unit: block body validation. Integration: save creates a revision; publish points at it; a later save does not change the published version; unauthorized save rejected (site stubbed in the test messenger); duplicate slug per site rejected, same slug on two sites allowed.
