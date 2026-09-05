# Phase 0 Current-State Report

Audit only — no code changed, nothing deployed, no env vars or schema touched. Compiled 2026-09-05.

**Scope note:** This report combines two kinds of claims, labeled so they can be weighed differently: **verified today** (checked live in this session — curl, DB query, ruff/mypy/pytest run, or a direct grep of current source) and **known from this session's own work** (built or fixed earlier in this same session, not re-discovered). Anything not checked either way is marked **UNKNOWN**.

**On the 503 seen externally:** `/health/live`, `/health/ready`, and `/api/v1/health` all returned 200/HEALTHY when checked directly — DB and Redis both healthy. The 503 was almost certainly a Render free-tier cold-start (sleeps after ~15 min idle; a cold boot can take 30-60s and serve a 503 mid-boot) — a known, accepted trade-off of this $0 hosting tier, not evidence of broken deployment. Still a real reliability weak point worth tracking (see P1 list).

## 1. Production topology (verified live)

| Layer | Provider | Status |
|---|---|---|
| Frontend | Cloudflare Workers (OpenNext) | GREEN — loads, security headers (CSP/HSTS/X-Frame-Options/etc.) confirmed present under the actual deployed Workers runtime |
| Backend API | Render free web service | GREEN right now — healthy, but only 2 Render services exist total (see Celery finding) |
| Database | Supabase Postgres | GREEN — migrations auto-apply on boot, confirmed on both prod and a freshly-provisioned dev project |
| Redis | Upstash (`REDIS_URL`) | YELLOW — reachable/healthy, but nothing consumes its Celery queue |
| Payments | Stripe, live mode | GREEN for the escrow rail — `charges_enabled: true` verified against Stripe's own API. More current than older repo docs describing payments as sandbox/mock |
| Environments | prod / dev, isolated DB+Auth+payments each | GREEN structurally; dev intentionally still shares prod's Redis/storage/email as a free-tier simplification |

## 2. Subsystem status matrix

| Subsystem | Status | Basis |
|---|---|---|
| Frontend build/deploy | GREEN | Build succeeds, security headers confirmed live under Workers runtime, E2E suite passes in CI |
| Backend API health | GREEN | Verified live (all 3 health endpoints) |
| Auth (Supabase + legacy custom JWT) | YELLOW | Closed an unauthenticated admin-escalation bug, an authenticated-non-owner PII leak, a rate-limiter bypass, an OAuth email-verification gap this session. Two parallel auth systems still coexist (architectural debt, not unsafe today) |
| Authorization / IDOR | YELLOW | Two independent security audits this session found and closed real issues. Neither exhaustively covered every endpoint — contracts/groups/quality domains not deeply re-checked |
| Redis / Celery workers | **RED** | Verified: Render hosts exactly 2 services (prod + dev API), zero worker/beat processes. The `beat_schedule` (job-source sync, trending-skills refresh, stale-match recompute) has nothing to execute it in production |
| AI — resume parsing | **RED** | Verified: `ResumeParserAgent` has zero call sites outside its own Celery task definition; zero `.delay()`/`.apply_async()` calls exist anywhere in the codebase. `upload_resume()` only stores the file. **No uploaded resume has ever been AI-parsed in production** — `parsed_resume_data` is dead plumbing right now |
| AI — job enrichment | YELLOW | `JobEnricherAgent` has a real call site in `jobs/service.py` (better shape than resume parsing) — not re-verified whether it's reached from the live scheduled-sync path |
| AI provider health (Groq/fallbacks) | UNKNOWN | Carried over from earlier this session: Groq flagged down in production, never re-verified since |
| Observability / monitoring | **RED** | No Sentry SDK anywhere in either app. structlog JSON logging exists (genuinely good — request-ID correlation, no secret logging observed) but goes nowhere except Render's log viewer — no alerting, no external aggregation. `prometheus-fastapi-instrumentator` wired in `main.py` but nothing external scrapes it |
| Admin platform | YELLOW | Real endpoints exist (stats/users/activity-logs, role-gated). Queue monitoring, AI cost tracking, feature-flag control, infra health — not implemented |
| Payments — escrow rail | GREEN | Real Stripe account, live mode, webhook registered and signature-verified |
| Payments — SaaS subscriptions/billing | **RED** | Does not exist at all: no plans, no entitlements, no billing cycle, no invoices |
| Email | GREEN basic / UNKNOWN branded | Resend transactional email confirmed working (real signup/confirmation flow tested). Custom-domain sender vs default — not checked |
| CI/CD + branch governance | GREEN | `prod`/`dev` branch-protected (renamed from `main`), a CI gate rejects any PR into prod not from dev, CodeQL + dependency-review + pre-existing GitGuardian all active |
| Secrets hygiene | GREEN | No live URLs/credentials in the public repo, this private repo holds topology + a credential *location* map (no raw values), GitHub secret scanning + push protection on |
| Dependency CVEs | YELLOW | 31 Dependabot alerts found (3 critical, 15 high) — patched, verified via full local test suite, merged into `dev` |
| Frontend test coverage | **RED** | No test framework installed in `apps/web` at all — zero unit/component tests, E2E only |
| Backend test coverage | GREEN | 205 tests passing, ruff + mypy clean |
| Frontend UX/IA coherence | UNKNOWN | Not audited this session — needs a dedicated pass |
| Accessibility (WCAG 2.2 AA) | UNKNOWN | No automated a11y checks in CI; not manually reviewed |
| Legal pages (privacy/terms/impressum) | **RED** | `/privacy`, `/terms`, `/impressum` all 404 on the live site. Impressum in particular is a real legal requirement for EU/German users — needs real legal review, not an AI-drafted page |

## 3. Confirmed findings worth calling out specifically

**The Celery/worker gap is real, not documentation drift.** Render has exactly two services, both API web services, zero workers. `celery_app.py`'s 3-cron `beat_schedule` has never fired in production. Job-source sync is separately covered by a GitHub Actions cron hitting an API endpoint directly — that part genuinely works. Trending-skills refresh and stale-match recompute do not run.

**AI resume parsing has no production dispatch path.** Grepped the entire backend for `.delay(` and `.apply_async(`: zero matches, repo-wide. `ResumeParserAgent` is referenced only in its own file and in the never-dispatched Celery task. `upload_resume()` stores the file and returns — nothing else.

## 4. What was fixed this session

- Unauthenticated `POST /auth/sync` allowing self-escalation to ADMIN — removed. Checked the live users table: only 3 expected ADMIN accounts, no evidence of exploitation.
- `GET /engineers/{id}` leaked full resume PII to any authenticated caller — scoped to owner/admin only.
- Rate limiter identifier let an attacker reset their own bucket via `User-Agent` rotation / spoofed `cf-connecting-ip` — fixed.
- Google OAuth didn't check `email_verified` before auto-linking accounts — hardened.
- 31 dependency CVEs (litellm, python-jose, pyjwt, python-multipart, pytest, nanoid, js-yaml) — patched, verified, merged.
- Live domain, real Stripe live-mode account, genuine dev environment with isolated DB/Auth, branch-protected `dev`/`prod` with enforced promotion flow, security headers on the frontend.

## 5. Priority list

| ID | Sev | Area | Finding |
|---|---|---|---|
| P0-1 | P0 | AI | Resume parsing never runs — wire a real dispatch path or stop advertising the feature until it does |
| P0-2 | P0 | Reliability | No Celery worker exists anywhere — decide deliberately: build one, or formally retire Celery in favor of the proven GitHub-Actions-cron pattern |
| P0-3 | P0 | Legal | No privacy/terms/impressum pages on a commercial EU-facing site — needs real legal review |
| P1-1 | P1 | Observability | Zero error monitoring, zero external alerting |
| P1-2 | P1 | Monetization | No SaaS subscription/billing/entitlement system exists |
| P1-3 | P1 | Authorization | Contracts, groups, quality domains not specifically re-audited for IDOR this session |
| P1-4 | P1 | Reliability | Render free-tier cold-start can produce a real 503 under real traffic |
| P2-1 | P2 | Architecture | Two parallel auth systems (Supabase-native + legacy custom-JWT) — consolidate |
| P2-2 | P2 | Testing | No frontend test framework at all |
| P2-3 | P2 | Environments | Dev shares prod's Redis/storage/email |
| P3-1 | P3 | UX | Frontend IA/navigation coherence not audited |
| P3-2 | P3 | Accessibility | No WCAG audit performed |

## 6. Explicitly not covered in this pass

Frontend UX/IA coherence, accessibility, full cross-domain IDOR sweep (contracts/groups/quality specifically), Groq/AI-provider live health, WebSocket authorization, load/performance testing, backup/restore verification.

## 7. Recommended order

1. Merge the open PR chain so Dependabot alerts actually close
2. Decide P0-2 (Celery: build it for real, or retire it) — blocks a clean answer on P0-1
3. P0-1: wire resume parsing to something that actually runs
4. P1-1: Sentry (in progress, waiting on DSNs)
5. P0-3: legal pages with real legal review, in parallel
6. Everything else in the P1/P2/P3 list
