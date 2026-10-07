# 03 \u2014 Testing

- **Unit tests** per nanoservice: slug and block validation, menu ordering, the `publication` pass decision table, HTML escaping and template substitution in `delivery`.
- **effecttest audits**: none \u2014 no nanoservice owns an external effect (the Kratos admin lookup in `site` is a read, not an adapter, Directive \u00a76). Adding contact-form email or uploads later adds an adapter and its audit.
- **Integration** on `basable-testkit` (testcontainers Postgres, one database per test): repositories and handlers against real migrations; cross-nanoservice flows through the generated messenger with all three nanoservices registered; the `publication` worker run with two workers on one database to prove a due publication applies exactly once, plus reschedule, cancel and supersede cases.
- **Boot check**: the app refuses to start unless the migration ledger holds every embedded version.
- **CI gate** (before any deploy): lint (rustfmt, clippy incl. `await_holding_lock`, buf lint), Bazel build, all tests, then image push and `k8s/` render. A red gate deploys nothing.
