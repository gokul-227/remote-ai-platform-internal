# Phase 0 — Repository and production baseline (14 November 2026)

Program: `remote-ai-platform-implementation.md` (file dated 14 Nov 2026, 18 parts, phases 0–17).
This supersedes the 5 Nov 2026 checkpoint in `../go-live/`, which was written against an earlier
version of the same document. Nothing in this phase modified production, pushed, or deleted data.

## First progress report

| # | Item | Finding |
|---|---|---|
| 1 | Product repository | `/Users/gokulr/Developer/GitHub/remote-ai-platform`, `origin` = github.com/gokul-227/remote-ai-platform (public, proprietary LICENSE). |
| 2 | Internal repository | `/Users/gokulr/Developer/GitHub/remote-ai-platform-internal`, `origin` = github.com/gokul-227/remote-ai-platform-internal (private). |
| 3 | Branches and SHAs | Product: local `feat/launch-discovery-hardening` now at `70fd91e` (8 commits on top of `1df5b24`). `origin/dev` = `1df5b24`. `origin/prod` = `276e669` (8 "ci: restore / refresh build cache" commits over `1df5b24`, **zero file changes**, so prod content == dev content == the base of the feature branch). Internal: `main` = `685e5dd` with uncommitted 5 Nov docs. REA clone: `0beedcd`. |
| 4 | Unrelated worktree changes | Product: none after this phase (197 uncommitted files from the 5 Nov session were verified and committed in eight area-scoped commits, listed below). Internal: 5 Nov go-live docs are still uncommitted and are preserved. |
| 5 | Frontend architecture | npm workspaces + Turborepo. `apps/web`: Next.js 16.4.5 / React 19.3, hash-routed client product UI in `src/figma` (`App.tsx` routes, `rap_shell.tsx` catalog: My network, Discover, Communicate, Meet, Find jobs, My job search, Hiring, Find work, Engagements, Earnings, Plan & track, Documents, Development, Settings, Support), plus SSR public pages under `src/app` (jobs, people, companies, legal, sitemaps). Deployed as a Cloudflare Worker through OpenNext. `apps/admin`: static Next.js export staff console (Users, Organisations, Jobs, Freelancing, Delivery, Tickets, Moderation, Enforcement, Trust, Usage, Health, Audit, Settings). |
| 6 | Backend architecture | `apps/api`: FastAPI 0.142 / Python 3.11 / async SQLAlchemy 2.1 / Alembic (head `055_announcements`, 55 migrations) / LiteLLM free-tier chain. 32 domain packages, 33 routers under `/api/v1`, 70 tables. **No background worker**: resume parsing inline, job import runs on GitHub Actions (`scheduled-job-sync.yml`, every 3 h, direct to the production database). S3-compatible storage via boto3 (`core/storage.py`). Redis for rate limiting, cache, chat fan-out. |
| 7 | Database and authentication | Supabase Postgres per environment (prod `cvjypjsjmamoppwhuwdl` eu-west-1; dev `wqugjgtjjmixzsqqcmld`, paused on free tier). Supabase Auth is the only identity provider; API verifies JWKS tokens, role lives in `users.role` (`ENGINEER`/`COMPANY` = member, `ADMIN` = staff). Supabase Storage (S3) for resumes/assets. |
| 8 | Production infrastructure | Cloudflare DNS/CDN/Workers (web + admin). Render free web service for the API (Frankfurt, `api.remoteaiplatform.com`). Shared hosted Redis (provider not confirmable from this machine). Resend email, Sentry, UptimeRobot, Infisical secrets (CLI session on this machine has expired), GitHub Actions CI/CD. **Live on 14 Nov 2026 01:00 CET:** web `276e669` (deployed 12 Nov), API `276e669`, admin console 200. Dev API (`remote-ai-platform-api-dev-eu.onrender.com`) did not answer within 20 s (free instance asleep, Supabase dev project paused). |
| 9 | Real-user data dependencies | Live `users`, profiles, applications, contracts, documents, channels, meetings, notifications; 43k+ imported job posts (5 Nov sample). Resumes/assets in Supabase Storage. Supabase Auth identities keyed by `users.auth_subject` (never overwrite). Deletion refused while signed contracts exist (`auth/retention.py`). |
| 10 | Incomplete / broken modules | See capability inventory below. Headline gaps against the new plan: no durable background processing (Celery/outbox); no Files application (folders, shared, recent, starred, trash); no notification preferences/batching; search is per-domain `ilike` except jobs FTS; single `ADMIN` staff role (no least-privilege roles); no incident, service-catalogue, runtime feature-flag or deployment-history modules; native WebRTC is STUN-only; Office editors are bounded; matching score is unversioned. |
| 11 | Test status (this session, local) | Backend `pytest`: **668 passed, 10 skipped** (SQLite). PostgreSQL layer: **10 passed** on a disposable migrated DB; `alembic upgrade head` (55 migrations) and a `downgrade -1` / `upgrade head` round-trip succeeded. `ruff check`: clean. `mypy app`: clean (188 files). Web: `tsc` clean, **114 vitest passed**. Admin: `tsc` clean, 2 passed. `make docs-check`: up to date. Playwright Chromium suite: running in the background against a rebuilt isolated stack (result recorded in the phase record). |
| 12 | Immediate risks | (a) **Production Redis is DOWN** (`/health/ready` = DEGRADED, redis "unavailable"): rate limiting, list cache and chat fan-out are degraded on prod. (b) **Branch protection is OFF on `prod` and `dev`** (404 from the protection API; backups in `~/Developer/GitHub/.protection-backup/`). (c) Job imports write to the production DB from a GitHub runner with `PROD_DATABASE_URL` as a GitHub secret. (d) Dev Supabase project paused; dev API asleep. (e) Supabase free tier has no automatic backups; the restic/pg_dump path is scripted but not scheduled. (f) 10 open Dependabot PRs. (g) Owner-side rotations still pending (GitHub OAuth secret, Infisical token, Postman key, Resend key pasted in chat earlier). |
| 13 | REA installed? | Yes. `rea-agents@6.8.0` under `~/.local/share/rea-tool`, isolated Node 22.23.4, registered in `~/.claude.json` (servers: rea, chrome-devtools, drawio, postman, MCP_DOCKER). Verified in this Claude Code session in Phase 1 (see `01-rea-research-runbook.md`). |
| 14 | First prioritized backlog | See "Executable backlog" below. |

## Commits made in this phase (feature branch, not pushed)

| SHA | Scope |
|---|---|
| `8953f01` | deps: Next 16.4.5, Sentry 10.78.0, pypdf 6.21.4, pinned pip/setuptools, docker-ignore |
| `e9115ba` | lint: ruff import ordering across alembic migrations (no schema change) |
| `ac58412` | api: source permission gates, channel search/history cursors, social discovery, delivery portfolio, access fixes |
| `34397d3` | web: crawlable public homepage, legal pages, brand assets, sitemap shards, source attribution |
| `228527d` | web: channel history paging/search, feed discovery, Office draft recovery |
| `c4b5fc3` | admin: Delivery Operations screens, focus-trapped reason dialog |
| `735fba0` | e2e: new journeys; CI serves the staff console |
| `70fd91e` | docs: regenerated architecture and OpenAPI |

## Capability inventory (status vocabulary from the plan)

Statuses: **Working and verified** (runtime verified on prod or the isolated stack this session),
**Implemented, not runtime verified** (source + passing automated tests), **Partial**, **Broken**,
**Mock**, **Missing**, **Deprecated**, **Blocked**.

| Feature | Frontend | Backend / tables | Auth / authz | Tests | Status | Problems | Recommendation |
|---|---|---|---|---|---|---|---|
| Public job discovery (SSR, sitemaps, source attribution) | `src/app/*`, `remote-jobs`, sitemaps | `jobs`, `seo` routers; `job_posts` | anonymous read | `test_public_job_boards`, `test_sitemap_entries`, `public-discovery.spec` | Working and verified (prod returns listings, 43k posts) | Indexing/ranking unverifiable | Keep |
| Job import (15 sources) | admin Jobs/Health | `jobs/runner.py`, aggregators, `api_sync_logs` | GitHub secret DB URL | `test_ingestion_*` | Implemented, not runtime verified for this branch; prod runs old policy | Runs on GitHub Actions against prod DB; no retry/dead-letter/checkpoints; per-source quotas only partly modelled | **Replace** with Phase 5 Celery/outbox subsystem |
| Professional profile / onboarding | `rap_auth`, `rap_talent` | `engineers`, `engineer_profiles`, `user_skills` | Supabase JWT | `test_public_pages`, `test_self_service_privacy` | Implemented, not runtime verified | Real OAuth/refresh unverified in this session | Keep, verify |
| Organisation profile | `rap_talent`, `rap_enterprise` | `companies`, `company_profiles` | owner | `test_jobs` | Implemented | Single-owner org, no org membership/roles | **Improve** (Phase 8: organisation members/roles) |
| Jobs CRUD, applications, recruiter review, saved jobs | `rap_jobs`, `rap_work` | `jobs`, `applications`, `saved_jobs`; `job_applications`, `saved_jobs` | owner/applicant | `test_applications`, `test_jobs`, `journeys.spec` | Implemented, tests pass | Invitations/stages partly modelled | Keep, improve |
| Matching / recommendations | `#recs`, `#ride` | `matching`; `job_matches`, `recommendations` | user | `test_quality_engine` | Partial | Score version not persisted; hard eligibility not separated | Improve (Phase 8) |
| Freelance briefs, proposals, offers | `rap_work` | `projects`, `marketplace`; `work_submissions`, `task_assignment_offers` | server state rules | `test_work_lifecycle*` | Implemented | — | Keep |
| Contracts, milestones, deliverables | `rap_work`, `rap_collab` | `contracts/lifecycle.py`, `deliverables` | participants | `test_contract_lifecycle_enforcement`, `test_deliverables` | Implemented | Payments gated off (by design) | Keep |
| Project management (tasks, board, sprints, deps, watchers) | `rap_projects` | `projects`; `project_*`, `task_*` | members | `test_project_management` | Implemented | No unified workspace shell (overview/members/files/docs/messages/meetings/activity in one place) | **Improve** (Phase 9 unified workspace) |
| Social feed, posts, reactions, comments, groups, reports | `rap_social` | `social`, `groups`, `network`, moderation | audience rules | `test_social_*`, `test_groups` | Implemented | — | Keep |
| Connections / network | `rap_social` | `network`; `connections`, `user_blocks` | — | `test_network_layer` | Implemented | — | Keep |
| Channels, DMs, threads, reactions, search | `rap_collab`, `ChannelSearch` | `channels`; `channel_*`, `conversations`, `messages` | membership | `test_channels`, `test_channel_search`, `channel-history.spec` | Implemented | Attachments not established; polling + Redis fan-out (Redis currently down on prod) | Improve (Phase 11) |
| Meetings (schedule/invite/join) | `rap_collab` | `meetings`; `meetings`, `meeting_participants`, `call_signals` | participants | `test_meetings`, `platform.spec` | Implemented | Native WebRTC STUN-only: **Partial / Blocked** for restrictive networks; Jitsi/external link is the supported path | Keep bounded (Phase 13) |
| Docs / Sheets / Slides (native editors) | `rap_office` | `documents`; `documents`, `document_versions`, `document_comments`, `document_shares` | share ACL | `test_documents`, `useDocumentDraft.test` | Implemented, bounded | No DOCX/XLSX/PPTX **preview** pipeline for uploaded Office files; no PDF preview; no folders | Improve (Phase 12) |
| File management (My Files, Shared, Recent, Starred, Trash, folders, move/copy/rename) | — | — | — | — | **Missing** | Only resumes, deliverables and document rows exist | **Build** (Phase 12 Files app) |
| Demo viewer (sandboxed iframe) | `rap_collab`, `demoUrl.ts` | `browser` (proxy retired 410) | — | `test_browser`, `demoUrl.test` | Implemented | — | Keep |
| GitHub connection, commit deliverables | `#repos` | `github`; `github_connections` | encrypted token | `test_github` | Implemented, provider unverified | OAuth `repo` scope is broad | Keep, document |
| Notifications | shell bell | `notifications`; `notifications`, `email_outbox` | user | `test_notifications` | Implemented | **No preferences, batching, dedup or push**; email per event with outbox | **Improve** (Phase 8/11 unified notifications) |
| Search | per-screen | `search` router (`ilike`), jobs FTS | user | `test_search_indexing` | Partial | No unified cross-entity search, no trigram indexes | **Build** (Phase 8 unified search) |
| AI assistant (typed tools, confirmed writes) | assistant dock | `assistant/tools.py`; `assistant_actions`, `ai_reports` | per-tool authz | `test_assistant`, `test_ai_*` | Implemented | Tool classes are read/draft/write; "reversible vs consequential" not distinct; audit of tool calls partial | Improve (Phase 14) |
| Support tickets | `#support`, admin Tickets | `support`; `support_tickets`, `support_messages` | user/staff | `test_support` | Implemented | No incident module, no SLA timers beyond severity fields | Improve (Phase 15) |
| Admin console | `apps/admin` | admin, moderation, operations, portfolio routers; `audit_events`, `activity_logs`, `announcements` | `ADMIN` role only | `test_admin_*`, `operations-console.spec` | Implemented | **Single staff role**; flags env-only; no incidents, deployment history, queue/worker monitoring, service catalogue, backup status | **Improve** (Phase 15) |
| Feature flags | admin Settings (read) | `core/feature_flags.py` (env-only) | — | `test_feature_flags` | Partial | No runtime toggle/persistence | Improve (Phase 15) |
| Payments / billing | — | `payments`, `billing` (gated) | — | `test_payments*` | Deprecated / out of scope | Keep gated off; never claim | Keep gated |
| Observability | — | Prometheus instrumentator, Sentry (no-op without DSN), structlog | — | `test_observability` | Partial | No Grafana Cloud integration; no worker/queue metrics | Improve (Phase 4/15) |
| Backups | — | `ops/backup/*.sh` (pg_dump + objects) | — | — | Partial / not scheduled | Supabase free tier no automatic backups | Schedule + restore drill (Phase 4/6) |

## Capability gap matrix against the 14 Nov plan

| Gap | Business importance | User impact | Complexity | Dependencies | Security | Infra | Acceptance | Tests |
|---|---|---|---|---|---|---|---|---|
| Durable job import (Celery Beat/workers, Redis queue, PG task history, outbox, retry, dead-letter, checkpoints) | High (plan's first milestone) | Fresh catalogue without blocking API; operator visibility | Medium | Redis reachable from workers; worker host | DB URL leaves GitHub secrets | Worker container(s); Oracle VM later | Imports survive worker restart and retries; history in PG; manual retry/import from console | Unit + PG integration + restart scenario |
| Oracle Cloud Docker deployment (api, scheduler, worker, redis, proxy) | High | Reliability vs Render free sleep | Medium | **Owner must create OCI account (card verification), verify ARM capacity** | Private networking, no public Redis | OCI Always Free VM | New backend healthy on OCI without touching prod | Health, restart, resource limits |
| Cloudflare R2 object storage | Medium | Larger files, no Supabase storage cap | Low (S3 code path exists) | **Owner previously rejected R2 because it requires a payment method** | Signed URLs only | R2 bucket | Signed upload/download; metadata in PG | Integration with MinIO locally |
| Neon dev/staging DB | Medium | Unpause dev; migration rehearsal | Low | Owner creates Neon project (no card) | — | Neon free | Migrations apply; app runs | Migration tests |
| WorkOS AuthKit evaluation | Medium | — | High (identity mapping) | Owner account; dual-validation | Highest risk area | — | Compatibility report; no user loses access | Auth tests |
| Files application | High | Central files across projects/orgs | High | Storage, sharing model | Object-level ACL | Storage | Navigation set in plan; folder ops; versions; trash/restore | Permission + recovery tests |
| Unified project workspace | High | One place to work | Medium | existing tables | member ACL | — | Overview/Members/Tasks/Board/Files/Docs/Messages/Meetings/Deliverables/Activity/Settings | E2E journey |
| Unified notifications (prefs, batching, dedup, push) | High | Less noise | Medium | outbox, workers | — | FCM (owner) | Prefs honoured; digests; dedup | Unit + PG |
| Unified search (PG FTS + trigram) | Medium | Discovery | Medium | migrations (pg_trgm) | permission-aware | — | Results respect ACL | Authz tests |
| Organisation membership/roles | High | Teams, recruiters | Medium | companies model | org-level authz | — | Owner/admin/recruiter/member | Authz tests |
| Admin least-privilege roles, incidents, runtime flags, deployment/queue/worker/backup monitoring, service catalogue | High | Operations | Medium | job system, audit | server-side authz | — | Each module real data; no fabricated metrics | Admin security tests |
| Office previews (DOCX/XLSX/PPTX/PDF) | Medium | Deliverable review | Medium | storage | sandboxed | — | Preview or honest fallback | Round-trip tests |
| Native video reliability | Low (bounded) | — | High/cost | TURN | — | bandwidth | Not claimed; external links remain | Browser tests |

## Executable backlog (first prioritized)

1. Phase 1: verify REA inside this session; write the research runbook. (done this session)
2. Phase 2: bounded research packages, prioritized LinkedIn, Upwork, Slack, Jira, Facebook, Zoom, Files, Word, Excel, PowerPoint, Brave, Notion, admin/ops platforms. Official documentation first; prior 5 Nov REA interface evidence reused, not repeated.
3. Phase 3: architecture and migration design, dependency graph, rollback for every migration.
4. Phase 4: Docker Compose production profile (api, scheduler, worker, redis, caddy) with health checks, limits, private network; Grafana Cloud/backup hooks; **owner actions** for OCI/Neon/R2 accounts recorded in the cost register.
5. Phase 5: Celery + Beat + workers + PG task history + outbox + import adapters with checkpoints/dedup/retry/dead-letter + admin monitoring; GitHub Actions import kept until the replacement is proven.
6. Phases 6–7: storage/DB/auth evaluations behind owner-provided accounts; code made provider-agnostic now.
7. Phases 8–15: product completion in the order of the plan.
8. Phase 16–17: evidence-based release assessment; production migration only on owner authorization.

## Gate

No destructive modification was made. Production, Supabase, Render, Cloudflare and GitHub remote state are unchanged.
