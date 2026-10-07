# 05 — Deployment

## Free tier sizing (2000m CPU / 4096Mi / 10Gi per environment)

The organisation has no payment method, so every environment stays at or below the free maximum.

| Component | Replicas | CPU req | Memory req | Storage |
|---|---|---|---|---|
| app (both nanoservices) | 1 | 250m | 256Mi | — |
| frontend (nginx-unprivileged) | 1 | 50m | 64Mi | — |
| Kratos public | 1 | 100m | 128Mi | — |
| Kratos admin | 1 | 100m | 128Mi | — |
| CNPG operator | 1 | 100m | 256Mi | — |
| Postgres (CNPG, `instances: 1`) | 1 | 500m | 1Gi | 5Gi |
| migration Jobs (transient) | — | 100m | 128Mi | — |
| **Total** | | ~1200m | ~2Gi | 5Gi |

**Deviation:** app `replicas: 1` instead of the standard 2 — the free tier allows one replica per Deployment. The frameworks are multi-replica safe, so raising it after adding a payment method is a config change only.

`.basable/config.yaml`: both `preview` and `live` at the free maximum (rendered by the scaffold).

## Database

One CNPG cluster, application database with schemas `site` and `content` (one role each), separate `kratos` database. dbmate-format migrations run by a Job before the app rolls.

## Secrets

CNPG-minted: app DB credentials, Kratos DB credentials. User-supplied later (via `app-secrets`, per environment, in the editor): object-storage credentials if a media library is added; SMTP credentials for Kratos login codes / contact forms.

## CI

GitHub (`basable/rust-me`), workflow under `.github/workflows/`, images to `registry.basable.com/abandon-above-tooth/rust-me`; lint → Bazel build/push → render `k8s/` → upload `rendered-manifests`. `paths-ignore: docs/**`. `dev` → `*.preview.basable.com`, `main` → `*.live.basable.com`.
