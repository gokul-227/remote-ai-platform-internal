# Implementation md §8 — feature checklist (2026-10-25)

Status legend: ✅ built + automated tests pass · 🟡 built, needs a signed-in live check by the owner ·
⏸ deferred with reason. "Live" below means deployed to prod and dev and verified by version/route
checks; no signed-in live test has been possible (no test credentials).

| § | Product inspiration | What exists in the app | Tests | Status |
|---|---|---|---|---|
| 8.1 | Facebook — social | Feed, posts, comments, groups, connections, blocking, reports/moderation, one-per-person changeable reactions with counts | test_social_feed, test_blocking, test_groups, test_research_features | ✅ 🟡 |
| 8.2 | LinkedIn — profiles & jobs | Profiles, organisations, job search/filters, applications + status history, invitations (7-day expiry, decline reasons, cancel, per-job cap, closed jobs can't invite), proposal editing, skill endorsements, hidden profiles stay hidden | test_applications, test_profile_visibility, test_inspired_features, test_research_features | ✅ 🟡 |
| 8.3 | Upwork — freelancing | Contracts + milestones (server-side lifecycle), deliverables (code/document/demo, versioned revisions, client review), "Freelance work for you", task offers, work log; copy says payment happens outside the platform; payments stay off | test_contracts, test_deliverables, test_freelance_recommendations | ✅ 🟡 |
| 8.4 | Jira — projects | Boards, item types, labels, priorities, watchers (auto-watch on create/comment/update; notified on changes and comments), dependencies with cycle rejection, optimistic concurrency, filters/search/pagination, sprints with backlog return, activity history | test_work_items, test_inspired_features, test_project_management | ✅ 🟡 |
| 8.5 | Uber — matching/allocation | Deterministic explainable scoring (skills/experience/role/timezone/availability/rate), recommendations for jobs and freelance work, invitation states incl. expiry and capacity, concurrency tested on PostgreSQL | test_matching, test_freelance_recommendations, test_postgres_concurrency | ✅ |
| 8.6 | Slack — messaging | DMs (polling), project/group channels, threads, mentions, server-side unread, per-channel notify level, mute, edit/delete, emoji reactions, idempotent sends | test_channels, test_inspired_features, test_research_features | ✅ 🟡 |
| 8.7 | Zoom — meetings | Scheduling, invitations, accept/decline, organiser add/remove participants, reschedule, join window; **in-app video calls** (WebRTC, free STUN, camera/mic/screen share, audio-only fallback) with the free Jitsi room as backup | test_meetings (incl. signalling privacy) | ✅ 🟡 — two real browsers + a strict-network test still needed |
| 8.8 | GitHub — code deliverables | Commit-SHA deliverables (immutable), public commit check, users connect their own GitHub (OAuth, encrypted token, revoke on disconnect), repo/commit picker, private-repo verification with the submitter's token | test_github, test_deliverables, test_inspired_features | ✅ 🟡 — prod only (OAuth app callback is the prod API) |
| 8.9 | Word / Excel / PowerPoint | Built-in Docs (rich text, .docx import/export), Sheets (grid, own formula engine, .xlsx import/export), Slides (layouts, present mode, .pptx export); sharing view/comment/edit, comments, autosave, restorable versions; secure document deliverables with safe CSV/TXT preview | test_documents, sheet_formula.test.ts, test_deliverables | ✅ 🟡 — not full Microsoft parity; no real-time co-editing |
| 8.10 | Brave — demo viewing | Validated https demo links (no private hosts), sandboxed in-app demo viewer, open-in-new-tab fallback | test_deliverables, collab.test.tsx | ✅ |
| 8.11 | AI assistant | Typed tools: search jobs, my tasks/contracts/meetings/applications/channels, freelance recommendations, drafts; confirmed writes: create task, schedule meeting, apply, post to channel, create doc, open ticket. Runs with the user's permissions, single execution, no admin tools, outage-safe | test_assistant, collab.test.tsx, PG double-confirm test | ✅ 🟡 needs a configured free AI provider |
| 8.12 | Separate admin app | admin.remoteaiplatform.com / dev.admin…: overview + health, support tickets (SLA, internal notes), people & access (search, role, suspend, sign-out-everywhere, reasons), content reports, job moderation, verifications, organisations, audit log, AI usage, flags (read-only), job sources, integrations; env-mismatch write guard; main-app admin pages point here | test_support, test_admin_access_controls, test_admin_console_cors, apps/admin vitest | ✅ 🟡 |

## Deferred / owner actions

- Google sign-in: the Google OAuth client was deleted in Google Cloud → recreate it and update Supabase
  (prod + dev). Email-code sign-in works meanwhile.
- `ADMIN_EMAILS` and GitHub settings are in Infisical; confirm they reach Render (sign in at the console).
- Rotate later (owner's choice): GitHub OAuth client secret and the Infisical token pasted in chat.
- Native calls on strict corporate networks need a TURN relay (not free) → Jitsi backup room is used.
- Real-time multi-cursor co-editing of Docs/Sheets/Slides, Office-file previews of .docx/.xlsx/.pptx
  deliverables (they import into the editors instead).
