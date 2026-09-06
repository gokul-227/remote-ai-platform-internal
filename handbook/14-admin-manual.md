# Admin Manual

A working guide to the platform's admin console, based on what `apps/web/src/app/admin/*` and the
backend's `admin` domain (`apps/api/app/domains/admin`) actually implement today. This assumes slightly
more technical comfort than the engineer/company manuals — you're the operator, not a first-time user.

## Getting in

Sign in with an account whose role is **ADMIN**. The admin console lives at `/admin/dashboard` and is
protected — only ADMIN-role accounts can load it; anyone else is redirected away. There's no separate
admin sign-up flow: an admin account is created by promoting an existing user's role (see **User
Management** below), which itself requires an existing admin to do it (or direct database access on the
very first admin account).

## The dashboard, top to bottom

Everything below is what actually loads on `/admin/dashboard` on a normal visit.

### Header bar
Shows a live **"N/M services operational"** badge (see System Health below) and quick links to **Users**,
**Jobs**, and the raw **API Docs** (FastAPI's interactive OpenAPI docs at `/api/docs`).

### Platform Overview (top metric cards)
Four live counts, pulled from `GET /admin/stats`:
- **Registered Professionals** — total engineer accounts
- **Organizations** — total company accounts
- **Active Positions** — total open job postings
- **Total Platform Users** — everyone, across all roles

### AI Token Cost Monitoring
Pulled from `GET /admin/ai-usage`, this section answers "what is the AI actually costing us": total LLM
API calls, total tokens consumed (broken into prompt vs. completion tokens), and an **estimated cost in
USD**. Below that, a breakdown of calls **by feature** (e.g., resume parsing vs. job enrichment vs.
matching) so you can see which AI feature is driving usage. This is genuinely computed from logged
activity, not a placeholder.

### System Health
Pulled from `GET /admin/health/details`, this lists each core service (Auth/Keycloak, Jobs API,
Professionals API, PostgreSQL, Redis, and others as configured) with a live status
(**OPERATIONAL** / an error state) and latency in milliseconds, plus a "last checked" timestamp. If this
call fails or times out, the dashboard falls back to showing every service as **UNKNOWN** rather than
silently hiding the panel — so an "UNKNOWN" row usually means the health-check call itself is having
trouble, worth investigating directly.

### Moderation Queue
Shows every **open** moderation report (a user- or system-flagged job or account) with the reason given.
For each, you can:
- **Take action** — resolves the report as either "hide the job" or "suspend the user," depending on
  what was reported.
- **Dismiss** — resolves the report with no action taken.

An empty queue shows a plain "No open reports — the queue is clear" state; this is a real, working
review queue, not a mockup.

### Job Source Sync Status
A table of every external job source (RemoteOK, Arbeitnow, Remotive, USAJobs, The Muse) showing when it
last synced, whether that run **SUCCEEDED** or **FAILED**, and how many jobs currently in the system came
from that source. This reflects the Celery beat schedule that re-runs all sources automatically every 6
hours — this table is your primary way to notice a source has silently started failing (e.g., an
external API changed its response format or started rate-limiting). The **Recent Sync Runs** panel in the
sidebar shows the last several runs with how many jobs each one inserted.

### Audit Log
A running, append-only log of sensitive admin actions — user status changes, role changes, deletions,
job status changes/deletions, and bulk text-recleaning runs — each with who did it, what changed, and
when. This is the accountability trail if you ever need to answer "who changed this and when." The
richer **audit-events** view (`GET /admin/audit-events`, filterable by action/resource type) backs this
same data if you need to search further than what's shown on the dashboard.

### Platform Stack (info panel)
A static reference panel confirming the current stack at a glance: version, auth mechanism, database,
API framework, and frontend framework. Useful for quickly telling a support/dev conversation what's
actually running in production without digging through docs.

## User Management (`/admin/users`)

From here you can, for any account:
- **Suspend or reactivate** it (toggles `is_active` — a suspended user can't log in, but their data and
  history are preserved).
- **Change their role**, including promoting someone to ADMIN. This is a separate, admin-only action from
  the self-service "pick Engineer or Company" role choice a new user makes at sign-up — an admin is the
  only one who can grant ADMIN or otherwise override a user's role after the fact.
- **Permanently delete** the account. This is irreversible: because every table referencing a user is set
  to cascade-delete, removing a user also removes their profile, applications, messages, notifications,
  and everything else tied to them. Use suspension for routine account issues; reserve deletion for
  cleaning up test/demo accounts or a definitive, requested account closure. You cannot delete your own
  admin account this way (the system blocks it as a safety rail).

## Job Management (`/admin/jobs`)

From here you can, for any job posting:
- **Pause or reactivate** it (toggles whether it shows up in search/matching without deleting the row).
- **Permanently delete** it — distinct from pausing; use this for cleaning up test/demo listings that
  shouldn't exist in production, not for routine "this role isn't hiring anymore" pauses.
- Trigger a **bulk text re-clean**, which re-runs the same HTML-entity/mojibake cleanup that every newly
  synced job automatically gets, but applied retroactively to older rows whose source listing has since
  disappeared from the aggregator feed (and so would otherwise never get cleaned again). This is a
  maintenance action, not something you'd run routinely.

## What the admin console does not (yet) do

Being direct about the edges, so you don't go looking for something that isn't there:
- There's no billing/plan management screen, because there's no billing system yet (see the Product
  Overview document) — nothing to configure there today.
- There's no feature-flag management UI in the admin console; if a feature needs to be toggled off,
  that's a code/config change today, not an admin-console action.
- There's no bulk-export or reporting/analytics-builder tool beyond the stats already on the dashboard —
  what you see there is the full extent of platform analytics today.
