# Required from you — step by step

Updated 2026-09-05 (evening). All 12 PRs from the audit phases are merged into `dev` and verified healthy (256 tests passing). Branch protection strict mode restored on `dev` for all future PRs.

## Done
- ✅ All 12 phase PRs merged
- ✅ `dev → prod` promotion PR opened: **#28** — https://github.com/gokul-227/remote-ai-platform/pull/28 (still needs your merge -- this is the one that actually ships everything live)
- ✅ Sentry: both projects created, DSNs wired in (Render env vars for backend prod+dev, GitHub secret + deploy workflow for frontend). PR **#29** wires the frontend deploy-workflow piece -- needs merge.
- ✅ Dev storage isolation: dev's own Supabase S3 keys set on Render, MINIO_ENDPOINT switched from prod's project to dev's own project ref
- ⏭️ Upstash Redis for dev: skipped -- free tier caps at 1 DB/account, upgrading needs a payment method (against the no-spend rule). Dev keeps sharing prod's Redis, low-risk per decision 0001.

## Still needed: merge 2 more PRs
1. **#29** — wires the Sentry frontend DSN into the deploy workflows (small, should be quick)
2. **#28** — the actual dev → prod promotion (the big one -- ships everything live, closes the remaining Dependabot alerts)

## Decisions pending your answer (just reply, no research needed)
1. **Auth system consolidation** (Supabase-native + legacy custom-JWT) -- "do it" or "later"?
2. **Accessibility contrast colors** (2 design tokens, ~180 usages, needs a brand-color call) -- "darken them" or "leave as-is"?
3. **Legal content** -- needs a real lawyer eventually; Impressum specifically needs your identity decision (personal name/address now vs. wait for company formation)
4. **Pricing/plans** -- architecture built (billing PR), zero real prices set. Whenever you're ready to define tiers/limits/prices.
