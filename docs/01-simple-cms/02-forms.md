# 02 \u2014 Nanoservice: forms

Owns what visitors send through contact forms. Plain executor: no processing-object type (a submission is stored in one transaction; nothing waits or converges), two plain tables, one schedule, no external effects yet.

## The form

An editor adds a `contact_form` block to a page (`content` stores it like any block). `delivery` renders it as a plain HTML `<form method=post action="/_contact">` (or `/s/{slug}/_contact` on the platform host) with fields `name`, `email`, `message`, the hidden `page_path`, and a hidden honeypot field `website`. No JavaScript is required.

## Public route

A raw axum route on the api crate: `POST /_contact` on a custom domain and `POST /s/{site_slug}/_contact` on the platform host, unauthenticated, form-encoded, body \u2264 16 KiB. It computes `ip_hash = sha256(client_ip \u2016 salt)` (the raw IP is never stored), makes ONE messenger send `SubmitContactFormRequest{host or site_slug, page_path, name, email, message, honeypot, ip_hash}`, then answers `303 See Other` back to the page with `?sent=1` (or `?error=\u2026`).

## Tables (Directive \u00a72)

### `form_submission`
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| site_id | uuid | from `ResolvePublicSite`, never joined |
| page_path | string | where it was sent from (informational; not validated against `content`) |
| name | string | \u2264 200 chars |
| email | string | syntactically valid, \u2264 320 chars |
| message | string | \u2264 5000 chars |
| ip_hash | string | for abuse review only |
| read_at | timestamp, nullable | |
| created_at | timestamp | |

Index `(site_id, created_at)` for the inbox.

### `form_rate_limit`
| Column | Type | Notes |
|---|---|---|
| key | string PK | `<site_id>:<ip_hash>:<hour window start>` |
| count | int32 | |
| window_start | timestamp | |

Accept rule, in the same transaction as the insert: `INSERT \u2026 ON CONFLICT (key) DO UPDATE SET count = count + 1 RETURNING count`; over **5 per hour per IP per site** \u2192 roll back, answer `rate_limited`. No advisory locks (\u00a77).

## Handler flow \u2014 `SubmitContactFormRequest \u2192 FormSubmissionReceipt`

1. Honeypot filled \u2192 answer `accepted` and store nothing (bots learn nothing).
2. Validate lengths and email syntax \u2192 `invalid` with the field.
3. Send `ResolvePublicSiteRequest{host | slug}` to `site` \u2014 remote I/O, outside any transaction (\u00a73.7). NotFound \u2192 `invalid`.
4. One transaction: rate-limit upsert + insert submission \u2192 `accepted`.

## Editor inbox

`ListFormSubmissionsRequest \u2192 FormSubmissionList` (site, unread filter, keyset paginated), `MarkFormSubmissionReadRequest \u2192 FormSubmission`, `DeleteFormSubmissionRequest \u2192 FormSubmission`. Each first sends `AuthorizeSiteAccessRequest` (min role `editor`) to `site`.

## Schedule

`purge_rate_limits`, every 1 h: a ticker worker (registered with `basable-app`, joined on shutdown) that deletes `form_rate_limit` rows with `window_start < now() - 2h`. Idempotent and safe on every replica at once (a plain `DELETE \u2026 WHERE`).

## Later: email alert

When the user supplies an email provider credential, `forms` gains one external call `send_submission_alert` with strategy `keyed_replay` (irreversible: an email), key = submission id, `replayWindow` from the provider's docs. Because the alert must survive crashes and retry, it would also justify a small lifecycle (an `alert` processing-object type per submission) \u2014 decided when the integration is added.

## API

`FormsService`: `ListFormSubmissions`, `MarkFormSubmissionRead`, `DeleteFormSubmission`. (Submission enters through the raw route only.)

## Directive sections that bind

\u00a72 for both tables; \u00a77 for `ResolvePublicSite` and `AuthorizeSiteAccess`; \u00a73.7 for sends outside transactions; \u00a71 for the ticker; \u00a710 docs.

## Tests

Unit: validation, honeypot, rate-limit key windows. Integration (testkit): accepted submission stored and listed; 6th submission in an hour from one IP rejected, another IP accepted; unknown site rejected; non-member cannot list; purge removes only expired windows; mark-read and delete.
