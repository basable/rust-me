# 05 — Deployment

## Free tier sizing (2000m CPU / 4096Mi / 10Gi per environment)

The organisation has no payment method, so every environment stays at or below the free maximum.

| Component | Replicas | CPU req | Memory req | Storage |
|---|---|---|---|---|
| app (site, content, forms, delivery) | 1 | 250m | 256Mi | — |
| frontend (nginx-unprivileged) | 1 | 50m | 64Mi | — |
| Kratos public | 1 | 100m | 128Mi | — |
| Kratos admin | 1 | 100m | 128Mi | — |
| CNPG operator | 1 | 100m | 256Mi | — |
| Postgres (CNPG, `instances: 1`) | 1 | 500m | 1Gi | 5Gi |
| migration Jobs (transient) | — | 100m | 128Mi | — |
| **Total** | | ~1200m | ~2Gi | 5Gi |

`.basable/config.yaml`: both `preview` and `live` at the free maximum (rendered by the scaffold).

## Deviations from the standard shape

1. **App `replicas: 1`** instead of 2 — the free tier allows one replica per Deployment. The frameworks are multi-replica safe (both workers are tested with two workers), so raising it after adding a payment method is a config change.
2. **HTTPRoute gains `/s/*` → app** on the platform host, for the public websites and contact-form POSTs. Reason: visitors need plain HTML URLs, not `/api/*`. The implementing session adds this match to the rendered HTTPRoute.
3. **Custom domains need the platform to route and certify each hostname.** The app side is fully planned (the `domain` type verifies ownership; `delivery` and `forms` serve by `Host`). What the plan cannot assume: that this platform lets a tenant attach arbitrary hostnames to its HTTPRoute and issues TLS certificates for them. The implementing session must confirm this; the intended mechanism is a second HTTPRoute whose `hostnames` lists the verified domains with every path → app, and the owner pointing a CNAME at the environment's host. If the platform does not support it, custom domains degrade to "verified, not yet served" and sites stay at `/s/{slug}/`. An app writing its own HTTPRoute would be a Kubernetes write — an external effect (§6) needing in-cluster RBAC — and is **not** planned.

## Database

One CNPG cluster: application database with schemas `site`, `content` and `forms` (one role each; `delivery` owns nothing and has none), a separate `kratos` database. dbmate-format migrations run by a Job before the app rolls.

## Network

The app needs egress to DNS (TXT lookups for domain verification) and to the in-cluster Kratos admin service.

## Secrets

CNPG-minted: app DB credentials, Kratos DB credentials. App-generated: the contact-form IP-hash salt lives in the `app-secrets` key `FORM_IP_SALT` so every replica hashes alike; the implementing session decides whether CI generates it or the user sets it once per environment. User-supplied later (via `app-secrets`, per environment): email-provider credentials for Kratos login codes through a real provider and for the `later` submission alert.

## CI

GitHub (`basable/rust-me`), workflow under `.github/workflows/`, images to `registry.basable.com/abandon-above-tooth/rust-me`; lint → Bazel build/push → render `k8s/` → upload `rendered-manifests`; `paths-ignore: docs/**`. `dev` → `*.preview.basable.com`, `main` → `*.live.basable.com`.
