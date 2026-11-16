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

Populated rehearsal: restored the16Novarchive again into isolated containers, verified99tables
and5objects, then mounted candidate885aa7d API/migrations over the old production image for
Alembic. Upgrade from075_job_source_boards through075_scale_read_models to076 succeeded;
restored job count remained69,916 (the backup snapshot, older than live84,797estimate).
The old068d41b API started directly via uvicorn (bypassing its incompatible old migrator)
and GET /api/v1/jobs?limit=2 returned200/twojobs when called inside its container. The
initial host-published-port harness did not connect on the internal Docker network;
inside-container read is the compatibility evidence, not the failing host-port probe.
This is a bounded read rollback check, not full authenticated write compatibility.

Candidate885aa7d multiarch build-only workflow43864101523 succeeded; no deployment by that
run. Architecture, Postman and CodeQL are green for885aa7d; fullCI43863645539 pending.

Before release: discovered JOOBLE's persisted schedule still17 */3 * * * despite the
candidate's documented daily default. Updated exactly that old value to0 6 * * * via guarded
SQL; one row changed. Source enabled/permission unchanged; no import manually triggered.
Fresh pre-release backup16Nov12:20:12UTC succeeded25s:111MB,99tables,5objects,offsite upload
and report201; archive remote-ai-platform-20261116T122012Z.tar.enc. Scratch drill containers
are gone; Redis database15 used for the API probe was inspected empty (DBSIZE0).

CI43862858700 at52db47d completed successfully:1020backend/34skipped,21repo-level,
10PostgreSQL,108browser/8skipped and5WorkOS journeys. Final885aa7d CI43863645539 has passed
backend/frontend/migration/secret scan; final browser suite started12:13UTC and is pending.
Do not treat earlier successful application tests as finalSHAcompletion until that run ends.

Final885aa7d CI43863645539 SUCCESS:1020backend/34skipped,21repo-level,
236webunit,10adminunit,10PostgreSQL,108browser/8skipped,5WorkOS. Architecture,
CodeQL and Postman checks also succeeded. Promoted885aa7d dev->prod fast-forward after
that result. Started production workflow43866474084 with deploy=true; app images now
building. Canceled duplicate prod-push CI43866475410 (same already-tested commit), so
the manual production workflow performs the deployment. No CI failure was bypassed.
Deployment/live final verification is still pending; previous runtime068d41b remains
until the health-gated rollout finishes. No owner provider gates have been enabled.

### Production rollback — 16 November12:35UTC

Workflow43866474084 successfully deployed885aa7d; exact public web/API SHAs and schema076
verified; legal/security/hub routes200. Manual catalogue refresh503fcb59-e292-4361-bc67-
2abede51d149 completed40.35s with34searches and7,357employers (35stored-statrows). Worker
heartbeat fix became healthy. However live hub renders repeatedly hit10s and salary/skills
queries accumulated15active sessions, some20s, starving readiness. Rolled application back
using the prepared legacy migration-bypass override. Rollback health gate passed for068d41b;
schema076/indexes remain additive. Public readiness healthy again. Gitprod still885aa7d;
DO NOT equate branch tip/deploy workflow success with current runtime.

Also found existing broker command stillallkeys-lru because app-only rollout does not recreate
Redis. Set noeviction immediately, then explicitly recreated only Redis from current Compose
with its existing AOF volume. Policy nowpersistentlynoeviction; dedicated query cache remains
allkeys-lru. Brief broker reconnect occurred; no data volume was removed. Auth/users untouched.
GitHub nowzero open Dependabot alerts. Admin shell200 with deployment UA; default Python UA
was403 at Cloudflare, not an admin outage.

Fix lane: coordinator adds lightweight landing-count endpoint, bounded hub count fetches and
API_INTERNAL_URL for web SSR; agentowns efficient bounded keyword aggregate SQL and PG tests.
Requires newtests/CI/candidate evidence before re-promotion. Do not declare release complete.

### Corrected candidate — 16Nov12:50UTC

Product692b504 pushed dev/integration, all product changes committed; runtime still068d41b
on additive076schema. Commits8aa04b7 count-only hub API/bounded4-at-a-time navigation and
API_INTERNAL_URL=http://api:8000; c98d1c4 keyword detail projection from one capped window;
6ea9db6 formatting;692b504 crucial partial-index predicate correction.

Actual cause verified with read-only production EXPLAIN (no ANALYZE): WHERE is_active IS TRUE
AND is_deleted IS FALSE choseParallelSeqScan(cost17580), whereas WHERE is_active AND NOT
is_deleted chose ix_job_posts_live_fts BitmapIndexScan(cost672). Existing partial indexes
have the latter predicate; PostgreSQL does not infer eligibility from the former spelling.
Corrected shared_filtered predicate; NOTNULLbooleans preserve semantics. NewPGregression
forces bitmap-planning and proves oldshape cannotuseFTS/newshape can. Three PGtests passed
(2,105/84,000fixture semantics plus indexeligibility), Ruff/mypy passed. NewPGtestincluded
CI migration-validation invocation. Previous synthetic timing alone showed no speedup and
didnotreproduceproduction: fixed~149ms/legacy144ms on short descriptions; do not claimitdid.

IsolatedVMcandidatec98 on the latest backup reproduced the problem without productionload:
Pythoncount4.935s; unfilteredstats380ms; jobslist123ms. Rootrender exceeded30s while API
readinessremained8–143ms dueboundedfanout. Allscratchcontainers/volumesremoved. Thusc98and
6eaareNOTaccepted;692b504 indexfix requires a fresh isolatedVMrender test using builtimages.
Helper/tmp/rap-candidate-c98-verify.sh onVM performsrestore/start/read/cleanup, latestlogs in
/tmp/rap-candidate-c98-verify.log and API/webdiagnosticsroot-only. Do not treat timeoutaspass.

692 workflows: CI43870170740, build-only43870170459, Architecture43870170667(success),
Postman43870170576, CodeQL43870170648. Earlierc98/6eaCI canceled as superseded; c98front
failurewasformatting (fixed); currentPrettierallsourcecheck passed. Localfocused5browser,
237webunit,17SEOAPItests and countsAPI+9deployregressions passed. Next: isolated692coldhub
andreadinessevidence; exact692CIgreen; onlythenpromoteprodandhealthgateddeploy. Prodbranch
currently885aa7d whileVMrollback068d41b; noautomaticbadre-deploypending.

Temporary registryloginasopc was established securely fromghauthtoken for candidateimage
pulls; log out after candidateverification. Never printtokenorbackupenv. Backup/source
credentialsremainroot-only. ScratchRedis muststayseparate (candidateharness does this).

692 isolated verification PASSED with production-like web outbound networking:
85,009restored jobs; firsthome1205ms, subsequenthome81ms, hubs71/77/62ms, APIhealth3samples
max90ms allhealthy. Count204ms vs priorc98observed4935ms. Exact692API/webimages; separate
scratchPG/Redis/MinIO; cleanupremovedcontainers/volumes/network. No productionload. Full
report2026-11-16-release-recovery.md. CI43870170740pending; build43870170459success.
