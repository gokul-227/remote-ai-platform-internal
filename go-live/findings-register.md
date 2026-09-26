# Go-live findings register

Started 2026-09-26 against app commit `6d16b77` (prod = dev) and internal commit `f4ab2c1`.
Companion files: `continuation.md` (where work stands), `progress.md` (Figma migration history).

**Evidence labels:** Verified live · Verified in source · Verified by test · Documented but unverified ·
Hypothesis · Blocked by access.
**Status labels:** Open · Implemented locally · Tested locally · CI verified · Deployed to dev ·
Production deployed · Production behavior verified · Blocked · Deferred and gated.

Nothing below is deployed yet. All fixes are on branch `fix/go-live-p0-containment` (not pushed).

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
| A | P0 | Legal pages were placeholders | Verified live | **Blocked**: content written from recovered drafts and matched to the product; operator facts render as "Pending: …" until the owner supplies them in `legal_content.ts` `OPERATOR`. Needs legal review | 12afc65 |
| B | P0 | "Release payment" set milestone PAID and said "Payment released" with no money moving | Verified in source | Tested locally | 3aea9e7, 62c01ad |
| C | P0 | No payout rail, yet escrow fund/capture was reachable and the UI said "the engineer receives these funds" | Verified in source | Tested locally: `MARKETPLACE_PAYMENTS_ENABLED` (default off) gates fund and release on both `/payments` and `/projects/*/payments`; refunds stay open | 3aea9e7, 62c01ad |
| D | P0 | Contract state not enforced: milestones changed on unsigned contracts, terms editable after signing, worker could add milestones, re-terminate | Verified in source | Tested locally (`contracts/lifecycle.py`, board sync uses the same rules) | 3aea9e7 |
| E | P0/P1 | Configured AI provider is down in prod (`AI_PROVIDER` default `ollama`, `OLLAMA_BASE_URL` default `localhost`) | Verified live | **Blocked by access**: needs Render env inspection. Resume upload now degrades honestly (P) | — |
| F | P1 | Journey E2E tests are in `tests/e2e/legacy`, outside the active suite | Verified in source | Open | — |
| G | P1 | `src/figma/**` lint-excluded and `@ts-nocheck` | Verified in source | Open. Dead mock data found (`rap_kit` PEOPLE/JOBS/COMPANIES, `rap_home` CANDS, `rap_jobs` EXP, `rap_pages` FINDINGS, unreachable `rap_enterprise` screens) | — |
| H | P0 | `enforce_admins` off on prod/dev; 0 required approvals | Documented (progress.md) | **Needs owner action** (repo settings) | — |
| I | P0 | Dependabot #41 sharp < 0.35.4 (GHSA-rgj7-g3m4-5g8c) via wrangler → miniflare | Verified in source | Tested locally: scoped override, clean `npm ci` + `cf:build` pass | 6f31a42 |
| J | P0 | Frontend version endpoint hardcoded SHA | Verified live | Tested locally (value inlined into built route bundle). Deploy workflow now asserts the live SHA | 5c4fc62 |
| K | P0 | Deploys race; backend hook deploys the branch tip, not the tested SHA | Verified in source | Implemented locally: concurrency group per env, tip gate, Render hook `?ref=<sha>` (documented; disables Render auto-deploy) | 5c4fc62 |
| L | P0 | Logout didn't end the Supabase session; logout-all ineffective for Supabase tokens | Verified in source | Tested locally (migration 033 `users.sessions_revoked_at`) | 0f6546d |
| M | P0 | Hidden profiles readable by ID; AI review/score/salary floor public | Verified in source | Tested locally | dda288a, 3814905 |
| M2 | P0 | **New.** `/matching/candidates/{job}` serialized the full private profile (resume URL, parsed resume) to companies and ignored `is_public` | Verified in source | Tested locally | dda288a |
| N | P1 | Deep links lose resource ID (hash routing + storage selections) | Documented | Open | — |
| O | P1 | Jobs: 50-row cap, non-engineering roles, mojibake, defaulted fields | Verified live (audit) | Open | — |
| P | P1 | Resume import: 15 s client timeout vs longer AI budget; AI overwrote user fields; "queued" message false | Verified in source | Containment tested locally; durable job + draft model still open | aadb7f4 |
| Q | P1 | Observability: Workers logs/traces off, Sentry CSP, raw-path metric labels, `/metrics` 404 | Documented (audit) | Open | — |
| R | P1 | Env isolation (shared Redis), no restore rehearsal, Supabase controls unverified | Documented / Blocked by access | Open | — |
| S | P0 | **New.** Fabricated claims in unreachable code ("48K+ verified engineers", "$212M paid via escrow", "SOC 2 aligned") | Verified in source | Removed | 62c01ad |
| T | P0 | **New.** "Prototype state" selector in production menu could replace any page with fake Loading/Error/Access-denied screens | Verified in source | Removed | 17dbae9 |
| U | P1 | **New.** Post-a-job "Fund milestones through escrow" checkbox was never sent to the API | Verified in source | Removed | 62c01ad |
| V | P2 | **New.** Wallet sums mixed currencies and labels them USD | Verified in source | Open (Phase 8; payments gated meanwhile) | — |
| W | P1 | **New.** Candidate matching scores only the first 30 public engineers | Verified in source | Open (Phase 3) | — |

## Documentation corrections

- `AGENT-ONBOARDING.md` and `handbook/03` say "real Stripe payments live in production". The runtime
  `PAYMENT_PROVIDER` was **not verified**. Either way, no working funding path existed: the frontend never
  confirms a PaymentIntent, so a Stripe escrow could never reach ESCROWED. After this branch ships,
  money movement stays off until `MARKETPLACE_PAYMENTS_ENABLED` is deliberately set.
- `handbook/03` says Render auto-deploys on push and no Actions workflow deploys the backend. That is stale:
  deploy hooks are called from `deploy-frontend*.yml`, and with `?ref=` Render's auto-deploy is disabled.
- `handbook/03` says both branches block direct pushes even for admins. That is stale: see H.
- The recovered legal drafts said "password stored as a salted hash" and "hosted in the EU (eu-west-1)".
  The first is wrong for OTP/OAuth sign-in. The second is unverified. Neither was carried over.
