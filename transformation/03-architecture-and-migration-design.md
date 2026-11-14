# Phase 3 — Architecture and migration design (14 November 2026)

> **Readiness correction — 2026-11-14:** Read [the current phase checkpoint](2026-11-14-phase-readiness-checkpoint.md) before using this historical design/runbook. Phases 4/6/7/16/17 are not complete. Provider prices, allowances and account status below are historical planning claims, not newly verified entitlements. This document does not authorize deployment or provider changes.
>
> Development must use isolated dev secrets/data, not the production export/canary instructions below. Do not use the 1 GB micro fallback or assume workloads prevent OCI reclaim without capacity/entitlement validation. Host port 8000 is not published by Compose; verify through Caddy or inside the API container and inspect readiness JSON. Workers/scheduler need validated external egress. Require a tested image digest; `latest` is not release proof.
>
> Migration rollback must preserve new data: do not routinely downgrade and drop tables. Actual identity design uses `user_identities` (057), not `auth_subject_workos`; source migrations extend through 060 at the checkpoint. R2 configuration currently uses the S3-compatible `MINIO_*` interface, not the proposed `R2_*` variables. Backup scripts exist, but scheduled/restic backups and a current full restore are unverified. The linked checkpoint supersedes conflicting instructions and lists the required evidence.

Inputs: Phase 0 audit, Phase 2 research packages, owner decisions of 14 Nov (card-on-file OK for free tiers; owner provisions accounts; direct push to dev/prod authorized; branch protection later).

## 1. Existing module map and reuse decisions

| Logical domain (plan §7.1) | Existing table(s) / package | Decision |
|---|---|---|
| User | `users` (role ENGINEER/COMPANY/ADMIN, auth_subject) | Keep. Add `staff_roles` for least-privilege staff (Phase 15). Never overwrite `auth_subject`. |
| Professional profile | `engineer_profiles`, `user_skills`, `skills`, `skill_endorsements`, `user_verifications`, `user_trust_scores` | Keep. Add `job_preferences` (Phase 8). |
| Organisation | `company_profiles` | Keep. Add `organisation_members` (role: owner/admin/recruiter/member) (Phase 8). |
| Job | `job_posts`, `job_skills` | Keep. Add `source` config table `job_sources` for scheduling/quotas (Phase 5). |
| Application | `job_applications`, `saved_jobs` | Keep. Add tracker stage field if missing (Phase 8). |
| Freelance project / proposal | `projects` (brief kind), `work_submissions`, `task_assignment_offers`, `project_reviews` | Keep. |
| Contract / milestone | `contracts`, `contract_milestones`, `deliverables` | Keep; `lifecycle.py` stays the only state machine. |
| Project workspace / work item | `projects`, `project_members`, `project_tasks`, `task_*`, `project_sprints`, `project_activity`, `milestones` | Keep. Add project workflow statuses (Phase 9) and a workspace shell in the web app. |
| Conversation / message / channel | `conversations`, `messages`, `channels`, `channel_members`, `channel_messages`, `channel_message_reactions` | Keep. Add `scheduled_at`, attachments referencing `file_nodes` (Phase 11). |
| Meeting | `meetings`, `meeting_participants`, `call_signals` | Keep. Add passcode/waiting-room/recurrence (Phase 13). |
| File / folder | — | **New** `file_nodes`, `file_versions`, `file_shares`, `file_activity` (Phase 12). Resumes and deliverable objects remain where they are; new uploads go through Files. |
| Document | `documents`, `document_versions`, `document_comments`, `document_shares` | Keep. Add `parent_id`, `space`, review state (Phase 12). |
| Deliverable | `deliverables` | Keep; link to `file_nodes` when uploaded through Files. |
| Notification | `notifications`, `email_outbox` | Keep. Add `notification_preferences`, `notification_digests` (Phase 8/11). |
| Support ticket / incident | `support_tickets`, `support_messages` | Keep. Add SLA fields and **new** `incidents`, `incident_events` (Phase 15). |
| Audit event | `audit_events`, `activity_logs` | Keep; every new admin mutation writes `audit_events` with reason. |
| Background jobs | `api_sync_logs` | Keep for history; **new** `task_runs`, `outbox_events`, `job_import_checkpoints`, `dead_letters` (Phase 5). |
| Feature flags | env-only | **New** `feature_flags` with audit (Phase 15); env values remain defaults. |
| Service catalogue | — | **New** `tools/architecture/catalog.yaml` already exists for docs; extend into `service_catalog` read by admin (Phase 15). |

Rule: no duplicate business entities. New tables reference existing ids; nothing is copied.

## 2. Authorization boundaries

1. **Member** (ENGINEER/COMPANY): owns their profile, applications, proposals, documents, files; ownership checks `user_id == current_user.id`.
2. **Organisation role** (new `organisation_members.role`): owner > admin > recruiter > member. Recruiter: publish/close jobs, review applicants. Admin: members, org files, org pages. Owner: everything + delete org. Job and application endpoints gain `require_org_role(org_id, {"owner","admin","recruiter"})`.
3. **Project role** (`project_members.role`): owner/manager/member/viewer. Transition rights per role; viewer read-only.
4. **Share grants** (`file_shares`, `document_shares`): principal user/org/project/link with role viewer/commenter/editor; folder inheritance; "broadest wins".
5. **Staff role** (new `staff_roles`): super_admin, ops_admin, support, moderator, eng_operator, auditor. Every `/admin/*` dependency resolves the staff role set; `ADMIN` in `users.role` remains the gate to the console, roles refine it. Auditor is read-only (enforced by a dependency that rejects non-GET).
6. **AI tools**: each tool declares `kind` (read / draft / reversible_write / consequential_write / prohibited) and re-checks the domain authorization at execution time; consequential writes require confirmation tokens (exists) and are audited.

## 3. Frontend structure

- `apps/web` keeps hash routing. Primary navigation evolves from the current seven sections into the plan's eleven: Home, Discover, Jobs, Freelance, Projects, Messages, Meetings, Files, AI Assistant, Notifications, Profile. Sub-navigation stays inside each section; the catalog in `rap_shell.tsx` is the single source.
- **Project workspace** route `#project/<id>/<tab>` with tabs Overview, Members, Tasks, Board, Files, Documents, Messages, Meetings, Deliverables, Activity, Settings; existing screens are mounted into tabs rather than rewritten.
- **Files** section is new (`rap_files.tsx`).
- `apps/admin` remains the separate console; new modules: Roles, Incidents, Queues & Workers, Imports, Flags (writable), Catalogue, Backups.

## 4. Infrastructure transition (no big bang)

| Step | Change | Fallback / rollback |
|---|---|---|
| 4.1 | Add production Compose profile `infra/docker/docker-compose.prod.yml`: api, scheduler (Celery Beat), worker, redis (no published port), caddy reverse proxy, alloy (optional). Health checks, restart policies, memory/CPU limits, named volumes. | Nothing deployed until an OCI VM exists; Render keeps serving. |
| 4.2 | Owner creates OCI Always Free tenancy (A1 2 OCPU/12 GB or E2 micro), opens 80/443 only, installs Docker; secrets delivered via Infisical CLI/agent. | VM is a single point of failure (documented); Render remains fallback until Phase 17. |
| 4.3 | Dev backend on OCI (`api-dev`), DNS `api-dev.remoteaiplatform.com` via Cloudflare proxied. | Point DNS back to Render dev. |
| 4.4 | Workers/scheduler run on OCI against the **production** database only after Phase 5 proves them on dev. | Re-enable GitHub Actions schedule (kept disabled, not deleted). |
| 4.5 | Grafana Cloud free stack: Alloy remote-writes API/worker metrics; UptimeRobot stays. | Remove Alloy container. |
| 4.6 | Backups: `ops/backup/backup.sh` scheduled by Celery Beat on OCI to restic → Oracle Object Storage (20 GB free) once verified; until then the existing GitHub artifact workflow. | Keep GitHub artifact backups. |
| 4.7 | Cloudflare: Turnstile on public signup/contact forms; WAF managed rules; rate-limit rule on `/api/v1/auth/*`. | Disable rule. |

## 5. Authentication migration design (Phase 7)

- Keep Supabase Auth as the issuer for all existing users. Evaluate WorkOS AuthKit in **staging only** (free, not billed).
- Backend `auth/dependencies.py` gains a multi-issuer JWKS verifier keyed by `iss` (allow-list from settings); audience per issuer.
- Identity mapping: on first WorkOS token, match `users` by verified email; store `auth_subject_workos` alongside `auth_subject` (never overwrite). Unmatched → create user as today.
- Password hashes are not exported; users re-authenticate through WorkOS (magic link/OAuth) when switched. No deletion of Supabase users; rollback = remove the WorkOS issuer from the allow-list and the frontend provider flag.
- Decision gate: compatibility report with measured sign-in, refresh, revocation, recovery, OAuth (Google/GitHub) results on dev before any production switch.

## 6. Database strategy (Phase 6)

- Production stays on Supabase. Neon free (no card) for **dev** and **staging**: owner creates project; `DATABASE_URL` for dev API points to Neon; Alembic `upgrade head` on empty Neon proves migrations; a sanitized subset or synthetic data only.
- Extension check list: `pg_trgm`, `uuid-ossp`/`gen_random_uuid`, `citext` if used; RLS policies in migration 042 must apply on Neon (no Supabase roles → guard with `DO $$ IF EXISTS role` blocks).
- Production move to Neon only after benchmark + backup/restore drill; not in this program's scope unless the owner asks.

## 7. Object storage (Phase 6/12)

- S3-compatible client already abstracts endpoint. Add bucket config per purpose: `resumes`, `assets`, `files`. R2 when the owner enables it (card on file, free 10 GB / 1M Class A / 10M Class B, free egress); until then Supabase Storage for prod and MinIO locally.
- Signed PUT upload (`create_upload` → client PUT → `finalize` verifies size/type via HEAD) and signed GET download with 5-minute expiry; no public objects for private files.
- Quotas in settings: `FILES_MAX_UPLOAD_MB=25`, `FILES_USER_QUOTA_MB=2048`, `FILES_ORG_QUOTA_MB=10240` (adjustable).

## 8. Background processing architecture (Phase 5)

```
Celery Beat (single instance, PG advisory lock) ─▶ Redis broker ─▶ Celery workers ─▶ domain services ─▶ PostgreSQL
                                   ▲                                            │
PostgreSQL outbox_events ──dispatcher task (every 10 s)──────────────────────────┘ (publishes events transactionally created by API requests)
task_runs: one row per task execution (queued/started/succeeded/failed/retrying/dead), args hash, attempts, next_retry_at, worker id, timings
job_import_checkpoints: per source cursor/page/since, last_success_at, last_error
dead_letters: exhausted tasks with payload for manual retry from the admin console
```

- Idempotency: task id = hash(task name + args + schedule slot); duplicate enqueue is a no-op if a run exists in a non-terminal state.
- Retry: exponential backoff (1 m, 5 m, 15 m, 1 h), max 5 attempts, then dead letter.
- Source scheduling: `job_sources` table (enabled, cron, max_per_run, rate_limit_per_minute, permission_confirmed); Beat reads it every minute.
- Single Beat: `pg_try_advisory_lock(hash('celery-beat'))` held by the scheduler process; a second Beat exits.
- Durable record: Redis is a transport only; `task_runs` is the record.
- Health: `/health/ready` adds `scheduler_heartbeat_age` and `worker_heartbeat_age` from a `worker_heartbeats` table; admin console shows queues, runs, dead letters, imports.
- Compatibility: `python -m app.domains.jobs.runner` remains usable; GitHub Actions schedule disabled only after Phase 5 gate (import survives worker restart and retry test).

## 9. Realtime

Keep current polling + Redis fan-out. Measure Supabase Realtime limits in Phase 11; Cloudflare Durable Objects (free tier SQLite-backed) is the candidate if a WebSocket hub is needed without the OCI VM. No change until measured.

## 10. Unified notifications and search

- Notifications: `notify(event)` service → in-app row, preference check (`notification_preferences`: per category, channel in-app/email/push, digest window), dedup key (`event_type:object_id:user_id` within 10 min), digest batching by worker every 15 min, email via existing outbox, push via FCM when the owner adds credentials.
- Search: PostgreSQL FTS (`tsvector` generated columns) + `pg_trgm` indexes on jobs, profiles, organisations, projects, tasks, files, documents, channel messages; a `search` router returning typed results filtered by the caller's access; one migration adds the extension and indexes (`CREATE INDEX CONCURRENTLY` is not available inside Alembic transactions, so the migration runs with `transaction_per_migration` and plain CREATE INDEX on tables small enough; `job_posts` index is created by a one-off script with CONCURRENTLY).

## 11. Dependency graph (build order)

```
Phase 4 compose/prod profile ──▶ Phase 5 Celery/outbox ──▶ Phase 8 notifications digests, Phase 12 trash sweeps, Phase 15 queue monitoring
Phase 6 Files storage (signed URLs) ──▶ Phase 12 Files app ──▶ Phase 11 attachments, Phase 9 workspace Files tab
Phase 8 org members/roles ──▶ Phase 9 workspace roles ──▶ Phase 15 org management
Phase 8 unified search ──▶ Phase 12 file/doc search, Phase 14 assistant search tools
Phase 15 staff roles ──▶ all new admin modules
Phase 7 auth evaluation is independent (staging only)
```

## 12. Migration rollback policy

Every Alembic revision in this program: additive only (new tables, nullable columns, indexes); `downgrade()` drops exactly what `upgrade()` added; no data rewrites in migrations; backfills run as idempotent tasks. Each is round-tripped (`upgrade head` → `downgrade -1` → `upgrade head`) on a disposable PostgreSQL before commit. Production applies migrations at API start (`start-production.sh`), so a bad revision is reverted by deploying the previous SHA after `downgrade`.

## 13. Infrastructure cost register (verified 14 Nov 2026)

| Provider | Free allowance (official page) | Card needed | Status |
|---|---|---|---|
| Oracle Cloud Always Free | 2× E2 micro or A1 2 OCPU/12 GB; 200 GB block; 20 GB object; 10 TB egress; idle reclaim rule | Yes (verification) | Owner to create |
| Neon Free | 100 projects, 1 GB/project (20 GB total), 100 CU-h/project, autosuspend 5 min, 6 h PITR | No | Owner to create |
| Cloudflare R2 | 10 GB-month, 1M Class A, 10M Class B, free egress | Yes (billing enabled) | Owner to enable |
| Cloudflare Workers Free | 100k req/day, 10 ms CPU; KV/D1/DO free quotas | No (in use) | In use |
| WorkOS AuthKit | 1M MAU free; staging free; SSO $125/connection | Not stated | Evaluate (Phase 7) |
| Grafana Cloud Free | 10k series, 50 GB logs/traces/profiles, 14 d, 3 users | No | Owner to create |
| Upstash Redis Free | 500k commands/month, 250 MB, 10 GB bandwidth | No | Candidate for prod Redis replacement (prod Redis is currently DOWN) |
| Render Free | 750 instance-hours/month | No | In use (API) |
| Supabase Free | 2 projects (prod, dev paused) | No | In use |
| Resend, Sentry, UptimeRobot, Infisical, GitHub | existing free tiers | No | In use |

Caps to implement in code: AI token allowances (exist), upload size/quota (Phase 12), import per-run caps (Phase 5), email per-user daily cap (Phase 8), R2 operation budget alert via admin storage monitor (Phase 15).
