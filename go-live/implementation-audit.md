# Implementation Audit — Remote AI Platform

Phase 0 deliverable (temporary, internal). Audited 2026-09-26 against commit `6e7afac`
(branch `fix/deploy-verify-use-git-sha`), the live deployments, and the Figma Make export.
Code and live behaviour were treated as the source of truth; README content was not trusted.

## 1. Live environment snapshot

| Surface | URL | Result (2026-09-26 14:03 UTC) |
|---|---|---|
| Backend health | `remote-ai-platform-api.onrender.com/health/ready` | `HEALTHY`, env `production`, DB 1.6 s latency, Redis 0.6 s |
| Backend version | same | reports **`0.1.0`**, but `main` bumped to `0.1.1` in `d59094f` → latest backend build is **not** live, or the version string is stale. The open branch (`6e7afac`, verify against `git_sha`) addresses the verification side of this. |
| Jobs API (anonymous) | `/api/v1/jobs?limit=1` | returns real aggregated jobs |
| Frontend (workers.dev) | `remote-ai-platform.gokulraj22797.workers.dev` | 200 |
| Custom domain | `remoteaiplatform.com` | 200 |

DB/Redis latencies of 0.6–1.6 s on a readiness probe are high; worth tracking in Phase 5
(observability) — likely Render free-tier cold paths or cross-region Supabase.

## 2. Codebase inventory

**Backend** (`apps/api`, FastAPI modular monolith): 21 domain packages, **179 API operations**
(OpenAPI), 49 SQLAlchemy models, 32 Alembic migrations, Celery with 4 queues and 3 beat jobs
(job sync every 6 h, trending skills every 12 h, stale matches daily 02:00). LLM calls go through
LiteLLM only. No WebSocket endpoints — messaging and notifications are REST + polling.

Models: `User, PasswordResetToken, EngineerProfile, CompanyProfile, Skill, UserSkill, JobSkill,
JobPost, JobApplication, SavedJob, JobMatch, Recommendation, Contract, ContractMilestone, Project,
ProjectMember, Milestone, ProjectTask, TaskComment, TaskDependency, TaskAssignmentOffer,
WorkSubmission, WorkLedgerEntry, PaymentTransaction, ProjectReview, ProjectActivity, AIReport,
Connection, Conversation, Message, Notification, Group, GroupMembership, GroupPost, Post, PostLike,
PostComment, UserVerification, UserTrustScore, ModerationReport, AuditEvent, ActivityLog,
ApiSyncLog, AnalyticsEvent, AIUsageLog, Plan, Subscription, StripeWebhookEvent`.

**Frontend** (`apps/web`, Next.js 16 App Router): 51 routes, 25 TanStack Query hooks, ~18.7 K lines.
Auth = Supabase (email OTP, Google, Microsoft/azure, GitHub OAuth); session tokens in
`localStorage`; axios client with silent Supabase refresh on 401. Deployed to Cloudflare Workers via
OpenNext. Sentry wired (no-op without DSN).

**CI** (`.github/workflows`): `ci`, `codeql`, `dependency-review`, `gitleaks`,
`enforce-branch-flow`, `deploy-frontend`, `deploy-frontend-dev`, `keep-render-warm`,
`scheduled-job-sync`. Backend deploys via Render deploy hooks (`680af01`).

## 3. API contract check

Every distinct API call made by the frontend (**123**) resolves to a real backend operation — no
dangling calls. **56 backend operations are not used by the frontend**, and several of them are
exactly what the Figma screens need:

- Projects: `GET /{id}/tasks`, `GET /{id}/milestones`, `PATCH /milestones/{id}/status`,
  `POST /tasks/{id}/comments`, `GET /{id}/activity`, `GET /tasks/{id}/submissions`,
  `PATCH /task-offers/{id}/cancel`, `POST /{id}/ai/documentation`, `GET /reputation/{user}`
- Contracts: `PATCH /contracts/{id}`, `PATCH /contracts/{id}/milestones/{id}/status`
- Groups: `GET /{id}/members`, `PATCH /{id}`, `DELETE /{id}`, member role changes
- Admin: `GET /admin/dashboard`, `/audit-events`, `/feature-flags`, `PATCH /users/{id}/role`,
  `DELETE /users/{id}`, `DELETE /jobs/{id}`, `POST /jobs/reclean-text`
- Trust & safety: `PATCH /trust/verifications/{id}/review`, `POST /moderation/reports`
- Quality: `GET /quality/dashboard`, `POST /quality/batch-evaluate`
- Social: `GET /social/posts/public`, `GET/PATCH /social/posts/{id}`
- Legacy password auth: `/auth/login`, `/auth/register`, `/auth/token`, `/auth/refresh`,
  `/auth/login-url`, `/auth/logout-url` — superseded by Supabase; candidates for removal review
  in Phase 6 (attack surface), **not** removed now.

## 4. Test & tooling baseline (local)

| Check | Result |
|---|---|
| `pytest` (apps/api) | **305 passed, 2 failed** — both in `test_resume_upload_ai_parsing.py`, failing because MinIO is not running on `localhost:9000`. Environmental, not a code defect, but the tests should mock storage or be marked as integration. |
| `tsc --noEmit` (apps/web) | clean |
| `vitest` (apps/web) | 3 files, 12 tests, all pass — coverage is very thin (api client, `cn`, theme) |
| `eslint` (apps/web) | 0 errors, 2 warnings (`window.location.href` for internal navigation) |
| E2E | `tests/e2e` exists (axe-core a11y scans referenced in `9ca3146`) — not run in this audit |

Local dev findings (Phase 1 input): the checked-in `.venv` was missing declared deps
(`sentry_sdk`) until `pip install -e ".[dev]"` was re-run; the venv runs Python **3.12** from a
non-standard runtime path while the project targets **3.11**; Docker infra was not running.

## 5. Feature matrix

Legend — Prod: ✅ working end-to-end · 🟡 partial · ❌ missing. Figma: ✔ designed · — not designed.

| Feature | Backend API | Frontend | Figma | Tests | Prod | Missing work |
|---|---|---|---|---|---|---|
| Email OTP login / register | Supabase + `/auth/me`, `/auth/role` | `/auth/*` | ✔ | api client unit | ✅ | Reskin |
| Google / Microsoft / GitHub OAuth | Supabase | login/register | ✔ | — | 🟡 | Verify each provider live (Phase 7) |
| Engineer profile + completeness | `/engineers/me` | `/engineer/profile` | ✔ | backend | ✅ | Reskin |
| Resume upload + AI parsing | `/engineers/me/resume` | onboarding, profile | ✔ | 2 tests need storage | 🟡 | Storage mocking; verify R2/S3 in prod |
| AI profile enhance | `/engineers/me/ai-enhance` | profile assistant | — | backend | ✅ | Keep; restyle |
| Company profile | `/companies/me` | `/company/profile` | ✔ | backend | ✅ | Reskin |
| Job aggregation / sync | Celery + `/jobs/sync` | admin | ✔ (admin) | backend | ✅ | Admin sync page |
| Job search & filters | `/jobs`, `/search` | `/jobs` | ✔ | backend | ✅ | Reskin; `/search` page |
| Saved jobs | `/saved-jobs` | inside jobs | ✔ | backend | ✅ | `/saved` page |
| Applications + withdrawal | `/applications/*` | `/engineer/applications` | ✔ | backend | ✅ | Reskin |
| Company invitations | `/applications/jobs/{id}/invite/{eng}` | candidates | ✔ | backend | ✅ | Invitations tab |
| AI matching + explanations | `/matching/*` | recommendations, job detail | ✔ | backend | ✅ | Reskin |
| Skill gap analysis | matching explanation | recommendations | ✔ | backend | 🟡 | Surface in job detail |
| Candidate discovery + AI ranking | `/matching/candidates/{job}`, `/engineers/search` | `/company/candidates`, `/engineers` | ✔ | backend | ✅ | Reskin |
| Messaging | `/conversations` (REST) | `/messages` | ✔ | backend | 🟡 | Realtime (polling today) |
| Notifications | `/notifications` | `/notifications` | ✔ | backend | 🟡 | Realtime; unread badge in shell |
| Connections / network | `/connections` | `/network` | ✔ | backend | ✅ | Reskin |
| Social feed, likes, comments | `/social/*` | `/feed` | ✔ | backend | ✅ | Report post wiring |
| Groups | `/groups/*` | `/groups` | ✔ | backend | 🟡 | Detail page, members, admin |
| Projects / tasks / board | `/projects/*` | `/projects`, `/projects/[id]` | ✔ | backend | 🟡 | Board uses subset of APIs |
| Task offers marketplace | `/projects/task-offers` | workspace | ✔ | backend | 🟡 | Dedicated page |
| Submissions + AI review | `/projects/submissions/*` | project detail | ✔ | backend | 🟡 | "My submissions" list endpoint |
| Work ledger / time | `/projects/*/ledger` | project detail | ✔ | backend | 🟡 | Cross-project endpoint |
| Contracts + milestones | `/contracts/*` | `/contracts` | ✔ | backend | 🟡 | Milestone status UI; disputes |
| Escrow payments / wallet | `/payments/*`, Stripe webhook | `/payments` | ✔ | backend | 🟡 | Payouts/withdrawal missing; verify Stripe live mode |
| Reviews / trust scores / verification | `/trust/*` | badges, engineer detail | ✔ | backend | 🟡 | Admin verification queue list endpoint |
| Code-quality engine | `/quality/*` | `/quality` | ✔ | backend | 🟡 | Dashboard endpoint unused |
| Trending skills | beat task | gated "coming soon" | — | — | ❌ | Real implementation (Phase 4) |
| Admin: stats, users, jobs | `/admin/*` | 3 pages | ✔ | backend | ✅ | Reskin + role/delete actions |
| Admin: audit, AI usage/cost, sync logs, health, flags | `/admin/*` | partly in dashboard | ✔ | backend | 🟡 | Dedicated pages |
| Admin: moderation reports | `/moderation/reports` | — | ✔ | backend | 🟡 | Admin page |
| Admin: orgs, disputes, payouts, integrations | — | — | ✔ | — | ❌ | Backend gaps (see map) |
| Analytics events | `/analytics/*` | `lib/analytics` | — | backend | ✅ | Keep |
| Dark mode, ⌘K palette, workspace switcher | — | yes | — | theme unit | ✅ | Preserve through migration |
| Legal pages | — | yes | ✔ | — | ✅ | Reskin |

## 6. Top risks going into Phase 1–2

1. **Scope.** The Figma export is ~325 KB of minified mock UI across ~55 screens; a big-bang swap
   would violate "do not remove features". Migrate section by section (see migration map §5).
2. **Backend version drift** in production (0.1.0 live vs 0.1.1 on main) — resolve the open deploy
   verification branch before stacking frontend releases on top.
3. **Design ↔ accessibility conflict** on brand blue (`#0866FF` vs `#0552CC`) and missing dark mode
   in Figma — recommendations in the migration map §2.
4. **Backend gaps** surfaced by Figma screens (proposals, disputes, payouts, admin orgs, "my"
   cross-project submissions/ledger, verification queue listing) — to be built properly in Phase 3,
   never faked in the UI.
5. **Thin frontend test coverage** (12 unit tests) — each migrated section should land with
   component tests for loading/empty/error states.
6. **Docs location.** This repo's CLAUDE.md says in-depth docs live in the private internal repo;
   the go-live plan asks for `/docs/*.md` here. Decide before committing this file.
