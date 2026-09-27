# Go-live evidence index

The single index of go-live evidence, one section per phase of `Remote-AI-Platfrom-Go-Live.md` (the audit dated
2026-09-27). New findings are also added to `findings-register.md` (IDs from the audit, e.g. `REL-01`, `SEC-02`).
Nothing here contains credential values, tokens, database URLs or personal data.

**Evidence labels:** verified live · verified source · verified CI/test · documented only · hypothesis · blocked by access.

---

## Phase 00 — discovery and reproducible baseline (2026-09-27)

**Status: done for everything reachable from this machine.** Dashboard-only facts are listed under "Blocked by access".

### Identity

| Item | Value | Evidence |
|---|---|---|
| App checkout | `remote-ai-platform`, branch `prod`, clean tree | verified source |
| HEAD / `origin/prod` / `origin/dev` | `07867833` — all three identical | verified source |
| Local `fix/go-live-p0-containment` | ancestor of `prod` (0 unique commits) — fully merged, safe to delete | verified source |
| Internal repo | `main` at `6b91ca2`, clean | verified source |
| Live API (Render prod) `/health/version` | `07867833`, env production | verified live 12:19 UTC |
| Live web `remoteaiplatform.com/api/version` | `07867833`, deployment `36283984440-1` | verified live 12:23 UTC |
| Live web `*.workers.dev/api/version` | `07867833`, same deployment id | verified live |
| Dev web `dev.remoteaiplatform.com/api/version` | `07867833`, env dev | verified live |
| Dev API `remote-ai-platform-api-dev.onrender.com` | **no response in 60 s** (`/health/live`, `/health/version`) | verified live |
| Migration head | single head `036_supabase_only_auth` | verified source + fresh-PG run |

**G01 (provenance):** web and API in production both serve the CI-tested SHA `07867833`. The audit's command-line
403 on the web host did not reproduce (curl and a Googlebot UA both get 200) — it was client-specific.

### Baseline checks at `07867833` (local, macOS, Node 26.5 / npm, Python 3.11.16, Docker 29.6)

| Check | Command | Result | Time |
|---|---|---|---|
| Install | `npm ci` (root) | 846 packages | 12 s |
| Web lint | `apps/web: npm run lint` | 0 errors, **332 warnings** (almost all `no-explicit-any`, see UX-01) | — |
| Web types | `npx tsc --noEmit` | clean | — |
| Web unit | `npm test` (Vitest) | 9 files, **29 passed** | 1 s |
| Web build | `npm run build` | ok | ~20 s |
| Cloudflare build | `npm run cf:build` | ok ("OpenNext build complete") | 19 s |
| API lint | `.venv/bin/ruff check app` | clean | — |
| API types | `.venv/bin/mypy app` | clean, 139 files | — |
| API tests | `.venv/bin/pytest` (SQLite) | **352 passed** | 29 s |
| Fresh PostgreSQL migration | `alembic upgrade head` on disposable `postgres:16-alpine` | 000 → 036 ok, 48 tables, single head | 4 s |
| Hermetic E2E | compose `rap-e2e` rebuilt at HEAD + mock Supabase; `npx playwright test` | **29 passed** (journeys, navigation, privacy, WCAG AA scans) | 31 s |
| UX-02 repro | ad-hoc Playwright spec (not committed): header "Sign in" by click and Enter at 1280 and 375 px, then Back | **4 passed — not reproduced** in Chromium | 1 s |

The prior counts in `continuation.md` (352 / 29 / 29) are now reproduced fresh.
Noise, not failure: LiteLLM prints `ValueError: I/O operation on closed file` from an atexit hook after pytest ends.

### CI / repository settings (read with `gh api`, verified live)

| Item | Finding |
|---|---|
| Branch protection `prod`, `dev` | required checks: Backend, Frontend, E2E, Migration validation; strict; **`enforce_admins` false**; 0 approvals; force-push and deletion off; no rulesets. **SEC-02 confirmed open.** |
| Security checks as release gates | gitleaks, CodeQL and dependency-review are **not** required checks and do not gate deploys. **OPS-03 confirmed.** |
| CodeQL trigger | `push: [main, dev]` — the default branch is `prod`, there is no `main`. `prod` has not been analysed on push since 2026-09-05; the one open alert (`py/stack-trace-exposure`, `core/health.py`) is from `d54b1da` and points at code that now uses `_public_error`. **New: OPS-05.** |
| Repo secrets (names + last update only) | `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_API_TOKEN` (09-06), `CLOUDFLARE_ZONE_ID`, `NEXT_PUBLIC_SENTRY_DSN`, `NEXT_PUBLIC_SUPABASE_{URL,PUBLISHABLE_KEY}[_DEV]`, `PROD_ADMIN_EMAIL`, `PROD_ADMIN_PASSWORD`, `RENDER_DEPLOY_HOOK_{DEV,PROD}` (09-06), `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_URL`. **None updated after 2026-09-06 → no rotation evidence for SEC-01.** `SUPABASE_SERVICE_ROLE_KEY` is not referenced by any workflow — candidate for removal (owner). |
| Dependabot / secret-scanning alerts | 0 open / 0 open |
| Collaborators | 1 (`gokul-227`, admin) |
| Action pinning | all actions SHA-pinned except `actions/upload-artifact@v4` in `ci.yml` (Phase 09) |

### REL-01 — dev backend deploy failure (run 36284002499): root cause

1. **The dev API is not answering at all** (60 s timeouts now; the deploy saw "no response" for the whole run).
   `start-production.sh` runs `alembic upgrade head` before uvicorn, and the owner checklist records the dev Supabase
   project as **paused** — so the most likely cause is the dev database being unreachable, leaving the instance unable
   to start. *Hypothesis* until the Render dev service log is read (blocked by access).
2. **The "10-minute" wait ran 4 h 50 min** (00:56 → 05:46 UTC): the polling `curl` has no `--max-time`, so each of
   the 40 probes could hang on the unresponsive host. Same pattern in `deploy-frontend.yml`. No job-level
   `timeout-minutes` either. This burns Actions minutes and hides failures for hours. *Verified CI log + source.*

Fix plan: bound every probe (`--max-time`), add `timeout-minutes` to deploy jobs (Phase 01/09 branch); owner decides
whether dev should run at all (restore the paused dev Supabase project, or suspend the dev Render service — see COST-01).

### Public surface checks (verified live)

- Security headers present on `/`: HSTS (preload), CSP, `X-Frame-Options: DENY`, `nosniff`, referrer and permissions policy.
- `robots.txt`: `Allow: /`, no `Sitemap:` line. `/sitemap.xml` → **200 `text/html`** (app shell). Any unknown path → **200** (soft 404). SEO-01 confirmed.
- **New: SEO-02.** `dev.remoteaiplatform.com` serves the same `Allow: /` robots and no `X-Robots-Tag` — the staging site is indexable (and its API is down).

### Other new findings from source

- **New: OBS-02.** `app/core/config.py` defaults `GIT_SHA` to a hard-coded old SHA (`9389640…`). `/health/version`
  prefers `RENDER_GIT_COMMIT`, so Render reports correctly, but **Sentry `release` uses `settings.GIT_SHA`** — every
  production event is tagged with the stale release, and any non-Render environment reports a fabricated SHA
  (observed: local compose API reports `9389640`).
- **New: COST-02.** `keep-render-warm.yml` keeps prod up ~744 h/month and its comment assumes prod is the *only* free
  Render service. A dev service exists too; the workspace pool is 750 h, so dev uptime can exhaust it and suspend prod
  for the rest of the month. *Hypothesis* on the account's actual entitlements (blocked by access).

### Capability matrix (surface → state)

`App.tsx` registers 51 hash routes; the API mounts 19 domain routers (~165 HTTP operations) + 2 WebSockets.
State: **W** working (covered by CI tests, and live where noted) · **P** partial · **G** gated off · **U** unknown/unproven.

| Capability | UI routes | API | State | Evidence / gap |
|---|---|---|---|---|
| Public job discovery | `jobs`, `jobdetail/<id>`, `search` | `/jobs`, `/search` | **W** live | 861 jobs, `X-Total-Count`; E2E. Long plain-text descriptions (UX-03); no crawlable URLs (SEO-01) |
| Sign-in (email code, OAuth) | `login`, `register`, `forgot`, `reset`, `callback` | Supabase + `/auth/me` | **P** | Mock Supabase in E2E only; real OTP email + Google/Microsoft/GitHub never executed (G03) |
| Professional profile + resume AI import | `onboarding`, `profile`, `settings` | `/engineers`, resume upload | **P** | Tests cover parse/fill-empty; real Groq completion + real storage round trip unproven (OBS-01) |
| Visibility / privacy of profiles | `profile`, `engineer/<id>` | `/engineers/{id}`, matching | **W** tests | Findings M/M2 fixed + tests; staging cross-user proof pending (G03) |
| Matching / recommendations | `recs`, `dash` | `/matching` | **P** | Anonymous heading implies personalization (UX-03) |
| Applications / saved jobs | `applications`, `saved` | `/applications`, `/saved-jobs` | **W** tests | E2E journey; imported vs direct application distinction to confirm (Phase 02) |
| Company profile, post job, candidates | `codash`, `coprofile`, `postjob`, `cojobs`, `candidates`, `company/<id>`, `companies` | `/companies`, `/jobs`, `/applications/company` | **W** tests | Multi-company / org membership rules undefined (PROD-01) |
| Network / connections | `network`, `engineers` | `/network` | **W** tests | Block semantics end-to-end unproven (TRUST-01) |
| Social feed, groups | `feed`, `groups`, `group/<id>` | `/social`, `/groups` | **W** tests | Reporting present; moderation journey unproven |
| Messaging | `messenger` | `/network/conversations`, WS `/messages/ws/{id}` | **P** | UI polls 8 s / 30 s; WS unused by UI; WS token in query (SEC-04) |
| Notifications | `notifications` | `/notifications`, WS `/ws/{user_id}` | **P** | Polling; no preferences/unsubscribe proven |
| Projects / board / quality | `projects`, `workspace`, `quality`, `worklog` | `/projects` (39 ops), `/quality` | **P** | Broad surface; launch visibility undecided (Phase 02) |
| Contracts / milestones | `contracts`, `contractsign/<id>` | `/contracts` | **W** tests | Lifecycle enforced (finding D) |
| Task marketplace / submissions / reviews | `taskmarket`, `submissions`, `reviews`, `work`, `talent`, `ride` | `/projects`, `/trust` | **P** | Launch visibility undecided (Phase 02) |
| Payments / escrow / earnings | `earnings`, `copayments` | `/payments` | **G** | `MARKETPLACE_PAYMENTS_ENABLED=false`; `/payments/escrow` 503 live; wallet currency mix (PAY-02) |
| Billing / subscriptions | — | `domains/billing` (entitlements, no router) | **G** | No checkout; Stripe plans deferred by owner |
| Trust / verification | `security`, `verifications` | `/trust` | **P** | Badge semantics undefined (Phase 07) |
| Data export / account deletion | `settings` | `GET /auth/me/export`, `DELETE /auth/me` | **P** | E2E passes; DATA-01 (identity not deleted) and DATA-02 (export scope) open |
| Admin console | `admin*`, `reports`, `audit`, `sync`, `aiusage`, `health`, `flags`, `orgs` | `/admin`, moderation | **W** tests | Global ADMIN only; service account also ADMIN (Phase 14) |
| Job ingestion | — | `/jobs/sync` via scheduled workflow | **W** CI | Runs every 6 h; freshness/completeness not measured |
| Help, legal | `help`, `terms`, `privacy`, `impressum` | — | **P** | Operator identity "Coming soon" (LEG-01) |
| Analytics | — | `/analytics` | **U** | Consent + funnel design pending (Phase 12) |

### Environment and provider inventory (names only)

- **Render API** (blueprint `infra/deploy/render.yaml`: free plan, Oregon): blueprint sets `APP_ENV`, `DEBUG`, `PORT`,
  `WEB_CONCURRENCY`, `AI_PROVIDER`, `RATE_LIMIT_*`; dashboard-set names per `config.py`: `DATABASE_URL`, `REDIS_URL`,
  `MINIO_*` (Supabase Storage S3), `SUPABASE_URL`, `SUPABASE_JWT_AUDIENCE`, `CORS_ORIGINS_RAW`, `GROQ_API_KEY` (+ optional
  `GEMINI/CEREBRAS/OPENROUTER/MISTRAL/OPENAI_API_KEY`), `EMAIL_PROVIDER`, `RESEND_API_KEY`, `SENTRY_DSN`,
  `PAYMENT_PROVIDER`, `STRIPE_*`, `MARKETPLACE_PAYMENTS_ENABLED`, `USAJOBS_*`. Actual values/presence: **blocked by access**.
- **Cloudflare Workers**: `remote-ai-platform` (prod), `remote-ai-platform-dev`; observability on; build-time
  `NEXT_PUBLIC_*` from GitHub secrets.
- **Supabase**: prod project (Ireland per handbook), dev project paused (owner checklist).
- **GitHub Actions**: secrets listed above; variables `PROD_API_URL`, `PROD_API_BASE_URL`.
- **Unused**: Northflank (owner). Infisical holds copies of keys (owner checklist).

### Blocked by access — owner actions needed

| Needed | Why | Owner |
|---|---|---|
| Render dev service logs + plan/hours usage | REL-01 root cause, COST-02 | gokul-227 |
| Supabase dashboard: storage bucket policies, RLS / Data API exposure, backups | DATA-03, OPS-01 | gokul-227 |
| Cloudflare: WAF/bot rules, Email Routing state | OPS-02 | gokul-227 |
| Evidence of key rotation (dates per identifier) | SEC-01 | gokul-227 |
| Decision: restore `enforce_admins` now (recommended) | SEC-02 | gokul-227 |

### Security notes for this phase

No production data copied. The local stack uses `.env.example` values and mock Supabase (cannot resolve production
credentials). No secret values were printed; secret inventory is by name and update date only.
