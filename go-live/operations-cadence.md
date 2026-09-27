# Post-launch operations (Phase 16)

- **Daily (first month):** canary e-mails, Sentry new issues, failed Actions runs, support inbox.
- **Weekly:** job ingestion health (newest job age, sync log), AI usage vs daily cap (`/admin` AI usage), signups →
  profiles → applications funnel (`/analytics/summary`), open reports.
- **Monthly:** restore drill from the latest backup (record in `backup-and-restore.md`); access review (GitHub
  collaborators, Render/Supabase/Cloudflare members, admins in the app); dependency updates (Dependabot, npm/pip
  audit); quota check against free-tier limits (Render hours, Supabase 500 MB DB / 1 GB storage / egress, Workers
  requests) — warn at 50 %, act at 75 %, escalate at 90 %.
- **Every release:** CI green (incl. secret scan), deploy smoke green, `/health/version` = tested SHA.
- **Incidents:** `incident-runbook.md`; blameless note within a week, with the test that now covers it.
- **Secrets:** rotate on any exposure immediately; otherwise yearly. Never paste keys into chats or issues.
