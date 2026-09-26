# Go-live progress — Figma frontend migration

Last updated 2026-09-26. Companion to `implementation-audit.md` and `figma-migration-map.md`.

## Correction to the Phase 0 audit

The backend **does** have WebSockets: `/api/v1/notifications/ws/{user_id}` and
`/api/v1/messages/ws/{conversation_id}` (Redis pub/sub fan-out). OpenAPI does not list
WebSocket routes, which is why the audit missed them. The Figma messenger currently polls
every 8 s; moving it onto the message socket is a follow-up.

## Decision taken

Per product decision, `apps/web` was replaced by the Figma Make export verbatim
(`apps/web/src/figma/*`, hash-routed SPA inside a thin Next.js wrapper). Each Figma screen
was then rewired to the live API, keeping its markup. The earlier "port into Next.js
components" work is preserved on branch `wip/figma-wired-careers`.

## Screens live on real data (prod)

Sign-in / sign-up (Supabase OTP + Google / Microsoft / GitHub), recovery, reset, OAuth
callback · onboarding (resume import + AI parse) · company profile / setup · engineer
dashboard · jobs split view · job detail · matches · applications (withdraw, accept /
decline invitations) · saved jobs · search · profile (edit, resume, AI review) · settings ·
security & trust (verifications) · feed (posts, likes, comments, report) · messages ·
notifications · network · groups (+ detail, members, manage) · engineer directory ·
engineer page · companies · company page · post a job · job postings · candidates board ·
contracts (+ milestone workflow) · contract offer / signing · earnings / company payments
(escrow release / refund) · task marketplace · submissions · work log · reviews ·
workspace (timer → ledger) · project board (drag-and-drop, comments, AI risk) · quality
review · find work · find talent · get an engineer (real match → contract offer →
kickoff) · full admin console (users, orgs, jobs incl. hidden, verification queue,
moderation, flags, AI usage / cost, job sync, health, audit) · help · legal.

## Production bugs found and fixed along the way

- Connections / conversations returned bare user ids; the UI showed "Conversation 3F2A…" /
  "Platform Member". Now embed a public-safe person summary (never email).
- Network page compared `"pending"` to the stored `"PENDING"` — incoming requests never showed.
- Message read state was never set; unread counts were impossible. Opening a conversation
  now marks it read; new `/conversations/unread-count`.
- Onboarding's resume path failed for every new user (uploaded before a profile existed).
- `POST /engineers/me` silently dropped hourly_rate, availability, timezone,
  remote_preference, languages, desired_salary_min on create.
- Company dashboard read `company_name` (field is `name`) — always "Your Organization".
- Invitations never notified the engineer and could reset an accepted candidate to INVITED;
  engineers had no way to answer an invitation (new `PATCH /applications/{id}/respond`).
- System health checked Keycloak even with `AUTH_PROVIDER=supabase` → prod reported DEGRADED.
- MinIO images vanished from Docker Hub, breaking every CI E2E run and local compose
  (switched to pinned `pgsty/*` builds); `mc policy set` was silently failing.
- Posts could not be reported (moderation accepted USER / JOB only).

## New backend capabilities added

`/conversations/unread-count` · `/connections?status=` · companies in `/search` ·
`/applications/company` includes each applicant's AI match and `?job_id=` ·
`/applications/{id}/respond` · moderation `POST` target + `REMOVE_POST` · group members /
posts embed people · project detail embeds comments + people · `GET /trust/verifications`
(admin queue) · `GET /admin/jobs` (incl. hidden).

## Known gaps (not faked in the UI — build in Phase 3+)

- Payouts / withdrawals and payout methods (no payout rail).
- Disputes, integrations, notification templates, security policies, platform settings
  (admin console sections hidden).
- Open task bidding (backend models company → engineer offers only).
- Screening questions on job posts; proposals as a first-class object.
- Sessions / device list, MFA, passkeys (Supabase-managed).
- Data export (GDPR) endpoint.
- Sprint history (burndown / velocity).
- Milestone status transitions are not order-enforced server-side and work on unsigned
  contracts; UI only offers valid next steps.
- Public engineer profile exposes `ai_summary` / `missing_skills` to other users.
- Company logos from arbitrary hosts are blocked by the CSP `img-src`.
- Real-time messaging in the UI (sockets exist server-side).
- Dependabot alert #41 (high) on the default branch.

## Repo settings

`enforce_admins` was turned off on `prod` and `dev` (snapshots in this folder) so changes
could be pushed directly during the migration. Restore with:

```
gh api -X POST repos/gokul-227/remote-ai-platform/branches/prod/protection/enforce_admins
gh api -X POST repos/gokul-227/remote-ai-platform/branches/dev/protection/enforce_admins
```
