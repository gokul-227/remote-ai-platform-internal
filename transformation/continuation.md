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

### 16 November 2026 resumed release validation

Active integration remains `rap-runtime-observability`; dev `52db47d` includes homepage
narrow-grid correction and native hash auth anchors. CI at `67dee38` had four E2E
failures: old homepage content expectations and 49px overflow at 320px. Following the
updated signup/back journey also exposed Next Link hash navigation missing AppEntry's
hashchange listener; native anchors fixed it. Five focused local discovery/accessibility
journeys passed, web236 unit tests passed, web TypeScript and focused ESLint passed.
Admin820px regression rerun passed with final output-directory config. Nine deployment
regression tests passed. Full final CI43862858700 is still running; no promotion yet.

Capacity HTTP smoke is now collected: 31 fixture jobs traversed as20+11 with no overlap;
3 k6 iterations,13/13 checks,7requests,zero failures. This is bounded loopback evidence,
not10M-job/1M-member capacity acceptance. Evidence saved in product tests/load/evidence.

Nightly backup16Nov02:36UTC succeeded (103MB). A fresh isolated VM restore into new
PostgreSQL17 and local MinIO containers verified99table row counts,5objects with matching
checksums/metadata and no missing references in34sec; report201 recorded. Scratch containers
and volumes removed by trap; production DB/storage untouched. This verifies the current
production schema backup, not yet the pending076 production migration.

GitHub's five dependency alerts are all in the separate docs-tool lock, not current app
production locks: fast-uri3.1.8 and legacy Faker5.5.3. A bounded agent is preparing compatible
converter updates and tests before final release. No secret values were printed.

Release follow-up: product HEAD `885aa7d` pushed to dev and integration branch. Added
missing isolated job-cache architecture component and regenerated YAML/draw.io/PNGs;
21 architecture tests and drift check passed. Docs converter locks now use fast-uri3.1.9,
Faker10.5.0 and nested YAML1.10.3; local audit zero vulnerabilities,6converter tests passed,
1API-export test skipped locally for absent worktree venv (separate export drift check passed).
All386 collection requests/hash remain unchanged; no Postman publishing during local tests.
Final workflows: CI43863645539, Architecture43863645573, Postman43863645491.

Production preparation:84,797 estimated job rows; schema075_job_source_boards.
Prebuilt16pending075_scale_read_models indexes CONCURRENTLY with3slock timeout/120sstatement
bound per command. All16pg_index.indisvalid=true. No table data/schema revision changed;
old application stays live and ready. This prevents the normal migrator building these
indexes under a write-blocking transaction. Production still068d41b pending exactSHAchecks.

Jooble live evidence: sourceJOOBLE enabled/permission true, latest09:17UTCenqueued run
SUCCEEDED09:18:29UTCwith2saved jobs (prior06:18run saved1). No manual import was triggered.
