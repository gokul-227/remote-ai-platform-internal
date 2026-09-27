# Continuation: go-live hardening

Last updated 2026-09-28. Read `evidence-index.md` (per phase) and `findings-register.md` (status tracker) first.

## Where things stand
- App repo: `prod` = `dev` = `a365ecc` (app code = `7dd0d0f`, 49 commits over `0786783`). CI, CodeQL, production
  deploy green; web and API report the SHA; dev API healthy after the Supabase restore.
- Local checks at release: API 434 passed (85.7 % coverage), web 51 unit, 35 E2E, cf:build, gitleaks clean.
- Migrations 037–040 applied in production.
- Secrets: Infisical (US) project `237deba5-…`; `prod:/render-api` syncs to Render (Secret Sync).
- Owner decision 2026-09-27: **not launching yet; stay 100 % free**; product quality is the next focus.

## Next
1. Product-quality pass (owner to list what feels unfinished).
2. DATA-03 Supabase storage/RLS review.
3. Owner items in `owner-checklist.md`.
