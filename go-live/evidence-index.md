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

---

## Phase 01 — critical containment and security foundations (2026-09-27)

**Status: code-side items implemented and tested locally on branch `fix/go-live-phase-01` (not pushed, not deployed).
Owner-blocked items remain open: SEC-01, SEC-02, DATA-03.**

### Commits (branch `fix/go-live-phase-01`, base `prod` `07867833`)

| Commit | Finding | Change | Test evidence |
|---|---|---|---|
| `cf4880f` | SEC-03 | Rate-limit buckets per client **and endpoint class** (sensitive / ai / ingest / general), not raw path; general = 30× base (300/min prod) since it now spans all ordinary requests; fallback capped at 10k keys with idle-first eviction; one pooled Redis client; unique ZSET members | 3 new tests failed before, pass after: path-variation bypass, class isolation, bounded fallback |
| `bb67d23` | REL-01, OPS-05 | `--max-time` on every curl in deploy/sync workflows; `timeout-minutes` on all deploy + CI jobs (5–10× observed); CodeQL on `[prod, dev]`; `upload-artifact` pinned to `ea165f8` | YAML parsed; `test_deployment_contract.py` 9 passed; observed CI durations 0–4 min vs ceilings 10–30 |
| `f066282` | OBS-02 | `GIT_SHA` = `RENDER_GIT_COMMIT` → `GIT_SHA` → `"unknown"`; `/health/version` and Sentry `release` share it | new test failed before (`9389640…`), passes after |
| `b69f033` | SEC-05, SEO-02 | Deployed web builds (`NEXT_PUBLIC_DEPLOY_ENV` production/dev) fail on missing/local/placeholder public env; `app/robots.ts` allows crawling only in production; non-production responses carry `X-Robots-Tag: noindex, nofollow` | 4 Vitest tests; three real builds: no env → `Disallow: /` + header; production env → `Allow: /`, no header; production without API URL → build refuses; `cf:build` ok |
| `248d0c2` | token negatives | Tests: `alg=none`, HS256 signed with the public key, malformed tokens → 401 (verifier already pins ES256/RS256/EdDSA) | 6 new tests pass (behaviour already correct) |
| `ee70453` | (regression from `b69f033`) | Keep `apps/web/public/` (`.gitkeep`): removing the static robots.txt emptied it and the web Docker image failed at `COPY public` | Caught by the local E2E rebuild before push; image builds |

### Reviewed, no change needed (verified source)

- Client IP for limits = last `X-Forwarded-For` hop (appended by Render). *Documented only* that Render appends; confirm in staging with a spoofed header once dev is healthy.
- Backend refuses production boot with dev DB/MinIO creds, `DEBUG`, missing `SUPABASE_URL`, wildcard CORS (`validate_production_settings`).
- Unhandled exceptions → generic `Internal server error`; health errors via `_public_error`.
- Public build output (`.open-next/assets`): 0 source maps, no secret patterns, no internal-repo references.
- Money movement stays off (`MARKETPLACE_PAYMENTS_ENABLED=false`; live `/payments/escrow` 503).

### Secret scanning (gitleaks 8, redacted)

- Branch commits `prod..HEAD`: no leaks.
- Full history: 5 `generic-api-key` hits — 3 false positives (E2E dummy key, `ci-test…` JWT fixture, compose `${…}` interpolation); 2 are a real-format Supabase **publishable** key committed 2026-09-04 in `ci.yml` / `apps/web/.env.example` (already replaced by placeholders at HEAD). Publishable keys are public by design (shipped in the browser bundle); their safety depends on RLS → tracked under DATA-03. No history rewrite proposed.

### Owner-blocked (unchanged)

- **SEC-01** rotation: no repo secret updated after 2026-09-06. Order: create new key at provider → update Infisical → Render / GitHub secrets → verify consumers (deploy, sync, `/health/dependencies`) → revoke old key → confirm old key fails. `SUPABASE_SERVICE_ROLE_KEY` repo secret is unused by workflows — delete rather than rotate.
- **SEC-02** `enforce_admins`: restore on `prod` and `dev` (commands in `progress.md`). Break-glass: owner temporarily disables, records reason in this file, re-enables.
- **DATA-03**: Supabase storage bucket policies / RLS / Data API exposure need a dashboard or Management-API read by the owner.

### Phase 01 validation at `ee70453` (local)

| Check | Result |
|---|---|
| API ruff / mypy / pytest | clean / clean (139 files) / **362 passed** (352 baseline + 10 new) |
| Web tsc / lint / Vitest | clean / 0 errors, 332 warnings (unchanged) / **33 passed** |
| `next build`, `cf:build` | ok |
| Hermetic E2E (stack rebuilt at `ee70453`, containers created 12:38 UTC) | **29 passed**; API `/health/version` → `git_sha: "unknown"` (no hard-coded SHA); web `/robots.txt` → `Disallow: /` + `X-Robots-Tag: noindex, nofollow` |
| gitleaks `prod..HEAD` | no leaks |

Process note: a first E2E rerun reported 29 passed against **stale containers** — the image build had failed
(`COPY public`) and macOS has no `timeout`, so the wait loop was skipped silently. Evidence now always records the
container creation time and `/health/version` of the stack under test.

**Not yet:** pushed, CI-verified, deployed to dev, or verified live. Dev deploy cannot be verified until the dev API
is reachable (REL-01 owner action).

### Next-phase findings confirmed in source while here (for Phase 04)

- **API-01 — severity raised to P0.** `AuthService.get_or_create_user` matches by email and **overwrites a non-null
  `auth_subject`** with the new token's subject. Supabase does not auto-link identities whose email the provider has
  not verified, so a second Supabase user with a victim's email can exist; its first API call would take over the
  victim's app account. Fix: never overwrite a set `auth_subject`; refuse with a support path.
- **DATA-01 confirmed.** `DELETE /auth/me` deletes app rows + resume object but not the Supabase user; the still-valid
  access token (≤1 h) or a refresh re-enters `get_or_create_user` and silently provisions a new blank account.

---

## Phase 02 — product scope and launch contract (2026-09-27)

**Status: implemented and tested locally on `feat/go-live-phase-02` (stacked on phase 01). Launch contract drafted in
`launch-contract.md`; three visibility rows + support targets await owner confirmation.**

| Commit | Change | Evidence |
|---|---|---|
| `e9fdc44` | `POST /applications/jobs/{id}`: 409 for imported listings ("apply on the source site") and closed postings, 404 for removed; UI says so when an imported listing has no link | 2 of 3 new tests failed before (201 for imported and closed), all pass after; lifecycle test fixture now has an owning organisation (assertions unchanged) |
| `11ee9e5` | Removed "Escrow-backed payments — funds are held…" (security page) and "Top job picks for you" (unpersonalised feed → "Latest remote jobs", matches `posted_at DESC`); `claims.test.ts` bans escrow/picks/NK+/SOC 2/guaranteed-outcome copy | claims test: 2 failed on old copy, 5 pass after |

Validation: API 365 passed; web tsc clean, lint 0 errors (332 warnings, unchanged), Vitest 38 passed, `next build` ok.

Checked, no change: payments gated in API (503) and UI (`payments_enabled` from `/payments/wallet`); Help's
match-score weights equal `matching/service.py`; role switching is audited and cannot self-assign ADMIN.

---

## Phase 03 — architecture and incremental refactoring (2026-09-27)

**Status: implemented and tested locally on `refactor/go-live-phase-03` (stacked). ADRs 001–004 in `handbook/adr/`.**

| Commit | Change | Evidence |
|---|---|---|
| `b83639d` | `useApi` data keyed to account + path + params: never shows the previous resource while loading or after its request fails; drops data on account change; reload of the same resource keeps it | 3 of 4 new hook tests failed before, all pass after |
| `6273212` | Sign-out also removes an abandoned sign-up's `pending_registration` (email, name) | auth test extended |
| `08ca3ac` | Mechanical Prettier format of `apps/web/src` (max line 1,811 → 155; UI now 10.8k lines); root config aligned to the code's double quotes; CI `Formatting` step; `npm run format` | esbuild-minified output of all 44 files identical except 9 JSX trailing-space splits (same rendered text); tsc/lint/tests/build pass |
| `e720aea` | http(s)-only links: shared `HttpUrlIn` on 9 input fields (post link/image, org website/logo, profile image/GitHub/LinkedIn/portfolio, project links); response schemas don't re-validate; UI `safeHref` on every user/job-board link (`href`, `img src`, `window.open`) | 6 API tests failed before, pass after; 2 `safeHref` unit tests |

Validation at `e720aea`: API 371 passed (ruff, mypy clean); web tsc clean, lint 0 errors / 332 warnings, Vitest 44
passed, formatting check clean; hermetic E2E **29 passed** (containers created 13:03 UTC from `e720aea`).

Architecture facts established: no `dangerouslySetInnerHTML` or inline app scripts (CSP comment corrected); 91 of 174
API operations declare a response model (ADR-004 plan); browser storage holds only tokens, the user object and `rap-*`
resource ids, all cleared on sign-out.

Deferred within Phase 03 (carried forward): splitting `rap_*.tsx` by domain and replacing `any` — done per endpoint as
response models are added (ADR-004), not as a blanket rewrite.

---

## Phase 04 — database, identity and API integrity (2026-09-27)

**Status: implemented and tested locally on `fix/go-live-phase-04` (stacked). Migrations 037, 038 rehearsed on PostgreSQL 16.**

| Commit | Finding | Change | Evidence |
|---|---|---|---|
| `36ff24e` | API-01 (P0) | Never overwrite a set `auth_subject` on email match → 409 with support path; legacy rows without identity still link. Sign-in screens show the API's reason and end the refused Supabase session | Takeover reproduced (200) then 409; 2 tests |
| `202e8da` | DATA-01 (P0) | Migration 037 `deleted_identities` (SHA-256 of subject + time); tokens issued ≤ deletion → 401; deliberate new sign-in → fresh account. Supabase Admin API delete when `SUPABASE_SERVICE_ROLE_KEY` set (off otherwise; provider failure never undoes erasure). UI signs out everywhere after deleting | 11 tests incl. mocked Admin API (URL, headers, 404, 5xx, non-UUID); E2E now asserts old token → 401 (was 404 = silently re-created account); PG 036→037→036→037 |
| `cd71d90` | DATA-02 (P0) | Export is an explicit source list (`auth/export.py`): rows where the user is the subject; drops staff ids/notes, capability URLs, provider refs; adds received messages, profile-keyed matches, own audit/analytics events, organisation postings, subscriptions (was silently skipped — model not imported at runtime). Guard test: every user/profile-linked column exported or excluded with a reason | Old export failed both privacy fixtures (admin export contained others' reports; reporter export had moderator notes) |
| `238a212` | integrity | Migration 038 `uq_job_application_user_job` (refuses, never deletes, if duplicates exist); apply insert in savepoint → 409; likes use SQL-expression counter + absorb duplicate insert | On PostgreSQL the old code stored **8 concurrent likes as 2** and 500'd a double like; new `test_postgres_concurrency.py` (CI migration job). 038's first draft id was 33 chars > `alembic_version` VARCHAR(32) — caught by rehearsal; guard test added |
| `f887114` | OPS-04 | Advisory lock around startup migrations + `lock_timeout 15s` | Two concurrent migrators on fresh PG: before = one crashed (`UniqueViolationError`), after = both exit 0 (×4); failing migration exits 1 and releases lock. CI now runs two migrators at once |
| `16254cb` | SEC-04 | WebSockets closed (1008) before token handling unless `FEATURE_REALTIME_WEBSOCKETS`; single realtime strategy = polling; uvicorn `--no-access-log` (it logged `?token=`) | WS auth tests run with flag on; new flag-off test |
| `6dc90e4` | JWKS | Verification in threadpool; JWKS timeout 30 s → 5 s; unknown `kid` refresh at most once/min | 2 tests (5 made-up kids → 1 fetch) |
| `ec81274` | IDOR audit | `/trust/{scores,reviews,verifications}/{user_id}` follow profile visibility; public sees VERIFIED badges only, no `verifier_notes`/`reviewed_by_id` (were anonymous and complete) | 2 tests; existing trust tests fetch as owner (assertions unchanged) |
| `1ac2548` | IDOR audit | `GET /jobs/{id}`: inactive → org + admin only; removed → admin only | test |

**Object-authorization audit:** 87 id-taking endpoints — 80 require auth (51 user, 29 role); the 7 unauthenticated ones
reviewed: 2 fixed above (trust ×3 counted as one fix, jobs/{id}), companies/{id} and jobs/company/{id} public by design
(active jobs only), engineers/{id} already visibility-checked. Existing two-persona tests cover contracts, projects,
applications, payments, admin.

**Pre-existing drift recorded (not changed):** 23 model-declared indexes never created by migrations (→ Phase 11), 19
DB indexes not in models, `plans`/`subscriptions` models not imported by `app.main`, `post_likes` "missing"
`uq_post_like` is redundant with its composite PK. Response schemas with input constraints (e.g. `title` min length)
can 500 on odd stored data (follow-up, ADR-004).

**Owner dry-run before pushing (read-only SQL in Supabase):**
`SELECT count(*) FROM (SELECT user_id, job_id FROM job_applications GROUP BY 1,2 HAVING count(*) > 1) d;`
must be 0, or migration 038 will stop the new instance from starting (Render keeps the old one).

Validation at `1ac2548`: API 399 passed + 3 PostgreSQL-only (run locally against PG: 3 passed); ruff/mypy clean; web
tsc clean, Vitest 44, formatting clean; E2E **29 passed** (stack built from `1ac2548`, 13:39 UTC).

---

## Phase 05 — design system, responsive UX and accessibility (2026-09-27)

**Status: implemented and tested locally on `feat/go-live-phase-05` (stacked).**

Method: an audit sweep of **240 screen views** (visitor 11 routes, professional 27, organisation 11, admin 11 × 320/375/768/1280 px) recording horizontal overflow and axe WCAG 2.2 A/AA results. Baseline: **8 overflowing views, 3 serious rules** (`scrollable-region-focusable` ×5 pages, `target-size` Help/Talent, `color-contrast` Notifications). After fixes: **0 overflow, 0 violations** except one triaged artifact (Talent name link "obscured" by the fixed mobile bottom bar at axe's scroll position; clears on scroll). Limits: fresh accounts, so populated states are less covered.

| Commit | Change |
|---|---|
| `a0d13c2` | Overflow fixes (search inputs `min-w-0`, sort chips wrap, local tab row scrolls, grid item `min-w-0`); `TableScroll` (focusable, labelled region) around all 7 tables + pipeline/transactions scrollers — root cause a global `div:has(>table){overflow-x:auto}` rule; Help `<summary>` 24px targets; Talent name is a link; contrast fixes; tabs expose `aria-pressed`. New `responsive.spec.ts`: every signed-in screen × 4 widths, no overflow, no serious/critical axe |
| `4f5f165` | UX-03 ingestion: `clean_rich_text` keeps paragraphs/headings/bullets (0 of 40 live descriptions had a line break; real Arbeitnow sample 1 → 25 lines, median 57 lines over 325 jobs); admin re-clean no longer flattens |
| `f00d92f` | UX-03 UI: `RichText` renders headings, paragraphs, lists at `max-w-prose`; React elements only (markup-looking text stays text) |
| `e5468bd` | `useDialogFocus` shared by Modal + main menu + project task panel + job drawer (menu had no Escape/focus handling; the other two had no dialog semantics). E2E: menu by keyboard, focus trapped, Escape restores focus |
| `2f97d62` | `Notice`: one always-mounted polite live region per screen for 19 outcome toasts (were mostly click-to-dismiss buttons); `prefers-reduced-motion` rule |
| `0b6106c` | 11 navigations put the id in the URL (`#engineer/<id>` …) instead of storage + bare route |

Validation at `0b6106c`: web tsc clean, lint 0 errors (332 warnings, unchanged), Vitest **49**, formatting clean; E2E **32 passed** (29 + 2 responsive + 1 keyboard menu; containers 14:03 UTC); API 404 passed.

Not done (needs a person/device): manual VoiceOver/NVDA pass; Safari/iOS retest of UX-02; 200% zoom was approximated by the 375/768 px widths, not tested as browser zoom. Remaining `button`-as-link navigation outside entity names is a known follow-up.

---

## Phase 06 — core journeys and AI reliability (2026-09-27)

**Status: implemented and tested locally on `feat/go-live-phase-06` (stacked). Migrations 039, 040 rehearsed on PostgreSQL.**

| Commit | Finding | Change | Evidence |
|---|---|---|---|
| `ae65253` | AI-01 | Reserve-then-settle metering: RESERVED row with `AI_RESERVATION_TOKENS` estimate, checked + committed in its own transaction under an advisory lock; settled in another independent transaction; platform cap always applies (also with no signed-in user) | 3 new tests failed on old code (rollback erased usage; no-actor skipped cap; in-flight not counted). PostgreSQL: with the check-to-insert window widened, **5 of 5** concurrent calls slipped through a 1-call allowance without the lock, exactly 1 with it |
| `5b580af` | uploads | Read ≤ limit+1 bytes; DOCX must contain `word/document.xml`, ≤ 1,000 entries, ≤ 30 MB unpacked (a 60 MB zip bomb compressed below 5 MB passed before); PDF ≤ 20 pages parsed | 4 tests |
| `515a9a1` | ingestion | Migration 039 `last_seen_at`/`expired_at`; imported jobs unseen among their source's recent listings for 30 days expire (freshness policy; "may no longer be open" copy); only after a successful fetch; posted jobs never; lists/search exclude, apply refuses, old links show a notice | 2 tests; all 900 live jobs < 14 days old → first run expires none. Hypothesis "external_id collides across sources" **disproved** (ids are source-prefixed) |
| `ab132ff` | TRUST (block) | Migration 040 `user_blocks`; `/blocks` API (only the blocker lifts; nobody sees who blocked them); blocks refuse requests/conversations/messages both ways and remove connections; Block/Unblock on a person's page; export includes people you blocked only | 3 API tests; E2E `safety.spec.ts` (UI block → API 403 → unblock → 201). (First E2E draft was flaky: it read the profile before its save finished — fixed by polling) |
| `1718c02` | AI output | `clean_ai_profile_fields`: headline ≤ 255, bio ≤ 5,000, ≤ 50 skills ≤ 60 chars, typed, de-duplicated, own skills first; input schemas bounded (headline over its column was a PostgreSQL 500) | 3 tests |
| `61c1f2e` | visitor journey | Going to sign in remembers the page; email-code and OAuth sign-in return there once (validated plain route only) | 2 unit tests + E2E |

Verified, no change: all 9 server-side outbound HTTP calls use configured URLs (no SSRF surface); no email notifications are sent (`EMAIL_PROVIDER=none`, no call site passes `email_html`) — **gate: unsubscribe + preferences before enabling email notifications**; prompts are truncated to 4,000 chars; AI parse fills empty fields only and keeps the full parse as a draft.

Validation at `61c1f2e`: API **419 passed** + 5 PostgreSQL-only (passed against PG); web Vitest 51; E2E **34 passed** (stack 14:42 UTC).

Blocked: real OTP/OAuth and real Groq completions in staging (dev API down, REL-01); message send idempotency keys (UI prevents double submit only) — follow-up.

---

## Phase 07 — trust, safety, privacy and legal readiness (2026-09-27)

**Status: code-side items implemented on `feat/go-live-phase-07` (stacked). Legal facts and review remain owner-blocked (LEG-01).**

| Commit | Change | Evidence |
|---|---|---|
| `8329834` | Privacy notice reconciled with behaviour (LAST_UPDATED 27 Sep 2026): Resend = sign-in codes only (no notification emails exist); deletion is immediate + Supabase identity removed + one-way identity hash kept; blocks and AI allowance usage listed; public visibility of verified badges/reviews; blocks private. Production warns if `SUPABASE_SERVICE_ROLE_KEY` is unset. Moderation: post removal / job hiding notify the owner (what + appeal address, never reporter or moderator note); suspended accounts are told they are suspended and how to appeal (`SUPPORT_EMAIL`) | legal tests pass; 2 moderation tests failed before, pass after; warning test |

Verified, no change: moderation decisions are audit-logged (`MODERATION_DECISION`); reporters are never revealed to the reported person; `is_verified` (organisations) cannot be set through any API — only directly in the database; the professional "Verified" tag on Talent reads a field engineer profiles don't have (never shown). **Gap:** no admin UI to grant organisation verification — badge semantics must be defined before it is used (Phase 14).

**Owner actions required before this notice is accurate in production:**
1. Set `SUPABASE_SERVICE_ROLE_KEY` on Render (deletion promises Supabase erasure).
2. Confirm Supabase Auth sends sign-in codes through Resend SMTP (else correct the provider line).
3. LEG-01: operator legal name/address, VAT/register, governing law, supervisory authority, log/backup retention in `OPERATOR`.
4. Qualified legal review of Terms/Privacy.

Remaining uncertainty: GDPR/DSA obligations assessed only at the level of "notice matches behaviour"; no legal compliance is claimed.

---

## Phase 08 — testing and resilience verification (2026-09-27)

**Status: implemented and rehearsed locally on `test/go-live-phase-08` (stacked).**

| Commit | Change | Evidence |
|---|---|---|
| `d09f260` | Coverage measured correctly (`concurrency = greenlet`): real total **86%** (reported 71%; whole endpoints looked unexecuted); critical modules 87–100%, network router 71% (mostly switched-off WebSockets). CI test service fails under 80% | CI image run: 422 passed, 85.67% |
| `e87bb0e` | OPS-01 tooling: `ops/backup/{backup,restore}.sh` + `objects.py` (encrypted archive, SHA-256 manifest, row counts, version-pinned client via `PG_IMAGE`, refuses restoring over source or into an older server, `RESET_IDENTITIES`) | Rehearsal: 50 tables / 18 objects, backup 1 s, restore+verify 2 s, restored DB at head 040; negatives (wrong passphrase, tampered archive, over-source, version) fail. Runbook `backup-and-restore.md` |

Fault drill (verified locally): Redis stopped → `/health/ready` 200 `DEGRADED` (redis DOWN), public API 200 < 50 ms, all 11 E2E journeys pass (rate limiter fallback, message fan-out fallback).

Existing coverage retained: PostgreSQL concurrency suite (5 tests), 36 E2E tests (journeys, privacy, safety, navigation, responsive, a11y).

**OPS-01 remains open for production** until the owner runs the first real backup and a timed restore of it (G07).
Not done: provider outage drills against real staging (dev API down), mutation testing.
