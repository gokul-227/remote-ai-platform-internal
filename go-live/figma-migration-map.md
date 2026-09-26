# Figma Make → Production Frontend Migration Map

Phase 0 deliverable. Source: `Remote-AI-Platform-Figma-Make-Design.zip` (repo root, exported 2026-09-26),
cross-checked against the live FastAPI OpenAPI schema (179 operations) and `apps/web` as of
`6e7afac`.

## 1. What the Figma Make export actually is

| Aspect | Figma Make export | Current `apps/web` |
|---|---|---|
| Framework | Vite SPA, React 19, no router — `window.location.hash` + a `REG` lookup table | Next.js 16 App Router, React 19 |
| Data | 100% in-file mock arrays (`PEOPLE`, `JOBS`, `COMPANIES`, per-page literals) | TanStack Query hooks against the real API (25 hooks) |
| Auth | Simulated ("use 12345678 to continue") | Supabase email OTP + Google / Microsoft (azure) / GitHub OAuth |
| Styling | Tailwind v4 utilities + ~10 KB of hand-written `rap-*` (shell) and `v2-*` (page) classes in `src/index.css` | Tailwind v4 + token system in `globals.css` (warm cream palette, dark mode) |
| Components | `rap_kit.tsx`: `Ic` (inline SVG icon set, ~70 icons), `Av`, `Lg`, `Brand`, `Btn`, `Card`, `Tag`, `Tabs`, `Modal`, `Field`, `inputCls`, `Bar`, `Spark`, `Bars`, `Stars` | `components/ui/*` (Button, Card, Badge, Tabs, Modal, Drawer, Table, Toast, Skeleton, EmptyState, …) + lucide-react |
| Size | ~325 KB of heavily minified TSX across 13 files | ~18.7 K lines |

Files to ignore in the export: `src/data.ts` and `src_pages_modules.tsx` (not imported by the app),
`AGENTS.md` / `CLAUDE.md` / `.figma/*` (Figma Make tooling). `rap_enterprise.tsx` pages (Submissions,
Work log, Reviews, Admin users/jobs, Feature flags, Help center, Company setup, Payments) are
**designed but not registered** in the export's `App.tsx` `REG` table — the shell links to them but
the preview shows "not available". They are still part of the target design.

**Decision: port the design into `apps/web`; do not adopt the Vite app.** The plan requires one
canonical Next.js frontend (§6–7), and all real auth/data wiring already lives in `apps/web`. The
Figma code is used as a visual/UX reference and its primitives are re-implemented as typed
components.

## 2. Design language to adopt

- **Shell** (`rap_shell.tsx` + `.rap-*` CSS): fixed 64px white top bar (brand + pill search left,
  5 icon tabs centre — Home / My network / Jobs / Workspace / Hiring — round grey tool buttons right:
  menu, messages, notifications, avatar). Fixed 280px left sidebar whose content switches by
  *section* (Community, Careers, Workspace, Hiring, Account, Admin) plus "Your shortcuts". Feed gets
  a 280px right rail (recommended opportunity + contacts). Mobile (<900px): sidebar hidden, bottom
  tab bar. A "Menu" overlay lists every page by section.
- **Tokens**: page `#F0F2F5`, cards white with `#E4E6EB` border and 10px radius, text `#1C1E21`,
  muted `#65676B`, Inter 15px base, h1 26px/700, h2 20px/700, h3 16px/600.
- **Page grammar** (`.v2-*`): `v2-page-head` (title + subtitle + actions), `v2-metrics` (3-up stat
  tiles), `v2-toolbar` (search + filters), `v2-list`/`v2-row` (list rows with status pill),
  `v2-table`, `v2-section`, `v2-empty`, `v2-toast`, `v2-field`.
- **State screens**: loading / empty / error / access-denied designs exist in the shell
  (`rap-state`) — these become the shared feedback components.

### Design conflicts to resolve (recommendations)

| Conflict | Recommendation |
|---|---|
| Brand blue: newest Figma CSS uses `#0866FF`; Figma kit buttons and current app use `#0552CC` | Keep **`#0552CC`**. `#0866FF` is 4.8:1 on white but ~4.3:1 on the `#F0F2F5` page — fails WCAG AA for text. Commit `15f0d57` chose `#0552CC` for exactly this reason. |
| Figma has no dark mode; current app ships a theme toggle | Keep dark mode (feature preservation rule). Define dark values for every new token. |
| Figma has no public landing page (auth screen doubles as marketing) | Keep `app/page.tsx` landing for anonymous visitors; restyle it in the new language. |
| Figma forgot/reset flow is password-based | Current auth is passwordless OTP, but `/auth/forgot-password` + `/auth/reset-password` still exist server-side. Keep both, restyle. |
| Current app has ⌘K command palette, workspace switcher, trust badges, AI profile assistant — not in Figma | Keep all; restyle to match. |
| Figma "Preview state" selector and "Design preview · Sample data" banners | Drop — real loading/empty/error states come from query status. |

## 3. Screen → route map

Status key: **Reskin** = route + data exist, restyle only · **Reskin+** = restyle and wire extra
endpoints the Figma screen needs · **New route** = new Next.js page over existing APIs · **Gap** =
backend capability missing (Phase 3) · **Decide** = product call needed.

### Auth & onboarding

| Figma screen (`#hash`) | App route | Existing route | Backend | Status / notes |
|---|---|---|---|---|
| `login` | `/auth/login` | same | Supabase OTP + OAuth; `GET /auth/me` | Reskin |
| `register` (role picker + name) | `/auth/register` | same | Supabase; `PATCH /auth/role` | Reskin |
| `forgot` | `/auth/forgot-password` | same | `POST /auth/forgot-password` | Reskin |
| `reset` | `/auth/reset-password` | same | `POST /auth/reset-password` | Reskin |
| `callback` | `/auth/callback` | same | Supabase session | Reskin |
| `onboarding` (resume import → review → preferences) | `/onboarding` | same | `POST /engineers/me/resume`, `POST/PUT /engineers/me`, `POST /engineers/me/ai-enhance` | Reskin |
| Company setup (`CompanySetup`) | `/company/profile` (setup mode) | same | `POST/PUT /companies/me` | Reskin |

### Community

| Figma screen | App route | Existing | Backend | Status |
|---|---|---|---|---|
| `feed` + right rail | `/feed` | same | `/social/feed`, posts, likes, comments; rail: `/matching/recommendations?limit=1`, `/connections` | Reskin+ — wire "Report post" to `POST /moderation/reports` (exists, unused) |
| `groups` | `/groups` | same | `/groups`, `/groups/me/joined`, create/join/leave | Reskin |
| `group` | `/groups/[id]` | — (inline today) | `/groups/{id}`, `/posts`, `/members` (unused), `PATCH /groups/{id}` (unused) | New route |
| `messenger` | `/messages` | same | `/conversations`, `/messages` (REST polling; no WebSocket exists) | Reskin; realtime = Phase 4 |
| `notifications` | `/notifications` | same | `/notifications`, unread-count, read, read-all | Reskin |
| `network` | `/network` | same | `/connections` CRUD | Reskin |
| `profile` | `/engineer/profile` (`/profile` redirects) | same | `/engineers/me`, `/trust/*` | Reskin |
| `search` | `/search` | — (⌘K palette only) | `GET /search` | New route |

### Careers (engineer)

| Figma screen | App route | Existing | Backend | Status |
|---|---|---|---|---|
| `dash` | `/engineer/dashboard` | same | `/engineers/me`, recommendations, applications | Reskin |
| `jobs` (split list/detail) | `/jobs` | same | `/jobs`, `/saved-jobs`, `/applications/jobs/{id}` | Reskin |
| `jobdetail` | `/jobs/[id]` | same | `/jobs/{id}`, `/matching/jobs/{id}` | Reskin |
| `recs` | `/engineer/recommendations` | same | `/matching/recommendations`, `PATCH /matching/{id}/status` | Reskin |
| `applications` (+ withdraw modal, invitations tab) | `/engineer/applications` | same | `/applications/me`, withdraw | Reskin |
| `saved` | `/saved` | — (inside jobs) | `/saved-jobs` | New route |
| `companies` / `company` | `/companies`, `/companies/[id]` | same | `/companies/public`, `/companies/{id}`, `/jobs/company/{id}` | Reskin; "Posts"/"People" tabs = **Gap** |
| `work` (freelance "Find work", proposals) | `/work` | — | `/jobs?employment_type=CONTRACT`, `/projects/task-offers` | New route; "Submit a proposal" = **Gap** (no proposal model — use application + cover letter, or add) |

### Workspace & delivery

| Figma screen | App route | Existing | Backend | Status |
|---|---|---|---|---|
| `workspace` | `/workspace` → `/engineer/workspace` | same (redirect) | `/projects/my-tasks`, `/my-offers`, `/contracts/me` | Reskin; time tracker → ledger entries |
| `projects` (Jira-style board) | `/projects`, `/projects/[id]` | same | `/projects`, `/{id}/tasks` (unused), `/{id}/milestones` (unused), `PATCH /tasks/{id}`, `POST /tasks/{id}/comments` (unused) | Reskin+ |
| `taskmarket` / `TaskOffers` | `/workspace/offers` | — | `/projects/task-offers`, `/my-offers`, accept/cancel | New route |
| `submissions` | `/workspace/submissions` | — | `/projects/tasks/{id}/submissions` | New route; **Gap**: no "my submissions across tasks" endpoint |
| `worklog` | `/workspace/worklog` | — | `/projects/{id}/ledger`, `POST /tasks/{id}/ledger` | New route; **Gap**: no cross-project "my ledger" endpoint |
| `reviews` | `/workspace/reviews` | — | `/projects/{id}/reviews`, `/trust/reviews/{user}` | New route |
| `contracts` (Active/Paused/Ended/Disputes) | `/contracts` | same | `/contracts/me`, terminate | Reskin; Disputes tab = **Gap** |
| `contractsign` | `/contracts/[id]` | same | `/contracts/{id}`, `/sign`, milestones, `PATCH …/milestones/{id}/status` (unused) | Reskin+ |
| `earnings` / `copayments` / `Payments` | `/payments` | same | `/payments/wallet`, `/transactions`, escrow release/refund | Reskin; "Withdraw funds" / payouts = **Gap** |
| `quality` | `/quality` | same | `/quality/evaluate`, `/review-code`, `/dashboard` (unused) | Reskin+ |

### Hiring (company)

| Figma screen | App route | Existing | Backend | Status |
|---|---|---|---|---|
| `codash` | `/company/dashboard` | same | `/companies/me`, `/jobs/company`, `/applications/company` | Reskin |
| `cojobs` | `/company/jobs` | same | `/jobs/company`, `PATCH /jobs/{id}` | Reskin |
| `postjob` | `/jobs/new` | same | `POST /jobs` | Reskin |
| `candidates` | `/company/candidates` | same | `/matching/candidates/{job}`, `/applications/company`, status, invite | Reskin |
| `engineers` / `engineer` | `/engineers`, `/engineers/[id]` | same | `/engineers`, `/engineers/search`, `/engineers/{id}`, `/trust/*` | Reskin |
| `talent` | `/engineers` (`/freelancers` redirects) | same | `/engineers/search` | Reskin (merge into directory) |
| `coprofile` | `/company/profile` | same | `/companies/me` | Reskin |
| `ride` ("Get an engineer" instant match) | — | — | could compose `/matching/candidates` + invite | **Decide** — no product spec; not in go-live feature list |

### Admin

Figma has a full console (`rap_admin.tsx`, 18 sections) plus shell links `adminusers`, `adminjobs`,
`reports`, `verifications`, `audit`, `sync`, `aiusage`, `health`, `flags`.

| Figma section | App route | Existing | Backend | Status |
|---|---|---|---|---|
| Overview | `/admin/dashboard` | same | `/admin/stats`, `/admin/dashboard` (unused) | Reskin |
| Users / People & access | `/admin/users` | same | `/admin/users`, status, role (role unused), delete (unused) | Reskin+ |
| Jobs & listings | `/admin/jobs` | same | `/admin/jobs/{id}/status`, delete, `reclean-text` | Reskin+ |
| Verification queue | `/admin/verifications` | — | `PATCH /trust/verifications/{id}/review` | New route; **Gap**: no admin list-pending endpoint |
| Moderation / Content reports | `/admin/reports` | — | `GET/PATCH /moderation/reports` | New route |
| Audit log | `/admin/audit` | — | `/admin/audit-events`, `/admin/activity-logs` | New route |
| Job sync | `/admin/sync` | — (in dashboard) | `/admin/sync-logs`, `POST /jobs/sync` | New route |
| AI usage | `/admin/ai-usage` | — (in dashboard) | `/admin/ai-usage` | New route |
| System health | `/admin/health` | — | `/admin/health/details`, `/health/queues` | New route |
| Feature flags | `/admin/flags` | — | `/admin/feature-flags` (read-only) | New route (read-only, as designed) |
| Organizations | `/admin/organizations` | — | `/companies/public` only | **Gap**: admin org list/status |
| Disputes, Payouts, Integrations, Notifications, Platform settings, Security | — | — | none | **Gap** / **Decide** — not in go-live feature list; ship as hidden until backed |

### Account & legal

| Figma screen | App route | Existing | Backend | Status |
|---|---|---|---|---|
| `settings` | `/settings` | same | `PATCH /auth/me`, `/auth/change-password`, `/auth/logout-all` | Reskin |
| `security` ("Security & verification", sessions) | `/settings/security` | `/security` is a public trust page | `/trust/verifications`, `logout-all` | New route; per-session list = **Gap** (Supabase-owned) |
| `help` | `/help` | — | static content | New route |
| `terms` / `privacy` / `impressum` | same | same | — | Reskin |
| `NotFound` | `app/not-found.tsx` | — | — | New |

## 4. Current-app features not in the Figma design (must be preserved)

Dark mode toggle · ⌘K command palette · workspace switcher (`PATCH /auth/role`) · AI profile
assistant (`ai-enhance`) · trust badges · match-score component · first-party analytics events ·
Sentry · `/api/version` + `/health/version` routes (used by deploy verification) · public landing
page · public `/security` trust page · feature-flag gating of trending skills.

## 5. Proposed Phase 2 order and component plan

1. **Foundation**: `components/rap/` design-system primitives typed from `rap_kit` (Icon, Avatar,
   Logo, Button, Card, Tag, Tabs, Modal, Field, ProgressBar, Sparkline, Bars, Stars) + `PageHeader`,
   `MetricTiles`, `Toolbar`, `ListRow`, `DataTable`, `EmptyState`, `ErrorState`, `LoadingState`,
   `AccessDenied`. Tokens in `globals.css` (light + dark). Existing `components/ui/*` become thin
   re-exports during the transition so untouched pages keep compiling.
2. **App shell**: replace `LayoutShell`/`TopNavbar`/`Sidebar`/`MobileBottomNav`/`RightSidebar` with
   the Figma shell driven by real `useAuth()` role (engineer / company / admin sections), real
   unread counts, real search submit → `/search`.
3. Then feature by feature in the plan's order (auth → onboarding → engineer dashboard → profile →
   jobs → job detail → applications → company dashboard → candidates → ATS → messaging →
   notifications → network → social → groups → contracts → payments → projects → admin), one PR each.
