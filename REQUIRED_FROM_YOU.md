# Required from you — step by step

Updated 2026-09-06. Big batch of background work just kicked off: PR #31 conflict fix,
repo hygiene (docs/secrets), and a 6-way security audit. Nothing needs you right now —
this section will update as each piece lands with PR links to review/merge.

## Already done
- ✅ PR #32 (Facebook-blue rebrand) — merged to dev, then dev→prod (#28) — **both live in production now**
- ✅ GitHub OAuth wired into both Supabase projects (prod + dev) using the Client ID/Secret you provided
- ✅ Resend domain verification in progress (DNS records added by you) — email rate-limit response (429, not 500) suggests the underlying send may now work; will confirm for real once PR #31's E2E reruns

## In progress — 7 parallel background agents
1. **PR #31 rebase fix** — it had a merge conflict against `dev` after the rebrand PR landed (both touched the login/register pages). An agent is resolving it correctly (colors + a step-count logic difference) and re-verifying.
2. ✅ **Repo hygiene — done.** PR **#33** → https://github.com/gokul-227/remote-ai-platform/pull/33 (targets `dev`, needs your merge). 140 doc/screenshot files moved here into `remote-ai-platform-docs/`. Public repo's root `CLAUDE.md` trimmed to a ~65-line bootstrap; `README.md` kept (normal project readme) with dead doc links fixed. Secret scan (gitleaks, full history + targeted grep): only two low-stakes findings, both fixed in the PR — (a) `apps/web/.env.example` had a real Supabase project URL + a real publishable key hardcoded (now placeholders; rotation not needed, publishable keys are meant to be public and this one's already in the live site's JS bundle anyway), (b) two obviously-fake test JWT secrets in CI config/docker-compose (no rotation needed). No Stripe/AWS/private-key/DB-password leaks found anywhere in history. Screenshots spot-checked — only fake demo data visible, nothing real.
3. **Security audit — Auth/AuthZ/IDOR/WebSockets** — running
4. **Security audit — File upload / AI-LLM / PII** — running
5. ✅ **Security audit — CORS/headers/API-docs/rate-limiting — done.** PR **#36** → https://github.com/gokul-227/remote-ai-platform/pull/36 (targets `dev`, needs your merge). Three MEDIUM fixes: (a) `/docs`/`/redoc`/`/openapi.json` had no production guard at all — now 404 in prod; (b) API responses had zero security headers — added (X-Content-Type-Options, X-Frame-Options, Referrer-Policy, Permissions-Policy, HSTS); (c) a few AI-cost endpoints (resume AI-enhance, project AI reports) were falling into the loose general rate-limit tier instead of the strict AI tier — fixed. CORS and CSRF reviewed and confirmed already safe (not just assumed — traced through Starlette's actual behavior and confirmed no cookie-based auth exists anywhere). Confirms the same test flakiness seen in PR #35's report is pre-existing Docker/test-env flakiness, not a real bug — same two tests, reproduces identically with or without changes, clears on container restart.
6. ✅ **Security audit — Payments/DB/Supabase/Redis/MinIO — done.** PR **#35** → https://github.com/gokul-227/remote-ai-platform/pull/35 (targets `dev`, **review this one personally before merging — it touches real payment logic**). Two real HIGH fixes: (a) a replayed/out-of-order Stripe webhook could revert an already-`RELEASED`/`REFUNDED` payment backward — added an event-dedup table + a guard that refuses to move terminal-status payments backward; (b) resumes were being served via a permanent public URL requiring a public-read bucket with no expiration (the presigned-URL helper existed but was never actually called) — now every resume access generates a fresh 15-minute presigned URL after the existing owner/admin check. Raw SQL, Supabase key separation, and Redis contents all reviewed clean. Flagged as unverified-from-source (needs you to check live): the actual Supabase Storage bucket ACL, production DB role privileges, and live Redis auth config. **Also noted**: 1 pre-existing test failure on `dev`, unrelated to this PR (confirmed present before these changes) — needs a look separately, not blocking.
7. ✅ **Security audit — Injection/SSRF/XSS/path-traversal/error-handling — done.** PR **#34** → https://github.com/gokul-227/remote-ai-platform/pull/34 (targets `dev`, needs your merge). One real MEDIUM fix: an auth dependency was leaking raw database error text (e.g. SQL error details) into the 401 response body if a DB error happened mid-auth-check — now returns a generic message to the client while still logging full detail server-side. Everything else audited (SSRF, XSS, command/code injection, path traversal) came back clean — no real issues found, confirmed via actual code search rather than assumed. 214 backend tests pass including 2 new regression tests.
8. **Security audit — CI/CD, GitHub Actions, dependencies, prod config** — running

Each of these opens its own PR against `dev` (not merged by any agent — full review required). Expect these to take anywhere from 20 minutes to over an hour each given the depth requested. A consolidated security report will be written here once they've all landed, following the report structure you specified (executive summary, per-finding severity/scenario/fix, VERIFIED/LIKELY SAFE/UNVERIFIED distinctions, manual-action checklist). No secret values will ever appear in any of these PRs or reports — only classifications and locations.

## What's still needed from you (unblocked, can do anytime)
1. Keep an eye on Resend — once it shows "Verified", say so and the OTP flow gets its final confirmation.
2. As each security-audit PR lands, you'll need to review and merge it yourself (payments-related PR in particular needs your own eyes before merging, given real money is involved).
3. Once all of this lands and merges, we'll do a full manual smoke test of dev + prod together and I'll fix anything that surfaces.

## Longer-standing decisions still pending your answer
1. **Auth system consolidation** (Supabase-native + legacy custom-JWT `service.py` path) — "do it" or "later"?
2. **Legal content identity fields** — Impressum needs your real name/address (or wait for a company) before lawyer review.
3. **Pricing/plans** — architecture built, zero real prices set yet.
4. **Repo visibility** — one of the audit agents will report on whether anything depends on the repo staying public; recommendation will be to go private once confirmed safe.
