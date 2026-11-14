# Background processing (worker, scheduler, job imports)

Added 14 November 2026 (program phase 5). Code: `apps/api/app/worker/` (Celery) and
`apps/api/app/domains/tasks/` (PostgreSQL records + staff API). Console: *Jobs → Background processing*.

## Shape

```
Celery Beat ──every 60 s──▶ tasks.tick_schedule ──▶ job_sources (cron, caps, permission) ──▶ task_runs(QUEUED) ──▶ Redis ──▶ worker ──▶ JobService.sync_all_job_sources(source)
            ──every 10 s──▶ tasks.dispatch_outbox ──▶ outbox_events → task_runs
            ──every 5 min─▶ tasks.requeue_stale   (runs stuck QUEUED/STARTED/RETRYING > 30 min are re-sent; a lost STARTED run counts an attempt)
            ──03:20 UTC───▶ jobs.maintain_catalogue_scheduled (retention, size budget)
            ──every 60 s──▶ notifications.flush_email_outbox_scheduled (email_outbox dispatch)
            ──04:05 UTC───▶ tasks.prune_runs (terminal runs older than TASK_RUN_RETENTION_DAYS=30)
```

- **Redis is a transport.** The record of truth is `task_runs` (status QUEUED → STARTED → SUCCEEDED | RETRYING → … → DEAD).
- **Idempotency.** `idempotency_key` is unique; a non-terminal run with the same key makes a new enqueue a no-op. Scheduled imports use `import:<SOURCE>:<cron slot>`, so a second scheduler tick in the same slot does nothing.
- **Retries.** Backoff 1 m, 5 m, 15 m, 1 h; after `max_attempts` (5 for scheduled imports, 1 for manual imports, 3 for dead-letter retries) the run is DEAD and a `dead_letters` row appears in the console with a Retry button (reason required, audited).
- **Exactly one scheduler.** Beat takes `pg_try_advisory_lock(7412908321)` on a dedicated connection and exits with code 3 if another Beat holds it. Compose restarts it; the lock holder keeps running.
- **Heartbeats.** Workers and the scheduler write `worker_heartbeats`; `/health/ready` adds `services.worker` / `services.scheduler` (HEALTHY if a heartbeat is younger than `WORKER_HEARTBEAT_STALE_SECONDS=180`, otherwise DEGRADED; UNKNOWN before any heartbeat). These never change the overall readiness verdict.
- **Source policy.** `job_sources.permission_confirmed=false` blocks scheduled and manual imports for that source (409 in the console). `source_policy.py` gates inside the aggregators still apply.
- **Checkpoints.** `job_import_checkpoints` holds last success, last error category and consecutive failures per source.

## Running

Local: `docker compose -f infra/docker/docker-compose.yml up -d worker scheduler` (same image as the API) or from the venv:

```bash
cd apps/api
.venv/bin/celery -A app.worker.celery_app worker -Q default,imports --concurrency 2
.venv/bin/celery -A app.worker.celery_app beat
```

Production (single Docker VM): `infra/docker/docker-compose.prod.yml` + `infra/docker/Caddyfile`; env from Infisical `prod:/render-api` written to `/opt/rap/.env` (names in `infra/docker/.env.prod.example`). Image from GHCR (`.github/workflows/publish-api-image.yml`, tags = commit SHA and branch). Redis is on the internal network only; Caddy publishes 80/443 with its own TLS behind Cloudflare.

Settings: `TASKS_ALWAYS_EAGER` (run inline, no broker), `TASKS_DISPATCH_DISABLED` (record runs, never contact the broker; the console shows a banner), `WORKER_HEARTBEAT_STALE_SECONDS`, `TASK_RUN_RETENTION_DAYS`.

## Cut-over from GitHub Actions

1. Deploy worker + scheduler with the production database URL (Phase 4 VM).
2. Watch *Background processing* for one full day: every source SUCCEEDED or honestly RETRYING/DEAD, heartbeats fresh.
3. Set repository variable `JOB_SYNC_GITHUB_ENABLED=false` (the workflow stays; rollback = delete the variable).
4. Only then consider removing `PROD_DATABASE_URL` from GitHub secrets.

## Verification on 14 Nov 2026 (local, dev database and Redis db 5)

- Beat: "Scheduler lock acquired"; first `tick_schedule` created 15 `job_sources` rows and 15 import runs in one tick.
- Worker: 14 sources SUCCEEDED with a 20-posting cap; THEMUSE (no API key configured) went RETRYING with backoff and will dead-letter after 5 attempts.
- A bug in the first run (`consecutive_failures` None on a new checkpoint) was fixed; the worker was killed with SIGKILL and replaced, the new worker's heartbeat appeared and the old one aged out.
- Tests: `tests/test_background_tasks.py` (9) cover idempotency, backoff/dead letter, stale requeue, outbox dispatch, cron slots, heartbeats, the import task lifecycle, staff endpoints and readiness. Migration 056 round-tripped on PostgreSQL.

## Known limitations

- Celery `inspect ping` health check needs the worker name to match `celery@<hostname>`; the Compose healthcheck uses that default.
- `requeue_stale` resends with the same task id; Celery treats it as a new message, so a still-running duplicate on a hung worker is possible once per 30 minutes until the hung process dies (acks-late + visibility timeout 1 h bound it).
- Rate limits per source are recorded (`rate_limit_per_minute`) but enforcement is inside each aggregator's own pacing, not a shared limiter yet.
