# Continuation: go-live hardening

Last updated 2026-09-28 (second session: plan `Remote-AI-Platform-GO-Live.md`). Read `evidence-index.md` (per phase) and `findings-register.md` (status tracker) first.

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

## 2026-09-28 session 2 — PR #50 (`fix/go-live-p0-p1` → `dev`), CI green, not merged
Fixes DATA-04, DATA-03 (migration 042), AI-02, UX-04, OPS-06/07, CACHE-01 test. Migrations 041 and 042 are additive.
Evidence: `evidence-index.md` (last section); status: `findings-register.md` ("Plan 2026-09-28").

Owner actions, in order:
1. Review and merge PR #50 into `dev` (Claude is not allowed to merge unreviewed). Then check the dev deploy:
   `/health/version` = merge SHA, `alembic` head `042_lock_supabase_data_api`, sign-in works on dev.
2. DATA-03 live proof on **dev**, then prod: with the publishable key from the site bundle,
   `GET <supabase>/rest/v1/users?select=id&limit=1` must be 401/403 or an empty/permission error, not rows.
   Check Storage too: listing the resume bucket anonymously must fail. Set MIME/size limits on both buckets
   (resumes: pdf/docx, 10 MB), and check that the assets bucket holds only intended public files.
3. AUTH-01 on dev with a designated test account: OTP, OAuth, refresh, sign-out everywhere, and account deletion.
   After deletion the old session must get 401, and signing up again must make a fresh account.
4. Promote `dev` → `prod` when 1–3 pass (production migrations 041/042 run at startup).
5. SEC-02: turn on `enforce_admins` and add `Secret scan (gitleaks)`, `Analyze (python)` and
   `Analyze (javascript-typescript)` as required checks on `prod` and `dev`. After that every change goes
   through a PR (0 reviews is fine for a solo owner). Emergency access: temporarily disable, record why, re-enable.
6. OPS-01: schedule `ops/backup/backup.sh` from a runner outside Supabase/Render/Cloudflare to a separate
   encrypted destination. Keep the passphrase apart from the archives, then do a timed restore drill with DRY_RUN/CONFIRM_TARGET.
7. SEC-01 rotation, LEG-01 operator details and PRIV-01 retention policy are unchanged (owner).
