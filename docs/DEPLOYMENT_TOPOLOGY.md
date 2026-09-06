# Deployment Topology (current, as of 2026-09-06)

> See `../handbook/03-deployment-and-infrastructure.md` for the polished,
> narrative version of this document (including today's Render OOM incident
> and its fix). This file stays the raw, IDs-included quick reference.

Public repo: https://github.com/gokul-227/remote-ai-platform (branches `prod` = production, `dev` = development; promotion path is feature branch -> `dev` -> `prod`, enforced by CI)

## Production

| Layer | Provider | Identifier | Live URL |
|---|---|---|---|
| Frontend | Cloudflare Workers (OpenNext adapter) | Worker: `remote-ai-platform`, Account ID `7d68e8481f9b2c593027df283a43b6be` | `https://remoteaiplatform.com`, `https://www.remoteaiplatform.com`, `https://remote-ai-platform.gokulraj22797.workers.dev` |
| Domain/DNS | Cloudflare (registered at Porkbun, nameservers delegated to Cloudflare) | Zone ID `2d6999e647a85544d1b3da85d1de8836` | `remoteaiplatform.com` |
| Backend API | Render (free web service) | Service ID `srv-d9uea4nlk1mc73elkm10` | `https://remote-ai-platform-api.onrender.com` |
| Database + Auth | Supabase (free project) | Project ref `cvjypjsjmamoppwhuwdl` | `https://cvjypjsjmamoppwhuwdl.supabase.co` |
| DB connection | Supabase session-mode pooler (NOT the direct db.*.supabase.co host -- IPv6-only, unreachable from Render) | host `aws-1-eu-west-1.pooler.supabase.com:5432`, user `postgres.cvjypjsjmamoppwhuwdl` | -- |
| Storage | Supabase Storage (S3-compatible, via boto3) | same project as above | -- |
| Payments | Stripe (LIVE mode) | Account `acct_1UAGip0wWqj47Hzf`, country DE, individual account type | -- |
| Keep-alive | cron-job.org (primary) + GitHub Actions schedule (secondary, unreliable alone) | pings `/health/live` | -- |

## Dev / Development environment

| Layer | Provider | Identifier | Live URL |
|---|---|---|---|
| Frontend | Cloudflare Workers | Worker: `remote-ai-platform-dev` | `https://dev.remoteaiplatform.com`, `https://remote-ai-platform-dev.gokulraj22797.workers.dev` |
| Backend API | Render (free web service) | Service ID `srv-dadgbpad0e5s73du4th0` | `https://remote-ai-platform-api-dev.onrender.com` |
| Database + Auth | Supabase (separate free project, fully isolated from prod) | Project ref `wqugjgtjjmixzsqqcmld` | `https://wqugjgtjjmixzsqqcmld.supabase.co` |
| Payments | `PAYMENT_PROVIDER=sandbox` (in-app fake provider) -- **deliberately never wired to real Stripe** | -- | -- |

**Known simplification (free-tier pragmatism, not an oversight):** dev shares prod's Redis instance, Supabase Storage bucket, and Resend email account. If dev testing ever needs to be fully isolated from prod (e.g. before a large migration test), provision separate instances of those three before proceeding.

## Accounts / organizations

- GitHub: `gokul-227` (personal account, owns both `remote-ai-platform` and this repo)
- Render: workspace owner ID `tea-d35655r3fgac73b6i2j0`
- Cloudflare: account `7d68e8481f9b2c593027df283a43b6be` ("Gokulraj22797@gmail.com's Account")
- Supabase: organization `hgelrtnobjyxojqksltf` ("Remote-AI-Platform")
- Domain registrar: Porkbun (`remoteaiplatform.com`, nameservers pointed at Cloudflare)

## Admin accounts (application-level, not infra)

- `gokulraj22797@gmail.com` -- ADMIN
- `gokulraj22797@outlook.com` -- ADMIN
- `ci-service@remote-ai-platform.internal` -- ADMIN, used only by CI (`scheduled-job-sync.yml`) to authenticate against the API

## CI/CD

- `ci.yml` runs on every push to `main`/`dev` and every PR: backend lint+typecheck+test, frontend lint+typecheck+build, E2E (Playwright against a full docker-compose stack), migration validation.
- `deploy-frontend.yml` / `deploy-frontend-dev.yml`: triggered by `CI` succeeding on `prod` / `dev` respectively, deploy via `wrangler deploy` to the matching Worker. Both are restricted to real `push` events on this repo only (a fork PR named `prod`/`dev` cannot trigger a deploy with this repo's secrets -- fixed 2026-09-06, see `reports/`).
- Render's own git integration auto-deploys the backend services directly on push to their tracked branch (`prod` for prod, `dev` for dev) -- independent of GitHub Actions.
- Both `prod` and `dev` are branch-protected: no direct pushes (enforced for admins too), PR + passing status checks required to merge. `prod` additionally only accepts PRs sourced from `dev` (enforced by `enforce-branch-flow.yml`).
- 2026-09-06: production briefly could not receive new deploys due to a Render free-tier OOM crash on boot (root cause: a duplicate in-process Alembic migration run at startup); fixed and merged to both branches same day -- see `decisions/0004-render-oom-blocks-deploy.md`.
- Auth: Supabase Auth is the live identity provider for both environments -- email OTP (passwordless) plus Google/Microsoft/GitHub OAuth, added 2026-09-06. A legacy self-issued-JWT path still exists in the backend but is not used by the live frontend -- see `handbook/01-technical-architecture.md`.

## Known free-tier constraints to respect when changing anything here

- Render free tier: 750 total instance-hours/month **shared across the whole account** -- adding another always-on service risks exceeding this. Only prod is kept warm 24/7 via cron-job.org; dev is allowed to sleep.
- Supabase free tier: limited number of free active projects per organization -- currently using 2 (prod + dev). Check before provisioning a 3rd.
- Cloudflare Workers free tier: 100k requests/day per Worker -- not a practical concern at current traffic.
