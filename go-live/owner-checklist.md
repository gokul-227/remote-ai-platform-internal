# Owner checklist

Things only the owner can do, or has chosen to do later. Last updated 2026-09-28.

## Done 2026-09-27
- [x] Supabase prod auth settings (earlier).
- [x] contact@remoteaiplatform.com forwarding works (Cloudflare Email Routing).
- [x] Infisical is the single secret store; Render production API fed by Infisical Secret Sync (`prod:/render-api`).
- [x] `JOB_SYNC_TOKEN` in Infisical, Render and GitHub; `PROD_ADMIN_EMAIL/PASSWORD` and unused
      `SUPABASE_SERVICE_ROLE_KEY` removed from GitHub secrets.
- [x] Dev Supabase project restored; dev API healthy.

## Later (owner's choice)
- [ ] **Rotate every key pasted into chat** (Stripe live first; Groq, Resend, Render, Cloudflare, Supabase secret key;
      delete unused GitHub/Figma/Supabase access tokens and the pasted Infisical client secret). New values go into
      Infisical; press "Deploy latest commit" on Render after the sync.
- [ ] Remove the old `SUPABASE_ACCESS_TOKEN` line from `~/.zshrc` (revoked token; breaks the Supabase CLI).
- [ ] Clean obsolete Infisical names: `JWT_SECRET_KEY`, `KEYCLOAK_CLIENT_SECRET`, `CELERY_*`, `AUTH_PROVIDER`,
      `SEED_DEMO_DATA`; root `SUPABASE_SERVICE_ROLE_KEY` is stale (use `SUPABASE_SECRET_KEY`).
- [ ] Demote the ci-service account (ADMIN role no longer needed by the sync).
- [ ] Branch protection: restore `enforce_admins`; add "Secret scan (gitleaks)" and "Analyze" as required checks.
- [ ] Impressum/legal facts in `OPERATOR` (shows "Coming soon"); legal review.
- [ ] Before real users: daily backups on (free, `ops/backup`); decide public visibility (A: sign-in only / B: public
      jobs); consider Render Frankfurt; Sentry alert rules; Safari/iOS + screen-reader pass.
- [ ] Before any paid feature: see Phase 13 paid-launch gate.
