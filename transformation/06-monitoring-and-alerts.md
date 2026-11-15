# 06 — Monitoring and alerts (Slack is the single point of contact)

Every alert about the Remote AI Platform lands in Slack. All tooling is free: a Slack bot token
(`chat.postMessage`), self-hosted GlitchTip and Uptime Kuma on the Oracle VM, the free UptimeRobot
account, and GitHub Actions. Code: `apps/api/app/services/slack.py`, `apps/api/app/worker/ops.py`,
`infra/docker/docker-compose.ops.yml`, `infra/ops/` (branch `feat/ops-slack`).

## Channels

| Channel (setting) | What arrives |
| --- | --- |
| #alerts-critical (`SLACK_CHANNEL_ALERTS_CRITICAL`) | outages, stale worker/scheduler, dead letters, failed backups, incidents, uptime |
| #alerts-errors (`SLACK_CHANNEL_ALERTS_ERRORS`) | GlitchTip error alerts (api, web, admin) |
| #deploys (`SLACK_CHANNEL_DEPLOYS`) | deploy workflow start / success / failure |
| #backups (`SLACK_CHANNEL_BACKUPS`) | every backup / restore-verify report, missed backups |
| #job-imports (`SLACK_CHANNEL_JOB_IMPORTS`) | import run failures, daily import summary |
| #signups (`SLACK_CHANNEL_SIGNUPS`) | each new account (email domain only), sign-up spikes |
| #security (`SLACK_CHANNEL_SECURITY`) | role/staff/flag changes, auth-failure bursts, identity conflicts, sign-up spikes |
| #costs (`SLACK_CHANNEL_COSTS`) | free-tier usage at 70 % / 90 % |

Channel ids and the bot token live in Infisical prod `/automation`. Messages from non-production
environments are prefixed `[<APP_ENV>]`.

## Alert catalogue

| Alert | Source | Channel | Threshold / trigger | Repeat behaviour |
| --- | --- | --- | --- | --- |
| Health component DOWN/DEGRADED | `GET /health/ready` and `/health/dependencies` (polled every 60 s by Uptime Kuma, 5 min by UptimeRobot) | critical | any component DOWN or DEGRADED (database, redis, worker, scheduler, ai, storage) | once per component+state per `SLACK_DEDUP_SECONDS` (900 s) per API process |
| No fresh worker/scheduler heartbeat | scheduler heartbeat thread (`worker/ops.check_liveness`, every 30 s) | critical | newest heartbeat older than `WORKER_HEARTBEAT_STALE_SECONDS` (180 s) | every 15 min while stale |
| Task moved to dead letters | `TaskService.mark_failed` | critical | run reached `max_attempts` | once per task name per 15 min per process |
| Dead letter retried by staff | audit `background.dead_letter.retry` | critical | every retry | every event |
| Incident opened / resolved | audit `admin.incident.create` / update to RESOLVED | critical | every event | every event |
| Backup / restore-verify succeeded | `POST /ops/backups/report` | backups | every report | every event |
| Backup / restore-verify failed | `POST /ops/backups/report` | critical + backups | `status=failed` | every event |
| Backup missed / never reported | `ops.check_backups` 08:00 UTC | backups + critical | no successful backup in 26 h | once a day |
| Restore unverified | `ops.check_backups` 08:00 UTC | backups | no verified restore in 35 days | once a day |
| Job import failed | `jobs.import_source` | job-imports | run RETRYING or DEAD | once per source+state per 15 min |
| Daily import summary | `ops.daily_import_summary` 07:00 UTC | job-imports | daily | once a day |
| New sign-up | `UserRepository.create` | signups | each account | every event |
| Sign-up spike | `UserRepository.create` | signups + security | > `SLACK_SIGNUP_SPIKE_PER_HOUR` (25) in the last hour | once per clock hour |
| Member role / staff roles / bootstrap / account status / sessions revoked | audit `admin.user.role`, `staff.roles.update`, `admin.bootstrap`, `admin.user.status`, `admin.user.revoke_sessions` | security | every event | every event |
| Feature flag changed | audit `feature_flag.update` | security | every event | every event |
| Rejected sign-in token burst | `get_current_user` 401s | security | >= `SLACK_AUTH_FAILURE_BURST` (30) in 10 min per API process | once per 15 min |
| Identity conflict refused | `AuthService` 409 (email claimed by another identity / unverified email) | security | every event | once per account per 15 min |
| Free-tier usage | `ops.cost_guard` every 6 h | costs | >= `COST_WARN_PERCENT` (70) warning, >= `COST_CRITICAL_PERCENT` (90) critical, of: PostgreSQL size vs `COST_DB_LIMIT_MB` (20 GB); object storage vs `COST_STORAGE_LIMIT_GB` (20 GB, OCI S3 API); VM disk (`COST_DISK_PATH`); AI tokens today vs `AI_GLOBAL_DAILY_TOKENS`; emails today/month vs `COST_EMAIL_DAILY_LIMIT` (100) / `COST_EMAIL_MONTHLY_LIMIT` (3000) | once per metric+level per day |
| Application errors | GlitchTip projects api/web/admin, alert "Slack #alerts-errors" | errors | 1 event in 1 min (new issue) | GlitchTip groups by issue |
| Uptime (on-VM) | Uptime Kuma: API ready, Web, Admin console | critical | 3 failed checks (60 s interval, 2 retries); HTTP >= 400 or timeout | on down and on up |
| TLS expiry | Uptime Kuma (`expiryNotification`) | critical | 21 / 14 / 7 days before expiry | once per threshold |
| Uptime (off-VM) | UptimeRobot (6 monitors, 5 min) | **email only (see gap 1)** | down/up | on down and on up |
| Deploys | `.github/workflows/deploy-notify.yml` | deploys | start, success, failure, cancel of "Deploy frontend (Cloudflare)", "Deploy frontend dev (Cloudflare)", "Publish API image (GHCR)" | every run |

## How to silence

* **Everything from the API/worker**: unset `SLACK_BOT_TOKEN` (or one `SLACK_CHANNEL_*` to silence
  one channel) in the runtime env and restart. The notifier becomes a no-op.
* **Noisy repeats**: raise `SLACK_DEDUP_SECONDS` (900) or lower `SLACK_MAX_PER_MINUTE` (20 per channel).
* **Thresholds**: `SLACK_SIGNUP_SPIKE_PER_HOUR`, `SLACK_AUTH_FAILURE_BURST`, `COST_*`,
  `WORKER_HEARTBEAT_STALE_SECONDS`.
* **A scheduled check**: remove `daily-import-summary`, `check-backups` or `cost-guard` from
  `beat_schedule` in `app/worker/celery_app.py`.
* **GlitchTip**: Project -> Settings -> Alerts -> disable "Slack #alerts-errors"; or mute/resolve the issue.
* **Uptime Kuma**: pause the monitor, use Maintenance windows, or disable the Slack notification.
* **UptimeRobot**: pause the monitor in the dashboard.
* **Deploy messages**: delete repo variable `SLACK_CHANNEL_DEPLOYS` (the job prints "skipped").
* **Personal quiet hours**: mute the channel in Slack.

## Access

* `ssh -i ~/.ssh/oci_rap -L 8001:127.0.0.1:8001 -L 3001:127.0.0.1:3001 opc@79.76.118.213`
  -> GlitchTip `http://localhost:8001`, Uptime Kuma `http://localhost:3001`; user
  `gokulraj22797@gmail.com`, passwords `GLITCHTIP_ADMIN_PASSWORD` / `UPTIME_KUMA_ADMIN_PASSWORD`.
* Infisical prod `/automation`: `SLACK_BOT_TOKEN`, `SLACK_CHANNEL_*`, `UPTIME_HOOK_TOKEN`,
  `WEB_PUSH_VAPID_PUBLIC_KEY`, `WEB_PUSH_VAPID_PRIVATE_KEY`, `GLITCHTIP_SECRET_KEY`,
  `GLITCHTIP_DB_PASSWORD`, `GLITCHTIP_ADMIN_PASSWORD`, `UPTIME_KUMA_ADMIN_PASSWORD`, `GLITCHTIP_DSN`,
  `GLITCHTIP_DSN_WEB`, `GLITCHTIP_DSN_ADMIN`. GitHub: secret `SLACK_BOT_TOKEN`, variable `SLACK_CHANNEL_DEPLOYS`.

## Known gaps / owner actions

1. **Off-VM uptime -> Slack.** UptimeRobot free refuses webhook alert contacts ("This integration is
   not available for current user"), and its Slack contact needs a Slack *incoming webhook* URL, which
   only the workspace owner can create (Slack app -> Incoming Webhooks -> Add to #alerts-critical; free).
   Then UptimeRobot -> Integrations -> Slack, attach to all monitors. Until then UptimeRobot alerts go
   to the existing email contact. `POST /api/v1/ops/uptime-hook` (token `UPTIME_HOOK_TOKEN`) is ready
   for any monitor that can send webhooks.
2. **Browser error DSNs** need a public GlitchTip ingest hostname (route through Caddy) before web/admin
   can report; the API DSN works on the VM network now.
3. **Worker egress.** In `docker-compose.prod.yml` worker and scheduler are only on the `internal: true`
   network, which has no internet: they cannot reach Slack (nor job boards / AI providers). Add them to
   `edge` or an egress network.
4. Dedup and rate limits are per process (in memory); a restart can repeat an alert once.
5. Audit-driven alerts post when the audit row is flushed; a request that then rolls back can alert on
   a change that did not persist (audit is written last, so this is rare).
