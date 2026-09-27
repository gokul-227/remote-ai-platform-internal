# Backup and restore runbook

Tooling: `remote-ai-platform/ops/backup/` (README there has the exact commands). Rehearsed 2026-09-27 against the
E2E stack; production has **not** been backed up yet (needs the owner's credentials and a destination).

## Proposed targets (owner to accept)

- RPO ≤ 24 h: one backup a day. RTO ≤ 4 h: restore + DNS/env switch. Not a contractual SLA.
- Keep 7 daily + 4 weekly archives, **outside** Supabase/Render/Cloudflare (e.g. the owner's machine plus a separate
  cloud drive). Do not use GitHub Actions artifacts: the app repository is public.

## Take a backup (owner)

1. Supabase → Project → Connect: session-mode connection string → `SOURCE_DATABASE_URL` (libpq form).
2. Supabase → Storage → S3 access keys → `S3_ENDPOINT`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`; buckets
   `remote-ai-platform-resumes,remote-ai-platform-assets`.
3. `PG_IMAGE=postgres:<Supabase major>-alpine` (Settings → Infrastructure shows the version).
4. Passphrase from the password manager (never stored with archives). Run `backup.sh`; keep the `.sha256`.

## Restore (drill monthly; for real only after an incident decision)

1. Create an **empty** target: new Supabase project or disposable PostgreSQL ≥ the dump's major version + buckets.
2. Run `restore.sh`. It stops on any checksum, row-count or object mismatch.
3. New Supabase Auth project? add `RESET_IDENTITIES=1`; people re-attach on their next verified sign-in. Tell users
   they may need to sign in again.
4. Point Render `DATABASE_URL`/storage env at the target, redeploy, check `/health/version`, `/health/ready`, sign-in,
   job list, a resume download.
5. Never restore over a database that has taken new writes without a written loss/reconciliation decision.

## Rehearsal record

| Date | Source | Target | Tables / objects | Backup | Restore+verify | Result |
|---|---|---|---|---|---|---|
| 2026-09-27 | E2E stack PG16 + MinIO | fresh PG16 + MinIO | 50 / 18 | 1 s | 2 s | pass; restored DB at head 040; identity reset path verified (164 users) |
| 2026-09-27 | pg_dump 18 → PG16 target | — | — | — | — | correctly refused (exit 3): use `PG_IMAGE` matching the server |
