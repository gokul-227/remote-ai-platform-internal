# Platform expansion — handoff (2026-10-25)

## Implemented (product repo, branch `feat/platform-expansion`)

| Area | What | Migration | Tests |
|---|---|---|---|
| Deliverables | Code commit / document / demo per contract milestone; client review; versioned revisions | 044 | `test_deliverables.py`, `collab.test.tsx` |
| Meetings | Scheduling, invitations, authorised join window, Jitsi or own link | 045 | `test_meetings.py` |
| Support | Tickets, staff queue, SLA, internal notes, reasons, versioned edits, audit | 046 | `test_support.py` |
| Work items | Types, labels, watchers, `expected_version`, cycle-safe dependencies, member-only assignees, filters | 047 | `test_work_items.py` |
| AI assistant | Typed tools, confirm/cancel, expiry, single execution, user permissions | 048 | `test_assistant.py`, PG concurrency test |
| Channels | Project/group channels, threads, mentions, unread, mute, idempotent sends | 049 | `test_channels.py` |
| Admin API | Reasons in audit, no self-lockout, session revocation, user search, per-env console CORS | — | `test_admin_access_controls.py`, `test_admin_console_cors.py` |
| Staff console | `apps/admin` static app + deploy jobs (dev/prod) | — | `apps/admin` vitest, CI build |
| Web | Meetings, Support, Assistant, Channels screens; deliverables on contracts; board fields | — | `collab.test.tsx`, existing suite |

All migrations are additive (new tables/columns with server defaults, RLS on, no policies).

## Disabled or deferred (honest list)

- Native in-app video/screen sharing and recording: deferred (TURN relay not zero-cost). Calls use Jitsi Meet or the organiser's own link.
- GitHub App (repo selection, revocation, signed webhooks): needs an owner-registered GitHub App + webhook secret.
- Office/PDF previews and co-editing: deferred; download only.
- Sandboxed embedded demo preview / remote browser: deferred; demos open in a new tab.
- Real-time transport: still polling (WebSockets flag unchanged).
- Payments: still disabled; no copy claims the platform moves money.

## Rollback

- Frontend/console: redeploy the previous release tag (release workflow) or revert the merge on `dev`/`prod`.
- API: all new endpoints are additive; reverting the code leaves the new tables unused. Do not downgrade
  migrations on production unless data in the new tables may be discarded (`alembic downgrade 043_email_outbox`).
- Admin console: delete the `remote-ai-platform-admin[-dev]` Worker or its custom domain; the API's
  ADMIN checks are unaffected.

## Release evidence (2026-10-25)

| Step | Evidence |
|---|---|
| CI on release commit | All jobs green (gitleaks, backend 578 tests / 88% coverage, frontend lint+types+tests+build, admin build, E2E Playwright, migration validation incl. concurrent migrators) |
| Postgres concurrency | `test_postgres_concurrency.py` 7 passed locally on Postgres 16 (incl. assistant double-confirm) |
| Dev | API + web report `b3c1f01`; new routes answer 401 unauthenticated; dev console origin allowed by CORS |
| Prod | API + web report `b3c1f01`; `admin.remoteaiplatform.com` serves (200, not indexable); prod CORS allows only the prod console; public jobs 200 |
| Branch protection | Restored on `prod` and `dev`, identical to the pre-work snapshot (8 / 7 required checks, admins enforced, strict) |

## Owner actions still needed

1. **Authenticated smoke tests** on dev and prod with a staff test account (sign in to the console,
   open a support ticket from the web app, reply from the console; schedule a meeting between two
   connected test users; submit and accept a deliverable). Not done: no test credentials were used.
2. **Staff access:** the console only admits users whose app role is ADMIN; sign-in uses existing
   Supabase accounts (no sign-up).
3. **Storage bucket:** the first document deliverable creates `remote-ai-platform-deliverables` via the
   S3 API. Confirm in Supabase Storage that it is **private** and set a 15 MB file limit, like the resume bucket.
4. **Meetings:** two real browsers and a restricted network test before advertising calling.
