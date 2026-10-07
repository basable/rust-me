# 03 — Testing

- **Unit tests** per nanoservice: slug, hostname and block validation; menu ordering; the `publication` and `domain` pass decision tables; form validation, honeypot and rate-limit windows; HTML escaping, meta tags, sitemap and robots generation in `delivery`.
- **effecttest audits**: none today — no nanoservice owns an external effect. The Kratos admin lookup and DNS TXT lookups in `site` are reads, not adapters (Directive §6); they sit behind traits stubbed in tests. The `later` email alert in `forms` adds one `keyed_replay` adapter and its mandatory ack-loss audit in `nanoservices/forms/tests/effects_audit.rs`.
- **Integration** on `basable-testkit` (testcontainers Postgres, one database per test): repositories and handlers against real migrations; cross-nanoservice flows through the generated messenger with all four nanoservices registered; each worker (`publication`, `domain`) run as two workers on one database to prove single application, plus reschedule / cancel / supersede and blip / give-up / nudge cases; the `purge_rate_limits` ticker run once against seeded rows.
- **Boot check**: the app refuses to start unless the migration ledger holds every embedded version.
- **CI gate** (before any deploy): lint (rustfmt, clippy incl. `await_holding_lock`, buf lint), Bazel build, all tests, then image push and `k8s/` render. A red gate deploys nothing.
