# Continuation record — transformation program (updated 14 Nov 2026, evening)

Read this first in any new session. Owner decisions: memory `transformation-program-2026-11` (card-OK free tiers; owner creates accounts at the end with a guide from Claude; direct push to dev and prod; include work from the parallel Codex/Claude session; Upstash for production Redis; Files app at `#files` with Office inside).

```text
PHASE: 0-16 implemented and tested on dev; 17 (provision + cut-over + single prod deploy) waits on owner accounts
STATUS: dev = 3eadd6e (CI green); prod = 67eaf19 (unchanged, by decision: one prod deploy at the very end)
CURRENT BRANCH: feat/launch-discovery-hardening (pushed to dev directly)

LATEST (14 Nov 2026, Claude):
- Staff roles + feature-flag overrides (067) with console screens; E2E staff-console.spec.ts
- Backup/restore evidence: POST /ops/backups/report (BACKUP_REPORT_TOKEN), /admin/backups,
  inventory shows real freshness; /admin/catalogue service catalogue; console screens (068 backup_runs)
- 068 also repairs messages.attachment_ids where an early 063 draft ran (IF NOT EXISTS)
- Restore drill on E2E stack: 93 tables, 16 objects verified. Found and fixed: extensions
  (pg_trgm) were not restored from a public-schema dump
- k6 tests/load/public-read.js: p95 jobs 27 ms, search 64 ms, 0 errors (local Docker)
- Full Playwright: 83 passed on Chromium, Firefox, WebKit
- Phase 9 milestone-task links and Phase 10 reaction tallies were already built (verified)

NEW SECRET NAMES FOR INFISICAL (owner): BACKUP_REPORT_TOKEN (API and backup host, same value)

NEXT: owner follows owner-infrastructure-guide.md; then Phase 17 cut-over steps in that guide;
restore branch protection from ~/Developer/GitHub/.protection-backup/; rotate keys pasted in chat.
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
