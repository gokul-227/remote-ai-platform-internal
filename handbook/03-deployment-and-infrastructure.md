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
- **Prod** (`remoteaiplatform.com`) — the real product, serving real users and
  real (live-mode) Stripe payments.

## The branch model

Promotion path: **feature branch → `dev` → `prod`**, enforced by CI, not just by
convention. A dedicated workflow (`enforce-branch-flow.yml`) rejects any pull
request opened directly against `prod` unless its source branch is `dev` — this
runs on every PR targeting `prod` and needs no repository secrets to do it (it
only inspects the PR's branch name). Both `dev` and `prod` are protected
branches: no direct pushes, even for repo admins; merging requires a passing PR
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

The FastAPI backend runs as a single Render **free-tier** web service, one per
environment, built from `apps/api/Dockerfile`'s `production` Docker target
(defined in `infra/deploy/render.yaml`, this repo's Render Blueprint). Render's
own git integration auto-deploys directly on a push to each service's tracked
branch (`prod` for the production service, `dev` for the dev service) —
independent of GitHub Actions; there is no GitHub Actions workflow that deploys
to Render.

**The free tier's 512MB RAM ceiling is a real, binding constraint, not a
theoretical one.** On 2026-09-06, after merging a batch of security and feature
fixes, the backend started failing to boot on Render with "Out of memory (used
over 512Mi)" roughly 20-30 seconds into every fresh process start, on both `dev`
and `prod`. Render correctly refused to swap production onto the broken build
and kept serving the previous, working version, so production was never fully
down — but it could not receive any of that day's fixes (including the
WebSocket fix described in `01-technical-architecture.md`) until the memory
issue was resolved. The root cause turned out to be a duplicate,
memory-heavy Alembic migration run happening in-process during application
startup on top of the migration Render already runs separately; removing the
duplicate run fixed it. The fix is merged and live on both branches as of this
writing. Because of this incident, `WEB_CONCURRENCY` is deliberately kept at 1
(a single Uvicorn worker process) on both Render services — see the Blueprint's
own comment for the reasoning. Treat any future change that adds startup-time
work, a second worker process, or a larger dependency footprint as something to
load-test against this ceiling before merging.

Render's free web service tier also spins down after roughly 15 minutes of no
inbound traffic and takes 10-60 seconds to cold-start on the next request. The
production service is kept warm continuously by external pings from
**cron-job.org** (primary) and a GitHub Actions scheduled workflow
(`keep-render-warm.yml`, secondary) hitting `/health/live` every few minutes —
together this uses close to the full 750 free instance-hours/month Render
allots per account, so adding a second always-on free Render service would
risk exceeding that budget. The dev service is allowed to sleep.

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

A single Redis instance is currently shared by **both** the dev and prod
environments — a deliberate free-tier simplification, not an oversight, and
called out explicitly as a known tradeoff. If dev testing ever needs to be
fully isolated from prod (for example before a risky migration test), a
separate Redis instance should be provisioned for dev first.

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

## What could not be independently verified

The exact current values of environment-specific configuration (which AI
provider/model is live, whether `AUTH_PROVIDER` is explicitly set to
`"supabase"` in each environment's Render dashboard versus relying on some other
mechanism, live Redis auth configuration, and the actual Supabase Storage
bucket ACL) live only in the Render/Supabase dashboards themselves and are not
committed anywhere this handbook can read. The live frontend's behavior
(exclusively Supabase-based sign-in) makes it near-certain both environments
have `AUTH_PROVIDER=supabase` set at runtime even though that is not the
in-code default — but this handbook states that as an inference from observed
behavior, not a confirmed fact, since the actual environment variable value
was not directly inspected.
