# 03 — Testing

- **Unit tests** per nanoservice: validation (slugs, block bodies), menu ordering.
- **effecttest audits**: none yet — no external calls. Media upload (object storage) would add one adapter and its audit.
- **Integration** on `basable-testkit` (testcontainers Postgres, one database per test): each nanoservice's repository and handlers against real migrations; cross-nanoservice flows through the generated messenger with both nanoservices registered.
- **Boot check**: the app refuses to start unless the migration ledger holds every embedded version.
- **CI gate** (before any deploy): lint (clippy, buf lint), Bazel build, all tests, then image push and `k8s/` render. A red gate deploys nothing.
