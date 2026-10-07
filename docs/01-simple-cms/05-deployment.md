# 05 \u2014 Deployment

## Free tier sizing (2000m CPU / 4096Mi / 10Gi per environment)

The organisation has no payment method, so every environment stays at or below the free maximum.

| Component | Replicas | CPU req | Memory req | Storage |
|---|---|---|---|---|
| app (site, content, delivery) | 1 | 250m | 256Mi | \u2014 |
| frontend (nginx-unprivileged) | 1 | 50m | 64Mi | \u2014 |
| Kratos public | 1 | 100m | 128Mi | \u2014 |
| Kratos admin | 1 | 100m | 128Mi | \u2014 |
| CNPG operator | 1 | 100m | 256Mi | \u2014 |
| Postgres (CNPG, `instances: 1`) | 1 | 500m | 1Gi | 5Gi |
| migration Jobs (transient) | \u2014 | 100m | 128Mi | \u2014 |
| **Total** | | ~1200m | ~2Gi | 5Gi |

`.basable/config.yaml`: both `preview` and `live` at the free maximum (rendered by the scaffold).

## Deviations from the standard shape

1. **App `replicas: 1`** instead of 2 \u2014 the free tier allows one replica per Deployment. The frameworks are multi-replica safe (the `publication` worker is tested with two workers), so raising it after adding a payment method is a config change.
2. **HTTPRoute gains `/s/*` \u2192 app**, for the public website rendered by `delivery`. Reason: visitors need plain HTML URLs, not `/api/*`. The implementing session adds this match to the rendered HTTPRoute.

## Database

One CNPG cluster: application database with schemas `site` and `content` (one role each; `delivery` owns nothing and has none), a separate `kratos` database. dbmate-format migrations run by a Job before the app rolls.

## Secrets

CNPG-minted: app DB credentials, Kratos DB credentials. The app also needs the in-cluster Kratos **admin** URL (no secret; admin is cluster-internal only) for `AddSiteMember`. User-supplied later (via `app-secrets`, per environment): SMTP credentials if Kratos login codes are sent through a real mail provider, or if contact-form email is added.

## CI

GitHub (`basable/rust-me`), workflow under `.github/workflows/`, images to `registry.basable.com/abandon-above-tooth/rust-me`; lint \u2192 Bazel build/push \u2192 render `k8s/` \u2192 upload `rendered-manifests`; `paths-ignore: docs/**`. `dev` \u2192 `*.preview.basable.com`, `main` \u2192 `*.live.basable.com`.
