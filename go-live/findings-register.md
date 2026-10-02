# Go-live findings register

Started 2026-09-26 against app commit `6d16b77` (prod = dev) and internal commit `f4ab2c1`.
Companion files: `continuation.md` (where work stands), `progress.md` (Figma migration history).

**Evidence labels:** Verified live · Verified in source · Verified by test · Documented but unverified ·
Hypothesis · Blocked by access.
**Status labels:** Open · Implemented locally · Tested locally · CI verified · Deployed to dev ·
Production deployed · Production behavior verified · Blocked · Deferred and gated.

**Status 2026-09-27:** all fixes are on branch `fix/go-live-p0-containment` (31 commits), fully tested locally
(backend 352, frontend 29 unit, 29 Playwright journeys + axe scans). "Tested locally" below means verified
by those suites; all of it is **production deployed** at `0786783` (2026-09-27) and the checks in `continuation.md` were verified live.

## Baseline re-verified 2026-09-26 18:44 UTC

| Check | Result | Evidence |
|---|---|---|
| Backend `/health/version` | `git_sha 6d16b77…`, env production | Verified live |
| Backend `/health/ready` | HEALTHY; DB 1.7 s, Redis 0.6 s latency | Verified live |
| Backend `/health/dependencies` | DEGRADED: AI provider `(ollama)` DOWN, "All connection attempts failed" | Verified live |
| Frontend `/api/version` | reports `9389640…` (hardcoded fallback, see J) | Verified live |
| Backend tests (local, SQLite) | 318 passed at baseline → 330 passed on branch | Verified by test |
| Frontend unit tests | 2 at baseline → 12 on branch | Verified by test |
| Frontend `tsc` / `eslint` | clean / 0 errors. **`src/figma/*` carries `// @ts-nocheck` and is lint-excluded**, so both say little about the product UI | Verified in source |
| E2E (Playwright) | not run in this session (needs Docker infra) | — |

## Register

| ID | Sev | Finding | Evidence | Status | Commit |
|---|---|---|---|---|---|
| A | P0 | Legal pages were placeholders | Verified live | Tested locally: verified facts filled (contact@remoteaiplatform.com, Groq, Supabase EU/Render US/Cloudflare); operator identity shows "Coming soon" by owner decision. Legal review still advised | 12afc65, 4d2fad6 |
| B | P0 | "Release payment" set milestone PAID and said "Payment released" with no money moving | Verified in source | Tested locally | 3aea9e7, 62c01ad |
| C | P0 | No payout rail, yet escrow fund/capture was reachable and the UI said "the engineer receives these funds" | Verified in source | Tested locally: `MARKETPLACE_PAYMENTS_ENABLED` (default off) gates fund and release on both `/payments` and `/projects/*/payments`; refunds stay open | 3aea9e7, 62c01ad |
| D | P0 | Contract state not enforced: milestones changed on unsigned contracts, terms editable after signing, worker could add milestones, re-terminate | Verified in source | Tested locally (`contracts/lifecycle.py`, board sync uses the same rules) | 3aea9e7 |
| E | P0/P1 | Configured AI provider is down in prod (`AI_PROVIDER` default `ollama`, `OLLAMA_BASE_URL` default `localhost`) | Verified live | Tested locally: `AI_PROVIDER=auto` free-tier chain (Groq gpt-oss-120b first), per-user allowances. `GROQ_API_KEY` + `AI_PROVIDER=auto` set on Render prod and dev (2026-09-27) | 4e050c8, 9e8ea12 |
| F | P1 | Journey E2E tests are in `tests/e2e/legacy`, outside the active suite | Verified in source | Tested locally: 11 journeys + navigation/privacy/a11y specs run in CI against a hermetic stack; legacy suite deleted | 87d3322, 0daadfb |
| G | P1 | `src/figma/**` lint-excluded and `@ts-nocheck` | Verified in source | Tested locally: `@ts-nocheck` removed, figma linted (0 errors), dead mock data removed | b349d0a |
| H | P0 | `enforce_admins` off on prod/dev; 0 required approvals | Documented (progress.md) | Deferred by owner: restore `enforce_admins` after launch | — |
| I | P0 | Dependabot #41 sharp < 0.35.4 (GHSA-rgj7-g3m4-5g8c) via wrangler → miniflare | Verified in source | Tested locally: scoped override, clean `npm ci` + `cf:build` pass | 6f31a42 |
| J | P0 | Frontend version endpoint hardcoded SHA | Verified live | Tested locally (value inlined into built route bundle). Deploy workflow now asserts the live SHA | 5c4fc62 |
| K | P0 | Deploys race; backend hook deploys the branch tip, not the tested SHA | Verified in source | Implemented locally: concurrency group per env, tip gate, Render hook `?ref=<sha>` (documented; disables Render auto-deploy) | 5c4fc62 |
| L | P0 | Logout didn't end the Supabase session; logout-all ineffective for Supabase tokens | Verified in source | Tested locally (migration 033 `users.sessions_revoked_at`) | 0f6546d |
| M | P0 | Hidden profiles readable by ID; AI review/score/salary floor public | Verified in source | Tested locally | dda288a, 3814905 |
| M2 | P0 | **New.** `/matching/candidates/{job}` serialized the full private profile (resume URL, parsed resume) to companies and ignored `is_public` | Verified in source | Tested locally | dda288a |
| N | P1 | Deep links lose resource ID (hash routing + storage selections) | Documented | Tested locally: `#engineer/<id>` etc. + old path redirects | 1fd769a |
| O | P1 | Jobs: 50-row cap, non-engineering roles, mojibake, defaulted fields | Verified live (audit) | Tested locally: `X-Total-Count` pagination, all professions, mojibake repair, no invented type/level, explicit salary period (migration 035) | 724bf84, 5809f50 |
| P | P1 | Resume import: 15 s client timeout vs longer AI budget; AI overwrote user fields; "queued" message false | Verified in source | Tested locally: 90 s client timeout, parse budget, fills empty fields only, honest status incl. quota. Durable job queue not needed after Celery removal (parse is inline within budget) | aadb7f4 |
| Q | P1 | Observability: Workers logs/traces off, Sentry CSP, raw-path metric labels, `/metrics` 404 | Documented (audit) | Tested locally: route-template labels, Sentry CSP, Workers observability on, generic public health errors | a1150c9 |
| R | P1 | Env isolation (shared Redis), no restore rehearsal, Supabase controls unverified | Documented / Blocked by access | Partly: Redis keys namespaced per env (da9b8e2). Supabase auth settings verified by owner (email rate limit, redirect allow-list). Restore rehearsal still open | da9b8e2 |
| S | P0 | **New.** Fabricated claims in unreachable code ("48K+ verified engineers", "$212M paid via escrow", "SOC 2 aligned") | Verified in source | Removed | 62c01ad |
| T | P0 | **New.** "Prototype state" selector in production menu could replace any page with fake Loading/Error/Access-denied screens | Verified in source | Removed | 17dbae9 |
| U | P1 | **New.** Post-a-job "Fund milestones through escrow" checkbox was never sent to the API | Verified in source | Removed | 62c01ad |
| V | P2 | **New.** Wallet sums mixed currencies and labels them USD | Verified in source | Deferred and gated (final payments phase; wallet unused while payments are off) | — |
| W | P1 | **New.** Candidate matching scores only the first 30 public engineers | Verified in source | Tested locally: candidate pool of all open public profiles (cap 1000) | 4a3a102 |

| X | P0 | **New.** Two sign-in systems (Supabase + self-issued JWT/Keycloak) and known-password demo seed | Verified in source | Tested locally: Supabase-only (migration 036), seed removed | 210844a, 102b04b |
| Y | P1 | **New.** No self-service data export or account deletion | Verified in source | Tested locally: `GET /auth/me/export`, `DELETE /auth/me` + UI | a044ece |
| Z | P1 | **New.** Visitors bounced to sign-in by the 401 interceptor; nav showed every persona's screens | Verified by test | Tested locally | 5809f50, 87d3322 |

## Documentation corrections

Handbook 00–03, 05, 10–13 and AGENT-ONBOARDING were rewritten on 2026-09-27 to match the branch. The notes
below record what was wrong before.


- `AGENT-ONBOARDING.md` and `handbook/03` say "real Stripe payments live in production". The runtime
  `PAYMENT_PROVIDER` was **not verified**. Either way, no working funding path existed: the frontend never
  confirms a PaymentIntent, so a Stripe escrow could never reach ESCROWED. After this branch ships,
  money movement stays off until `MARKETPLACE_PAYMENTS_ENABLED` is deliberately set.
- `handbook/03` says Render auto-deploys on push and no Actions workflow deploys the backend. That is stale:
  deploy hooks are called from `deploy-frontend*.yml`, and with `?ref=` Render's auto-deploy is disabled.
- `handbook/03` says both branches block direct pushes even for admins. That is stale: see H.
- The recovered legal drafts said "password stored as a salted hash" and "hosted in the EU (eu-west-1)".
  The first is wrong for OTP/OAuth sign-in. The second is unverified. Neither was carried over.

## Audit 2026-09-27 (`Remote-AI-Platfrom-Go-Live.md`) — status tracker

IDs follow the audit. Evidence per phase: `evidence-index.md`. Status as of **2026-09-28**, production at `2ad2dcb`
(app code identical to `7dd0d0f`, deployed and verified live 2026-09-27). Owner = `gokul-227`.

| ID | Pri | Status | Where |
|---|---|---|---|
| SEC-01 | P0 | **Open — owner.** Keys pasted in chat (again 2026-09-27, incl. Stripe live). Owner rotates later; Infisical now the single source, Render fed by Secret Sync | owner-checklist |
| SEC-02 | P0 | **Open — owner decision.** `enforce_admins` still off; restore + add Secret scan/Analyze as required checks | owner-checklist |
| SEC-03 | P1 | Production verified | `cf4880f` |
| SEC-04 | P1 | Production verified (WS off, no access log) | `16254cb` |
| SEC-05 | P1 | Production verified (build refuses bad env) | `b69f033` |
| LEG-01 | P0 | **Deferred by owner** — Impressum shows "Coming soon" | `OPERATOR` in legal_content.ts |
| DATA-01 | P0 | Production verified; Supabase erasure key live via Infisical sync (admin API 200) | `202e8da` |
| DATA-02 | P0 | Production deployed | `cd71d90` |
| DATA-03 | P0 | **Implemented, CI pending (PR #50).** No migration ever enabled RLS/revoked `anon`/`authenticated`; reproduced on PG16 with Supabase-style grants (anon read `users`). Migration 042 + PG guard test. **Still owner:** anonymous live probe of dev/prod Data API + Storage (blocked for Claude), bucket MIME/size limits, confirm assets bucket content | `d29c328`, evidence 2026-09-28 |
| OPS-01 | P0 | Tooling rehearsed; **no production backup yet by owner decision** (no real users) | `e87bb0e`, backup-and-restore.md |
| REL-01 | P1 | **Closed** — probes bounded; dev Supabase restored, dev API redeployed and healthy (2026-09-27); keep-awake workflow | `bb67d23`, `2ad2dcb` |
| OBS-01 | P1 | Production deployed | Phase 10 |
| OBS-02 | P1 | Production verified | `f066282` |
| PERF-01 | P1 | Production deployed (pool, statement cache, list cache); region move deferred (free only) | `d4923d8` |
| API-01 | P0 | Production deployed | `36ff24e` |
| AI-01 | P1 | Production deployed | `ae65253` |
| UX-01 | P1 | Partly: formatted, typed where touched; 332 `any` warnings remain | ADR-004 |
| UX-02 | P1 | Not reproduced (Chromium); Safari/iOS check pending | — |
| UX-03 | P1 | Production deployed | `4f5f165`, `f00d92f` |
| SEO-01 | P1 | Production verified (job pages, 404, sitemap) | `86c8420` |
| SEO-02 | P1 | Production verified (dev Disallow) | `b69f033` |
| OPS-02 | P1 | **Verified**: Cloudflare Email Routing forwards contact@ (2026-09-27; Gmail hid a same-account test) | — |
| OPS-03 | P1 | Secret scan gates deploys; required-check setting still owner (SEC-02) | `720e398` |
| OPS-04 | P1 | Production deployed (037–040 applied) | `f887114` |
| OPS-05 | P1 | **Closed** — CodeQL on prod; 0 open alerts | `bb67d23` |
| COST-01/02 | P1 | Owner chose free-only until launch; two free Render services share 750 h | — |
| PAY-01 | P0-if-enabled | Gated off (verified) | — |
| PAY-02 | P2 | Production deployed | `0d923e3` |
| TRUST-01 | P1 | Blocking, moderation notices/appeals deployed | `ab132ff`, `8329834` |
| PROD-01 | P1 | Documented (one user = one org, no members) | launch-contract.md |
| GROW-01, ENT-01 | P2 | Deferred | — |

### Plan 2026-09-28 (`Remote-AI-Platform-GO-Live.md`) — new findings

| ID | Pri | Status | Evidence / where |
|---|---|---|---|
| DATA-04 | P0 | **Implemented, CI pending (PR #50).** Reproduced in source + test: refreshed old session re-created a deleted account after failed Supabase erasure; admin delete wrote no tombstone at all. All tokens for a tombstoned subject refused; durable pending erasure retried (sync + admin endpoint). Real-provider staging check still owner (AUTH-01) | `988a230`, migration 041 |
| AI-02 | P1 | **Implemented, CI pending.** Admission was `used >= cap`; now `used + reservation > cap` refuses. Overshoot bounded by (actual − estimate) per in-flight call; documented as token allowance, not a financial cap | `1aa0ef2` |
| UX-04 | P1 | **Implemented, CI pending.** Native button per job row, `aria-current`, visible focus, live region; unit tests (Tab/Enter/Space). Manual screen-reader check (VoiceOver/NVDA) still open | `6feee2e`, `321dd10` |
| OPS-06 | P1 | **Implemented + drilled locally.** Identity-based source refusal, empty DB/bucket preflight, dry run + CONFIRM_TARGET, no `--clean`, single transaction, outer checksum | `6926ef2` |
| OPS-07 | P1 | **Implemented + drilled locally.** Counts from dump snapshot (proven under concurrent writes: 67 = 67 while live went to 340); key path containment tests; metadata kept; dangling-reference report | `6926ef2` |
| CACHE-01 | P1 | **Measured + tested.** Live job pages `private, no-store`, OpenNext has no incremental cache, API `cf-cache-status: DYNAMIC`, one worker → moderation immediate. Guard test on `WEB_CONCURRENCY=1`. Sitemap may list a hidden job until its next render (≤ 1 h if cached) | `3c65a8c` |
| REL-01 | P1 | **Closed (re-verified 2026-09-28).** `a365ecc` all pipelines green incl. dev deploy; prod API + prod/dev web report `a365ecc`. Dev API cold (no reply in 60 s) during check | live |
| OPS-01 | P0 | **Open — owner.** Tooling now safe; still no scheduled independent encrypted backup | owner-checklist |
| AUTH-01 | P1 | **Open — owner.** Real OTP/OAuth/refresh/global sign-out on dev with designated accounts | — |
| OBS-03 | P1 | **Implemented, CI green (PR #50, `541364e`).** `GET /health/ingestion` (per-source last success, failures since, fetched; count of erasures pending > 24 h); canary fails on a source silent 24 h or a stuck erasure. **Still open:** proof that the failure email reaches the owner (alert receipt drill) | `541364e` |
| PRIV-01 | P1 | **Open — owner decision.** Keep-and-anonymise policy for signed contracts/payments | owner-checklist |
| SEC-02 | P0 | **Open — owner.** Re-verified 2026-09-28: prod/dev `enforce_admins=false`, 0 reviews, 4 checks; `Secret scan (gitleaks)`, `Analyze (python)`, `Analyze (javascript-typescript)` not required | — |

## Plan 2026-10-02 (`Remote-AI-Platform-ready-for-GO-Live.md`) — plan IDs

| ID | Sev | Status | Owner / next action |
|---|---|---|---|
| SEC-01 | P0 | **Closed (verified live)** — internal repo PRIVATE, 0 forks, no secrets in history | Residual: copies made while public |
| SEC-02 | P0 | **Open — owner.** Historical chat-exposed keys: confirm revocation in each provider (names in owner-checklist) | Owner |
| SEC-03 | P0 | **Closed 2026-10-02 (verified via API).** prod + dev `enforce_admins=true`, 7 required checks incl. gitleaks + CodeQL. Releases now go through PRs | Owner merges PRs (Claude's `gh pr merge` is classifier-blocked) |
| DATA-01 | P0 | **Closed (verified live 2026-10-02).** API stores verified type (`80ff1cd`); bucket limits applied prod + dev (resumes 5 MB PDF/DOCX, assets 2 MB images); anon Data API 42501, anon storage list empty, anon upload refused | — |
| OPS-01 | P0 | **Open — owner.** Free only (R2 rejected: needs payment method). Runner ready (private GitHub artifact, inert); needs Infisical → GitHub sync of 5 secrets to the internal repo, first run, restore drill | Owner |
| LEGAL-01 | P0 | **Open — owner.** Operator name/address/retention facts for Impressum/privacy | Owner (+ legal review) |
| PAY-01 | P0-if-enabled | Gated off; unchanged | — |
| ING-01, OBS-01, PERF-01, UX-01, SEO-01, PRIV-01, SEARCH-01, MATCH-01, AUTH-02, TEST-01 | P1 | **Production deployed `492f9e7`, verified live** (sitemap 3640 URLs, ingestion per configured source, salary 422, versions) | — |
| SEC-04A | P1 | Partial: CSV, JWT claims done; proxy IP (last XFF hop) + request-ID validation already present; nonce CSP deferred (staged change) | — |
| UX-02 | P1 | 331 lint warnings (−1); large modules not split (needs visual-parity baseline) | Deferred, gated by Phase 03 baseline |
| AUTH-01 | P1 | **Open — owner.** Real OTP/OAuth/refresh/global sign-out on dev with designated accounts | Owner |
| MAIL-01 | P1 | **Production deployed `492f9e7`**: durable outbox (043) for invitations + application status changes; PG concurrency proven. **Open — owner:** verify `remoteaiplatform.com` in Resend (sandbox sender reaches only the owner), then DNS + `EMAIL_FROM_ADDRESS` | Owner, then Claude |
| COST-01 | P1 | Open: quota measurements (Render hours, Workers requests, Supabase size/egress) | Owner dashboards |
| DOC-01 | P1 | Open: companion docs still describe some superseded behaviour | Claude (next pass) |
| DEP-01 | P0/P1 | **Fixed** `8a6cf68`: critical Next + PyJWT, high pypdf; npm audit 0. Remaining: litellm 1.84 → 1.88.6 (medium), one PyJWT medium with no fix | Track |
| ENT-01, GROW-01 | P2 | Deferred expansion | — |
| API-02 | P1 | **Production verified `7937e65`.** `get_db` committed after the response was sent (FastAPI 0.141 runs yield-dependency teardown post-response): read-your-writes race (E2E "Saved" flake on `b88b702`) and false 2xx on failed commit. Commit now on the per-call exit stack before the response; reproduced old order `[response, commit]` → `[commit, response]`; failed commit → 500 | Verify in CI/dev, then prod |
