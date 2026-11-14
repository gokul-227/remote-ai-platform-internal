# Continuation record — transformation program (updated 14 Nov 2026)

Read this first in any new session. Owner decisions: memory `transformation-program-2026-11` (card-OK free tiers; owner creates accounts at the end with a guide from Claude; direct push to dev and prod; include work from the parallel Codex/Claude session; Upstash for production Redis; Files app at `#files` with Office inside).

```text
PHASE: 12 (Files) in progress; 0-8 done; 9 partial
STATUS: dev = c6ff169 (CI pending at time of writing); prod = 67eaf19 (live, healthy except Redis)
CURRENT BRANCH: feat/launch-discovery-hardening (pushed to dev directly; prod promoted after dev CI green)
COMMIT SHA: c6ff169

OBJECTIVES COMPLETED:
- Phase 0 audit, Phase 1 REA, Phase 2 research (product-research/*.md), Phase 3 design (03-*.md)
- Phase 4 infra code (prod compose, Caddy, GHCR publish) + runbooks (04-*.md); no VM yet
- Phase 5 Celery worker/beat, task history, outbox, dead letters, import scheduling, console screen
- Phase 7 dual-issuer auth (WorkOS-ready), linked identities, rollback by allow-list
- Phase 8 organisation roles, job preferences, notification preferences + digests + dedup, scoped search
- Phase 9 workspace tabs (Overview, Members, Resources, Activity, Settings), #project/<id>
- Phase 12 Files backend (folders, versions, inherited shares, links, trash, quota, sweep) + Files screen

FILES CHANGED: see git log 1df5b24..c6ff169 on the product repo

DATABASE CHANGES: migrations 056-061, all additive, each round-tripped on PostgreSQL:
056 background tasks, 057 user identities, 058 organisation members, 059 preferences,
060 trigram search indexes (PostgreSQL only), 061 files. Production is at 061 only after the next prod deploy.

INFRASTRUCTURE CHANGES: none in providers. New workflow publish-api-image.yml (GHCR).
scheduled-job-sync.yml gated by repo variable JOB_SYNC_GITHUB_ENABLED (not set; GitHub import still runs).

REA RESEARCH COMPLETED: signed-in Chrome snapshots (LinkedIn, Upwork, Slack, Jira, Facebook, Zoom);
native REA observations (Slack, Zoom, Brave, Word, Excel, PowerPoint). Evidence ids in research docs.

TESTS EXECUTED / RESULTS (last run):
backend 704 passed / 10 skipped; ruff + mypy clean; PostgreSQL concurrency 10/10;
web unit 114/114; Chromium e2e 60/60 (includes new files.spec.ts); docs-check clean.

SECURITY REVIEW: new endpoints enforce ownership/roles server-side; files never public (signed URLs,
5 min); link shares viewer-only with expiry; executables and macro Office files refused; unverified
extra-issuer emails cannot take over accounts (409).

RUNTIME VERIFICATION: worker/scheduler verified locally against real sources; prod live at 67eaf19.

COST AND QUOTA IMPACT: none yet. Files quota 2 GB/user, 25 MB/upload (settings).

KNOWN LIMITATIONS:
- Production Redis DOWN (owner: create Upstash free DB, put REDIS_URL in Infisical prod:/render-api)
- Files in production use the existing S3-compatible storage settings; FILES_BUCKET must exist there
  (or R2 when the owner enables it). Signed URLs from the E2E MinIO point at the internal host.
- Office previews (DOCX/XLSX/PPTX rendering) not implemented: preview says "download".
- Infisical CLI session expired on this machine.
- A parallel session edits the same checkout; always run full checks before committing.

ROLLBACK: redeploy previous SHA; every new migration has a downgrade; feature flags not needed
because all additions are new routes/tables.

NEXT EXECUTABLE TASKS (in order):
1. After dev CI is green: promote dev -> prod, watch deploy, verify /health/version and /health/ready.
2. Phase 12 rest: Office/PDF previews (client-side, no server execution), review/approval on files,
   files as chat attachments and deliverables.
3. Phase 9 rest: link contracts/milestones to project tasks.
4. Phase 10: group post approval queue, typed reaction tallies UI, moderation queue in console.
5. Phase 11: scheduled messages, my-threads view, meeting passcode + waiting room.
6. Phase 13 demo registry; Phase 14 assistant action classes + new tools + injection tests;
   Phase 15 staff roles, writable flags, catalogue, backup status (incidents started by parallel session).
7. Phase 16 assessment; Phase 17 production cut-over (OCI VM, worker on prod DB, disable GitHub import).
8. End: owner account guide (Upstash, Oracle, R2, Neon, Grafana, WorkOS, FCM) with card requirements;
   restore branch protection from ~/Developer/GitHub/.protection-backup.
```
