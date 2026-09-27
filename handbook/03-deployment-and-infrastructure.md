# Deployment & Infrastructure

Where everything actually runs today, as of 2026-09-06. No credential values,
API keys, or secrets appear anywhere in this document — see
`04-secrets-and-credentials.md` for how those are managed.

## Environments

There are three environments, deliberately kept separate:

- **Local** — Docker Compose (`infra/docker/docker-compose.yml`) running
  Postgres, Redis, MinIO (local S3-compatible storage stand-in), and the
  backend/frontend directly. No real third-party services are touched.
- **Dev** (`dev.remoteaiplatform.com`) — a fully separate live deployment with
  its own Supabase project, its own Cloudflare Worker, and its own Render
  service, used for testing changes against real infrastructure before they
  reach production. Payments on dev are **always** the sandbox provider —
  dev is deliberately never wired to real Stripe.
- **Prod** (`remoteaiplatform.com`) — the real product. Payments are switched off
  (`MARKETPLACE_PAYMENTS_ENABLED=false`); Stripe keys exist but nothing is charged.

## The branch model

Promotion path: **feature branch → `dev` → `prod`**, enforced by CI, not just by
convention. A dedicated workflow (`enforce-branch-flow.yml`) rejects any pull
request opened directly against `prod` unless its source branch is `dev` — this
runs on every PR targeting `prod` and needs no repository secrets to do it (it
only inspects the PR's branch name). Both `dev` and `prod` have branch protection, but `enforce_admins` is
currently OFF (turned off during the Figma migration; restore after launch — see
`go-live/progress.md`); required checks still gate PR merges
with the required status checks green.

## Frontend hosting — Cloudflare Workers

The Next.js app is built for Cloudflare's edge runtime using the OpenNext
adapter (`@opennextjs/cloudflare`) and deployed as a Cloudflare Worker via
`wrangler deploy`. Two Workers exist, one per environment, each bound to its own
custom domain and also reachable at its own `*.workers.dev` URL:

- Production Worker → `remoteaiplatform.com` / `www.remoteaiplatform.com`
- Dev Worker → `dev.remoteaiplatform.com`

Deploys are triggered by GitHub Actions: `deploy-frontend.yml` (prod) and
`deploy-frontend-dev.yml` (dev) each listen for the `CI` workflow completing
successfully on the matching branch (`workflow_run`, not a raw push trigger),
then check out that exact commit, build, and deploy. Both workflows are
deliberately restricted to run only in response to a real `push` event on this
repository (not a pull request build) — see `05-cicd-pipeline.md` for why that
distinction matters; it closes a real fork-hijack vulnerability found and fixed
today.

## Backend hosting — Render

The FastAPI backend runs as a Render **free-tier** web service per environment
(`remote-ai-platform-api` → `prod`, `remote-ai-platform-api-dev` → `dev`), built from
`apps/api/Dockerfile` (`production` target, see `infra/deploy/render.yaml`). Deploys are
triggered by the deploy workflows calling the service's deploy hook **with `?ref=<tested SHA>`**
(Render disables its own auto-deploy when `ref` is used); the workflow then waits for
`/health/version` to report that SHA. Migrations run on start (`start-production.sh`).
Region: Oregon (US). Free instances sleep after ~15 idle minutes; the `keep-render-warm`
workflow pings prod.

Runtime config set on Render (names only): APP_ENV, AI_PROVIDER=auto, GROQ_API_KEY,
CORS_ORIGINS, DATABASE_URL, SUPABASE_URL, REDIS_URL (+ legacy CELERY_BROKER_URL, read as a
fallback), MINIO_* (Supabase Storage S3), EMAIL_PROVIDER, RESEND_API_KEY, SENTRY_DSN,
RATE_LIMIT_*, STRIPE_* (unused while payments are off), WEB_CONCURRENCY. Obsolete and
ignored by the app: AUTH_PROVIDER, JWT_SECRET_KEY, KEYCLOAK_*, CELERY_RESULT_BACKEND,
SEED_DEMO_DATA, PAYMENT_PROVIDER — safe to delete from the dashboard.

## Database — Supabase Postgres

Two entirely separate Supabase projects, one per environment (not a
shared project with logically-separated data) — this is a genuine hard
isolation between prod and dev data. The backend connects through Supabase's
session-mode connection pooler rather than the direct database host, because
the direct host is IPv6-only and unreachable from Render's network.

## Storage — Supabase Storage

Both environments use Supabase Storage (S3-compatible) for file storage —
principally resumes. Private files (resumes) are served via short-lived
presigned URLs generated per request after an owner/admin authorization check,
never via a permanent public link.

## Redis

Documented as shared by dev and prod. Every key and pub/sub channel is prefixed with the
environment (`APP_ENV`), so rate limits, cache entries and chat fan-out never cross environments
even on a shared instance. Redis is disposable: nothing irreplaceable is stored there.

## DNS / CDN

**Cloudflare** provides DNS and CDN for `remoteaiplatform.com`. The domain is
registered at **Porkbun**, with its nameservers delegated to Cloudflare.

## Email — Resend

Transactional email (OTP sign-in codes, in-app notification emails) is sent via
**Resend**, from a verified sending domain on `remoteaiplatform.com`
(`hello@remoteaiplatform.com`). Both the prod and dev Supabase projects are
configured to send through the same Resend account/credentials via custom SMTP
— Supabase's own default email provider does not support the template
customization this product's OTP flow needs (and blocks template edits
entirely on free-tier projects using the default provider), which is why custom
SMTP was wired in for both environments. One real incident worth knowing: an
earlier change to the sender address via Supabase's Management API replaced
the *entire* SMTP configuration instead of merging just that one field, briefly
wiping email sending on both projects — caught and restored the same day. If
you ever change a single SMTP-related field via that API, re-verify all the
other SMTP fields are still populated afterward.

## Error monitoring — Sentry

Two separate Sentry projects: one for the frontend (`@sentry/nextjs`), one for
the backend (`sentry-sdk[fastapi]`). The backend integration is a genuine no-op
— no network calls, no overhead — until a `SENTRY_DSN` value is actually
configured for a given environment.

## Reference: account/organization ownership and IDs

Not secrets (no credential values), but the concrete account/resource
identifiers referenced elsewhere in this handbook, kept in one place:

| Account | Owner / ID |
|---|---|
| GitHub | `gokul-227` (personal account) |
| Render | workspace `tea-d35655r3fgac73b6i2j0` |
| Cloudflare | account `7d68e8481f9b2c593027df283a43b6be` |
| Supabase | organization `hgelrtnobjyxojqksltf` ("Remote-AI-Platform") |
| Stripe | account `acct_1UAGip0wWqj47Hzf` (Germany, individual account type) |
| Domain registrar | Porkbun (`remoteaiplatform.com`), nameservers delegated to Cloudflare |

Application-level admin accounts (not infrastructure): `gokulraj22797@gmail.com`
and `gokulraj22797@outlook.com` are ADMIN role; `ci-service@remote-ai-platform.internal`
is a service ADMIN account used only by the scheduled job-sync CI workflow.

## Free-tier limits worth respecting before changing infrastructure

- **Render**: 750 total instance-hours/month shared across the whole account —
  adding another always-on service risks exceeding this. Only prod is kept
  warm 24/7 (via cron-job.org pinging `/health/live`); dev is allowed to sleep.
- **Supabase**: a limited number of free active projects per organization —
  currently using both available slots (prod + dev). Check current limits
  before provisioning a third.
- **Cloudflare Workers**: 100k requests/day per Worker — not a practical
  concern at current traffic.

## Verified vs. not verified (2026-09-27)

Verified through the providers' APIs (names/settings only, no secret values read into docs):
Render env var **names** for both services; Supabase project regions and auth settings; Cloudflare
email routing; Groq model availability; Infisical secret names. `AUTH_PROVIDER` no longer matters
(Supabase is the only path in code).

Not yet verified: Supabase Storage bucket ACLs and RLS policies (no read-only API was used), backup
restore (Supabase free tier has no automatic backups — see `go-live/backup-and-restore.md`), and whether
dev and prod really share one Redis (the app now namespaces every Redis key by `APP_ENV`, so sharing is
safe either way).

## Email

- Sending: Supabase Auth sends sign-in codes through Resend SMTP from `hello@remoteaiplatform.com`
  (rate limit set to 60/hour in Supabase → Authentication → Rate Limits).
- Receiving: the only public address is **contact@remoteaiplatform.com**, forwarded by Cloudflare
  Email Routing to the owner's private inbox (never published). Verified 2026-09-27.

## Supabase projects

- `Remote-AI-Platform` (`cvjypjsjmamoppwhuwdl`, eu-west-1) — production. Redirect allow list is
  production origins only (localhost removed 2026-09-27).
- `remote-ai-platform-dev` (`wqugjgtjjmixzsqqcmld`, eu-west-1) — **paused** (free-tier inactivity);
  restore from its dashboard before using dev.remoteaiplatform.com.
