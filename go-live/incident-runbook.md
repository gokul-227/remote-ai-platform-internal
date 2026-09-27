# Incident runbook

Owner and on-call: gokul-227 (solo). Alerts arrive as GitHub e-mails from **Production canary** (every 30 min) and
failed deploy runs; Sentry (if alert rules are set in its dashboard).

## Severity

| Sev | Meaning | Response |
|---|---|---|
| 1 | Sign-in, job list or site down; data exposed or lost; suspected breach | Now. Contain first (roll back / disable), then diagnose |
| 2 | A core journey broken (apply, messaging, profile, resume) or AI down with no manual path | Same day |
| 3 | Degraded (slow, AI allowance exhausted, ingestion stale < 3 days) | Next working day |

## First checks

1. `GET /health/version` (API) and `/api/version` (web): which SHA is live?
2. `GET /health/ready`: database / Redis. As an admin: `GET /health/dependencies` (storage, AI last success/failure).
3. GitHub Actions: last CI, deploy, sync and canary runs.
4. Render logs (request_id correlates API log lines; clients may send X-Request-ID).

## Common cases

- **Redis down** → API keeps serving (in-memory rate limits, degraded health). Sev 3 unless prolonged.
- **Database unreachable** → Sev 1. Supabase status page; free projects pause after inactivity → restore project.
- **Deploy broke something** → roll back: Render → Deploys → redeploy previous; Cloudflare → Workers → Deployments →
  rollback. Migrations are additive, so the previous app version runs on the newer schema.
- **Migration refused at startup** (e.g. 038 duplicate applications) → Render keeps the old instance; fix data, redeploy.
- **Ingestion stale** (canary: newest job > 48 h) → run `scheduled-job-sync.yml` manually; check `/admin` sync log.
- **AI failing** → users get the manual path; check provider status and key; `/health/dependencies` last_failure_at.
- **Leaked secret** → rotate at provider first, update Render/GitHub/Infisical, revoke old key, record in SEC-01 log.
- **Data loss** → stop writes if possible; follow `backup-and-restore.md`; never restore over new writes without a
  written loss/reconciliation decision.

## Rollback triggers (decide before each release)

Security regression; failed critical journey (sign-in, job list, apply, messaging); error rate or p95 latency clearly
above the previous release; schema incompatibility; quota exhaustion; integrity mismatch.

## After

Write a short blameless note here: timeline, impact, cause, fix, the test that now covers it.
