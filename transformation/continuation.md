# Continuation record — transformation program (updated 14 Nov 2026, evening)

Read this first in any new session. Owner decisions: memory `transformation-program-2026-11` (card-OK free tiers; owner creates accounts at the end with a guide from Claude; direct push to dev and prod; include work from the parallel Codex/Claude session; Upstash for production Redis; Files app at `#files` with Office inside).

```text
STATE (15 Nov 2026, evening): PRODUCTION RUNS ON THE ORACLE VM
release: cc2c847 (+ a019cca backup-report fix) on branch integrate/all; pushed to dev and prod
VM: 79.76.118.213 (opc, ~/.ssh/oci_rap), compose project rap-prod: caddy, web, admin, api, worker,
    scheduler, postgres 17, redis; ops project rap-ops: GlitchTip, Uptime Kuma, Slack relay
Env: /opt/rap/.env rendered by ops/vm/render-env.sh from Infisical prod /render-api + /vm
Data: Supabase -> VM verified (96 tables, 53,282 rows; 5 files to OCI rap-prod-resumes)
DNS: Cloudflare A records (proxied) -> VM; SSL Full (strict); Render custom domain removed
Fallback: Render service suspended (resume + point api back) until ~22 Nov; workers detached, not deleted
Backups: rap-backup.timer 02:39 UTC -> OCI rap-prod-backups, reports to API (#backups)

OPEN:
- WorkOS production needs owner's card -> then import 40 real accounts (runbook 05-*), set
  AUTH_PRIMARY_PROVIDER=workos, NEXT_PUBLIC_AUTH_PROVIDER=workos (repo var AUTH_PROVIDER), then delete Supabase
- After fallback week: delete Render, Upstash, Cloudflare workers, Sentry project
- Owner: Google People API verification (08-*), LinkedIn app (09-*), Slack incoming webhook for UptimeRobot
- GlitchTip public route for browser errors; restore branch protection; rotate keys pasted in chat
```

## Latest checkpoint — 14 November 2026

Product commit `f1d39f4` saves all integrated work from the parallel implementation round.
Includes contract workspace activation/completion gates, demo registry and workspace Demos tab,
group moderation/private requests, channel threads and Files attachments for channels/DMs,
bounded Office text previews, reviewer-email selection, assistant permission tools, incidents,
and process-owned worker/scheduler monitoring. Migrations 063 and 064 follow 061 (062 unused).

Verification: backend 771 passed / 11 skipped; fresh PostgreSQL migration upgrade, downgrade to061,
and re-upgrade passed; PostgreSQL concurrency/RLS 10 passed; web unit120 passed; admin unit5 passed;
focused Chromium files/channel/feed journeys3 passed; backend Ruff/mypy and frontend type checks passed.
Docs regenerated and docs-check passed before the final save. Final rebuilt browser suite and remote
CI for this new commit have not yet been observed. Production was not promoted in this round.

Remaining: full browser/reconnect validation on final images, remote CI, provider provisioning and
verified backup restore/cut-over, plus exact phase acceptance gaps. See owner-provisioning-handoff.md.
Do not claim the whole implementation complete or mark infrastructure phases complete from local tests.

## Urgent save checkpoint

Product `275a4bc` pushed to dev and feature branch. Includes `62906aa` trash content-access fix,
meeting passcode/waiting-room implementation (migration065), staff inventory wiring and backup-evidence
status, and browser sign-in hydration wait. Inventory tests4 passed. Meeting final checks, migration065
round-trip, final browser reruns and remote CI still need review. Earlier full browser run failed the
serial messaging journey with a signed-out page; hydration helper adjusted, rerun results unobserved.
Current npm audit returned zero vulnerabilities. Production unchanged. Preserve and inspect all files
before continuing; do not claim the full specification completed.

## 15 November late release-save checkpoint

ACTIVE WORKTREE `/Users/gokulr/Developer/GitHub/rap-runtime-observability`, branch
`fix/vm-runtime-observability`, dev HEAD `b262715`. Do NOT resume in the older main
checkout and overwrite newer production history. Production remains068d41b.

Recovered Claude's saved legal/SEO/scale branches and integrated all into dev. Full combined
backend1038 passed16 skipped; web236 passed; filter integration58 passed. Migration076 merges
075 source boards and scale read models; fresh PostgreSQL upgrade/downgrade074/re-upgrade passed.

Found/fixed rollout blockers: explicit Bash failure propagation, rollback API command override
bypasses old Alembic (old image cannot resolve migration076), legacy unknown API SHA allowed only
in rollback, starts isolated job-cache. Nine deployment regression tests passed. Commit346c4d8.

Fresh backup initially failed: backup.env targeted old Supabase while primary runtime uses Oracle
via STORAGE_PROVIDER=r2. Corrected backup source from live primary settings securely, preserving
root-only /opt/rap/backup.env.before-storage-fix. Backup succeeded22:18UTC:98MB,99 tables,5 objects,
offsite upload to rap-prod-backups and evidence report201. No secrets printed.

Commit b262715 urgently saves capacity-tool fixes and admin tablet regression tests. Final agent
validation reports pending at save time. Run/inspect those tests before treating them as accepted.
Latest dev CI still running; older db2a7d2 frontend/migrations/backend passed, E2E pending.
Production NOT promoted. Do not delete fallback providers/credentials. Search Console credentials
absent and SEO weekly report disabled (owner gate); Adzuna permission false. Jooble key present.

Next: confirm clean trees; finish responsive/capacity tests; inspect latest CI. On green tested SHA,
promote dev->prod as authorized; observe deploy-vm build and health gate. Verify API/web exactSHA,
worker/scheduler fresh heartbeat, legal/hubs/sitemaps/cache, migration076 and job-cache live.
See 2026-11-15-final-action-audit.md and 2026-11-15-config-retirement-register.md for remaining
owner gates and cleanup dependencies. Never run full synthetic-account mutation E2E on production.

Final agent reports: admin820px regression passed, TS/unit10/build passed; old84px overflow not
reproduced, no CSS fix claimed. Last output-path test-config adjustment not rerun. Capacity tools:
3 regression tests, shell syntax, isolated PG17 31-job/1-member seed/query benchmark passed;
injected benchmark error exited1. HTTP smoke rerun result uncollected. Local capacity API and
rap-capacity-smoke-pg remain running. Capacity docs updated and committed after urgent save.
