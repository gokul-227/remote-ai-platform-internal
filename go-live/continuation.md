# Continuation: go-live hardening

Last updated 2026-09-26. Read `findings-register.md` first.

## Where things stand

- App repo branch: `fix/go-live-p0-containment`, 10 commits on top of `6d16b77`. **Not pushed, no PR.**
- Local checks on the branch: backend 330 passed (SQLite), ruff + mypy clean; frontend `tsc`, 12 vitest
  tests, `eslint` (0 errors), `next build` and `cf:build` all pass. E2E not run.
- Migration: `033_user_sessions_revoked_at` (nullable, additive) can run before or with the code.
- Config added: `MARKETPLACE_PAYMENTS_ENABLED` (default `False`), `RESUME_PARSE_BUDGET_SECONDS` (45).

## Before this branch reaches production

1. Push, open PR → `dev`, and let CI run (it runs the E2E job with Docker services).
2. On dev: sign out and back in, then use sign-out-everywhere across two browsers; check a hidden profile
   returns 404 to another account; walk a contract from offer to approval; upload a resume with the AI
   provider down; confirm that `/api/version` on dev reports the merged SHA.
3. Before the first prod deploy with `?ref=`, confirm in Render that the service follows deploy hooks
   (Render disables auto-deploy when `ref` is used).

## Decisions waiting on the owner

- **Legal (A):** every `OPERATOR` field in `apps/web/src/figma/legal_content.ts` (legal name, address, a
  monitored contact inbox, VAT/register, content responsible, AI providers and training terms, cache
  provider, data locations, log retention, governing law, supervisory authority), plus legal review.
- **AI provider (E):** which provider/model production should use. Today it is the `ollama` default
  against localhost.
- **Branch protection (H):** restore `enforce_admins` on `prod` and `dev` (commands in `progress.md`). As
  a solo maintainer, keep required CI checks and set required approvals to 0; review happens through the
  PR itself.
- **Profile projection (M):** confirm that `hourly_rate` stays public and `desired_salary_min` stays private.
- **Payments:** confirm the actual `PAYMENT_PROVIDER` in prod, and whether any ESCROWED or AUTHORIZED
  records exist that need refunds.

## Next engineering steps (in order)

1. G: bring `src/figma` under ESLint (start with `no-unused-vars`), delete verified-dead mock screens and data.
2. F: port the legacy E2E journeys (engineer, company, authorization, persistence) onto the Figma UI.
3. N: path-based resource URLs (`/jobs/{id}`, `/engineers/{id}`) that don't depend on storage selections.
4. O / W: jobs pagination and data quality; score candidates beyond the first 30.
5. P / E: durable resume-parse job and a draft/commit model for AI fields.
6. Q / R: observability, and a backup restore rehearsal.
