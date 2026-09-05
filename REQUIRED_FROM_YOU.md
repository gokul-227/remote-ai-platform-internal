# Required from you — step by step

Updated 2026-09-05. Do these in order — later steps depend on earlier ones landing.

## Step 1: Merge the 12 open PRs, in this exact order

Some of these share sequential database migrations, so the order below is not optional for #18/#20/#23. #26 and #27 must go first — both are critical CI/infra fixes that were blocking every other PR's checks (a broken Docker build for this monorepo layout, and a flaky npm bug), verified green in real CI, not just locally.

1. **#26** — Docker build fix (blocks E2E on every other PR)
2. **#27** — CI npm bug fix (blocks Frontend/E2E intermittently)
3. **#18** — Database integrity pass (indexes, migration-safety fix)
4. **#20** — Billing/entitlements architecture (its migration chains after #18's)
5. **#23** — Analytics funnel tracking (its migration chains after #20's)
6. **#19** — AI reliability (silent-failure fix, cost tracking, rate limiting)
7. **#21** — Nav fix (AI Quality Engine link for company users)
8. **#22** — Trust/reputation IDOR fix
9. **#25** — Performance fixes (N+1 queries, unbounded pagination)
10. **#24** — CI Node 20→22 bump
11. **#12** — Frontend test framework (Vitest) — confirmed fully green in real CI including E2E
12. **#15** — Accessibility audit (WCAG 2.2 AA)

After each merge, GitHub may show the next PR as briefly "out of date" — that's expected, just wait a minute for it to recheck, or hit "Update branch" if offered.

## Step 2: Promote dev to prod

Once all 8 are merged into `dev`, tell me (or open it yourself: PR from `dev` into `prod`). This is what actually ships everything live and closes the remaining Dependabot alerts for real (they're scanned against `prod`, not `dev`).

## Step 3: External accounts / manual dashboard steps

1. **Sentry** — create a free account at sentry.io, two projects (Python/FastAPI named `remote-ai-platform-api`, Next.js named `remote-ai-platform-web`), paste both DSN keys here.
2. **Supabase S3 keys for dev** (2 min) — dev Supabase project (ref `wqugjgtjjmixzsqqcmld`) → Settings → Storage → S3 Access Keys → New access key → paste both values here.
3. **Upstash Redis for dev** (optional) — upstash.com → new free Redis database → paste the `rediss://` URL here. Or explicitly tell me to skip this (dev sharing prod's Redis is low-risk).

## Step 4: Decisions only you can make

1. **Auth system consolidation** — two parallel auth systems exist (Supabase-native + legacy custom-JWT). I've deliberately held off touching this without your explicit go-ahead, since it's real surgery on login itself. Say the word when you want this tackled.
2. **Accessibility contrast colors** — PR #15 fixed 40/53 real WCAG violations but flagged 2 design-system color tokens (`--text-light`, `--text-muted`, used ~180× across 35 files) as needing a brand-color decision to fix the rest. Your call whether/how to adjust them.
3. **Legal content** — a real lawyer needs to draft privacy policy, terms, and Impressum. The Impressum specifically needs your decision: publish a real personal name/address now (German law requires this for any commercial site reachable from Germany, even run by an individual), or wait until company formation.
4. **Pricing/plans** — PR #20 built the billing architecture with zero real prices set. Whenever you're ready to define actual plan tiers/limits/prices, I can wire them into the existing entitlement system.

## Everything else

Full history of every finding, fix, and audit performed is in this repo's `reports/` and `decisions/` folders, and `docs/DEPLOYMENT_TOPOLOGY.md` for current live infrastructure.
