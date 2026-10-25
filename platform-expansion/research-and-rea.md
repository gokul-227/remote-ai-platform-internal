# Platform expansion — research record and REA evidence (2026-10-25)

Plan: `remote-ai-platform-implementation.md` (owner's workspace). Branch: `feat/platform-expansion`
in the product repo.

## How research was done

| Channel | Used for | Notes |
|---|---|---|
| REA (`rea-agents@6.7.0`, Claude Code MCP + CLI) | **Our own software only**: `apps/web/src/figma`, `apps/admin/src` | Static, inert parsing; nothing executed. Evidence stored outside the repo (scratchpad), not committed. |
| Public product documentation / general public knowledge of user-visible behaviour | Feature behaviour of Facebook, LinkedIn, Upwork, Jira, Uber, Zoom, Slack, GitHub, Microsoft Office, Brave | No decompilation, no source recovery, no scraping, no automation against those services. |

REA was **not** run against the named third-party products. Their terms forbid reverse
engineering and no permission exists; the plan (§3 rules 7–9, §6.1) forbids it too. The owner
asked for it repeatedly; the request was declined and the alternative explained. Permitted REA
targets for future feature research are the open-source equivalents listed at the end.

Behaviour specs below were written from widely documented, user-visible behaviour, not from
pages fetched and archived during this session. Re-check the official docs listed before relying
on any specific detail. Nothing here claims knowledge of any competitor's internal algorithms.

## REA evidence (owned targets)

| Target | Evidence id | Files parsed | Findings | Result |
|---|---|---|---|---|
| `apps/web/src/figma` (before expansion) | `ev_a4dcf867…b368fb5` | 26 | 624 | Baseline route/data map; board, contracts, epics, priorities already present |
| `apps/web/src/figma` (after expansion) | rea-web2 run | — | — | 43 literal API paths extracted; **all 43 exist in `docs/api/openapi.yaml`** |
| `apps/admin/src` | rea-admin run | — | — | Analyzer completed; the console uses template-string paths, which the literal check does not cover |

REA limitations (from its own output): graph coverage is incomplete; inert parsing does not prove
runtime execution; dynamic or template paths can stay unresolved.

## Product cards

Each card: user problem → what users visibly do → what we built (independent design) → what
stays unknown or deferred → acceptance tests.

### Facebook-style community
- **Problem:** professionals want a feed of updates from people they know, with groups and safety tools.
- **Visible behaviour:** posts, comments, reactions, groups, blocking, reporting.
- **Ours:** already existed (`social`, `groups`, `network` blocks, `moderation`). Unchanged in this round; the admin console's Moderation module now acts on reports with a recorded note.
- **Tests:** existing `test_social_feed.py`, `test_blocking.py`, `test_groups.py`.

### LinkedIn-style identity and jobs
- **Ours:** existing profiles, organisations, jobs, applications, invitations, explainable matching. Unchanged.
- **Unknown:** LinkedIn ranking internals (never claimed).

### Upwork-style contracts and delivery
- **Problem:** a client needs to see exactly what was delivered for each milestone and approve it.
- **Ours (new):** `deliverables` — code (GitHub repo + full 40-char commit SHA, immutable), document (PDF/DOCX/PPTX/XLSX/TXT/MD/CSV, ≤15 MB, content checked against extension, macro-enabled Office refused, private bucket, 10-min presigned downloads) and demo links (http(s) only, no private/loopback hosts, opened with `noopener`). Revisions are new versioned rows. Accepting records a review only; copy says payment happens outside the platform.
- **Fixed:** contract page said payment "is released" on approval (implied escrow) — corrected.
- **Tests:** `tests/test_deliverables.py` (25), `collab.test.tsx`.

### Jira-style work management
- **Ours:** existing board/tasks plus new item types (TASK/BUG/STORY/EPIC), labels, watchers with notifications, optimistic concurrency (`expected_version` → 409), server-side filters/search/pagination.
- **Bugs fixed:** dependency cycles (A→B→A) were accepted; non-members could be assigned.
- **Tests:** `tests/test_work_items.py`.

### Uber-style allocation
- **Ours:** existing deterministic, explainable matching and invitations. Unchanged. No claim about Uber's internal dispatch.

### Zoom-style meetings
- **Decision (owner delegated):** scheduling + invitations + authorised join; the call runs on a generated Jitsi Meet room (`meet.jit.si`) or any https link the organiser supplies, opened in a new tab.
- **Why not native WebRTC:** reliable calls need a TURN relay (bandwidth costs money); meet.jit.si limits embedding. Native calling and recording are **deferred**.
- **Rules:** invitees must be connected or share a contract; blocks always win; join link released only to non-declined participants from 15 min before start to end; 8 h max; UTC storage.
- **Tests:** `tests/test_meetings.py`. **Not yet done:** two real browsers + restricted-network test.

### Slack-style messaging
- **Ours:** existing 1:1 messages plus new `channels` — project channels (membership follows project access live) and group channels (owner-managed, connections only), one-level threads, validated @mentions that notify, server-side unread per member, mute, idempotent sends (`client_token`, race-safe), keyset polling; UI polls every 15 s only while visible.
- **Tests:** `tests/test_channels.py`.

### GitHub deliverables
- **Ours:** commit references (above). **Deferred:** GitHub App installation, repo selection and signed webhooks — they need an App registered by the owner (free) and a webhook secret in Infisical.

### Word / PowerPoint / Excel
- **Ours:** secure upload, versioning, review. **Deferred:** previews and co-editing (no zero-cost isolated converter yet). Generated CSVs already neutralise formula injection (`apps/web/src/lib/csv.ts`).

### Brave-style safe demo viewing
- **Ours:** validated demo links with environment labels, opened in a new tab with `noopener,noreferrer`. **Deferred:** sandboxed embedded preview and remote browser isolation.

### AI assistant
- **Ours (new):** `assistant` — one metered LiteLLM call per message chooses at most one typed tool. READ (search_jobs, my_tasks, my_contracts, my_meetings), DRAFT (draft_text — never sent), WRITE (create_task, schedule_meeting, open_support_ticket) become pending actions with frozen args, a 10-min expiry, single execution (atomic claim, Postgres-tested). Tools run with the user's own permissions; tool output is never fed back to the model; admin/role/payment/delete tools do not exist.
- **Tests:** `tests/test_assistant.py`, `test_postgres_concurrency.py::test_assistant_action_confirmed_twice_at_once_runs_once`, `collab.test.tsx`.

### Separate admin application
- **Ours (new):** `apps/admin` static console (admin.remoteaiplatform.com / admin-dev…): overview + health, support queue (SLA, assignment, internal notes, reasons, versioned edits), people & access (search, role, suspend, sign-out-everywhere; reasons and typed confirmation), moderation, audit log, AI usage, read-only flags, job sources, integrations inventory. Writes are disabled when the API's environment differs from the console's. CORS allows only the same environment's console.
- **API hardening:** reasons recorded in `audit_events`; admins cannot suspend or demote themselves; suspension and role changes end existing sessions; new `POST /admin/users/{id}/revoke-sessions`.

## Permitted REA targets for deeper feature research

Open-source equivalents with public source (check each licence before analysis): Mattermost or
Rocket.Chat (Slack), Jitsi Meet (Zoom), Plane / Taiga / OpenProject (Jira), Mastodon (social feed),
Odoo (organisation/HR flows). Owned code and explicitly licensed fixtures are always permitted.
