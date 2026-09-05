# Required from you — running list

Kept up to date as new items come up. Each item says exactly what to do and why it can't be done via API/automation.

## 1. Merge open PRs (ongoing)
Governance you set up requires every change to go through a PR you merge manually. Check `gh pr list` or the repo's Pull Requests tab for what's currently open and waiting.

## 2. Sentry DSNs
Once the Sentry code-wiring PR lands (in progress), create a free Sentry account + two projects (Python/FastAPI, Next.js) and paste the two DSN keys so they can be added as `SENTRY_DSN` (Render) and `NEXT_PUBLIC_SENTRY_DSN` (Cloudflare Worker + GitHub secret).

## 3. Dev environment storage — S3 access keys (2 minutes, dashboard-only)
Supabase's Management API has no endpoint for creating S3 access keys — confirmed by checking its OpenAPI spec, this is a dashboard-only action. The dev project's storage buckets are already created (`remote-ai-platform-resumes`, `remote-ai-platform-assets`). To finish isolating dev's storage from prod's:
1. Go to the **dev** Supabase project (ref `wqugjgtjjmixzsqqcmld`) dashboard → **Settings → Storage → S3 Access Keys**.
2. Click **New access key**, name it e.g. `render-dev-backend`.
3. Paste the Access Key ID and Secret Access Key here — they'll be set as `MINIO_ACCESS_KEY`/`MINIO_SECRET_KEY` on the dev Render service (currently reusing prod's).

## 4. Dev environment Redis — needs its own Upstash instance
No Upstash API token is available in this environment. To separate dev's Redis from prod's (currently shared):
1. Go to **upstash.com** → create a new free Redis database (or reuse an existing free-tier one if you already have Upstash access elsewhere).
2. Paste the connection URL (`rediss://...`) here — it'll be set as `REDIS_URL`/`CELERY_BROKER_URL`/`CELERY_RESULT_BACKEND` on the dev Render service.

Alternatively, if separating this isn't worth the effort right now: dev sharing prod's Redis is low-risk (used for rate-limiting fallback and an unused Celery broker — no worker consumes it either way, see decision 0001). Fine to leave as-is and revisit later.

## 5. Legal review
Once the legal-pages-scaffold PR lands: a real lawyer needs to draft actual privacy policy, terms of service, and Impressum content. The Impressum specifically has a decision only you can make — see `docs/...` in that PR's description: German Impressumspflicht requires a real published name/address once the site takes real commercial traffic, even as an individual, which conflicts with wanting brand-only anonymity for now.

## 6. Pricing/monetization decisions
Not started yet — when it comes up, decisions about actual plan tiers and prices are yours to make; I can build the entitlement/billing architecture but not choose the numbers.
