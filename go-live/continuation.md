# Continuation: go-live hardening

Last updated 2026-09-27. Read `findings-register.md` first; owner-only items are in `owner-checklist.md`.

## Where things stand

- App repo branch `fix/go-live-p0-containment`: 31 commits on top of `prod` (`6d16b77`).
- Local checks: backend 352 passed, ruff + mypy clean; frontend tsc, eslint (0 errors), 29 vitest,
  `next build` and `cf:build` pass; hermetic E2E (29 Playwright tests incl. WCAG AA scans) pass.
- Migrations `033`–`036` are additive or data-only except `036`, which drops password columns and the
  password_reset_tokens table (no real users exist; owner confirmed).
- Render prod + dev: `GROQ_API_KEY` and `AI_PROVIDER=auto` set. Infisical holds the same keys.

## Going live (owner authorised a direct push to `prod`)

1. Push the branch to `prod`, then fast-forward `dev`.
2. Watch CI (lint, types, unit, E2E) and the deploy workflows; the deploy only runs for the tested tip.
3. Verify `/api/version` (frontend) and `/health/version` (backend) report the pushed SHA.
4. Check `/health/dependencies`: AI chain HEALTHY via Groq; DB and Redis healthy; migrations at `036`.
5. Smoke: anonymous job feed with pagination, sign-in by email code, profile visibility, resume upload.

## Later

See `owner-checklist.md` (branch protection, key rotation, Impressum, Stripe plans, freelance payments).
