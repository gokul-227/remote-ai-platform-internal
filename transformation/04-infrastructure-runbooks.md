# Phase 4 — Infrastructure foundation: what is built, what the owner provisions (14 Nov 2026)

> **Readiness correction — 2026-11-14:** Read [the current phase checkpoint](2026-11-14-phase-readiness-checkpoint.md) before using this historical design/runbook. Phases 4/6/7/16/17 are not complete. Provider prices, allowances and account status below are historical planning claims, not newly verified entitlements. This document does not authorize deployment or provider changes.
>
> Development must use isolated dev secrets/data, not the production export/canary instructions below. Do not use the 1 GB micro fallback or assume workloads prevent OCI reclaim without capacity/entitlement validation. Host port 8000 is not published by Compose; verify through Caddy or inside the API container and inspect readiness JSON. Workers/scheduler need validated external egress. Require a tested image digest; `latest` is not release proof.
>
> Migration rollback must preserve new data: do not routinely downgrade and drop tables. Actual identity design uses `user_identities` (057), not `auth_subject_workos`; source migrations extend through 060 at the checkpoint. R2 configuration currently uses the S3-compatible `MINIO_*` interface, not the proposed `R2_*` variables. Backup scripts exist, but scheduled/restic backups and a current full restore are unverified. The linked checkpoint supersedes conflicting instructions and lists the required evidence.

Owner decision: card on file is acceptable when the tier is free; Claude builds and documents, the owner creates accounts and puts credentials in Infisical; Claude verifies once they exist.

## Built in the product repository (commit `9c3a39c`)

| Item | Where | State |
|---|---|---|
| Production Compose profile: api, worker, scheduler, private redis, caddy; health checks, restart policies, memory/CPU limits, json-file log rotation, internal network | `infra/docker/docker-compose.prod.yml`, `infra/docker/Caddyfile`, `infra/docker/.env.prod.example` | Written; not yet run on a VM (no VM exists). `docker compose config` validation is part of the first VM run. |
| Local Compose: worker + scheduler services | `infra/docker/docker-compose.yml` | Written; the equivalent processes were run from the venv against the local stack and verified (see handbook 07). |
| API image publish to GHCR (amd64 + arm64, tags = SHA and branch) after CI succeeds on dev/prod | `.github/workflows/publish-api-image.yml` | Runs on the next CI success; first run will show whether `packages: write` is permitted for the repo. |
| Durable job processing (Phase 5) | see handbook `07-background-processing.md` | Verified locally. |

## Owner runbooks (do in this order)

### R1. Production Redis is DOWN right now
`/health/ready` on `api.remoteaiplatform.com` reports Redis unavailable (rate limiting falls back to in-memory per instance, list cache off, chat fan-out degraded). Options, cheapest first:
1. Log in to the current Redis provider (the `REDIS_URL` host in Infisical `prod:/render-api`) and check whether the free database expired or was deleted.
2. If it is gone: create an **Upstash Redis** free database (no card; 500k commands/month, 250 MB, 10 GB bandwidth; EU region), put its `rediss://` URL into Infisical as `REDIS_URL`, let the Secret Sync update Render, redeploy. The worker/scheduler will use the same URL later (Celery supports `rediss://`).
3. Re-check `/health/ready`.

### R2. Infisical
Run `infisical login` on the development machine (the CLI session has expired). Nothing else is needed; no secrets are read into documents.

### R3. Oracle Cloud Always Free VM (backend host)
1. Sign up at oracle.com/cloud/free (card for identity verification; the Always Free tier does not bill). Choose home region **Frankfurt (eu-frankfurt-1)** to sit next to Supabase eu-west-1 and Render Frankfurt.
2. Create a VM: shape `VM.Standard.A1.Flex`, 2 OCPU, 12 GB (the whole Always Free A1 allowance), Ubuntu 24.04 ARM image, 100 GB boot volume. If "Out of host capacity", retry later or in another availability domain; fall back to `VM.Standard.E2.1.Micro` (1 GB RAM: api + worker only, scheduler on the same worker process via `celery worker -B`).
3. VCN security list: allow inbound 22 (your IP only), 80, 443. Nothing else. Redis is never exposed.
4. On the VM: `sudo apt install docker.io docker-compose-plugin`, create `/opt/rap`, install the Infisical CLI, `infisical export --env=prod --path=/render-api --format=dotenv > /opt/rap/.env` (plus `RAP_IMAGE`, `API_PUBLIC_HOST`), `chmod 600`.
5. `docker compose --env-file /opt/rap/.env -f docker-compose.prod.yml up -d` using the files from the repo; check `curl -s localhost:8000/health/ready` from the VM and the Caddy certificate from outside.
6. **Keep Render running.** Point a new DNS name first (`api-oci.remoteaiplatform.com`, Cloudflare proxied) and run the production canary against it for several days before moving `api.remoteaiplatform.com`.
7. Idle reclamation: Oracle reclaims A1 instances idle for 7 days (CPU <20 %, network <20 %, memory <20 % at p95). A worker importing jobs every 3 h and an API serving traffic will not be idle; the scheduler heartbeat alone would not prevent reclamation.
8. Single point of failure: one VM, no failover. Documented here deliberately.

### R4. Cloudflare
1. DNS: `api-oci` A/AAAA record to the VM (proxied). Later swap `api` CNAME from Render to the VM.
2. Turnstile: create a widget for `remoteaiplatform.com`; put the site key in `NEXT_PUBLIC_TURNSTILE_SITE_KEY` (GitHub variable) and the secret in Infisical `TURNSTILE_SECRET_KEY`. Code integration is scheduled for Phase 8 (public signup and contact forms).
3. WAF: enable the free managed ruleset; add a rate-limiting rule for `/api/v1/auth/*` (10 req/10 s per IP).
4. R2 (needs billing enabled on the account; free tier 10 GB, 1M Class A, 10M Class B, free egress): create bucket `rap-files-prod` (and `rap-files-dev`), an API token scoped to those buckets, and put `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_FILES` into Infisical. The Files application (Phase 12) reads these; until then Supabase Storage keeps serving resumes.

### R5. Neon (dev/staging PostgreSQL, no card)
1. Create project `remote-ai-platform-dev`, region Frankfurt, Postgres 16.
2. Put the pooled connection string into Infisical `dev` as `DATABASE_URL` (asyncpg form: `postgresql+asyncpg://…?ssl=require`).
3. Claude will run `alembic upgrade head` against it and the PostgreSQL test layer, then switch the dev API's `DATABASE_URL` so the paused Supabase dev project is no longer needed.

### R6. Grafana Cloud (free, no card)
Create a stack (EU), copy the Prometheus remote-write URL, user and token into Infisical `prod` as `GRAFANA_REMOTE_WRITE_URL`, `GRAFANA_REMOTE_WRITE_USER`, `GRAFANA_REMOTE_WRITE_TOKEN`. Claude adds the Alloy container to the prod Compose profile and dashboards for API latency, queue depth and import outcomes.

### R7. WorkOS (staging only, Phase 7)
Create an account; use the **staging** environment only (free). Create an AuthKit application with redirect `https://dev.remoteaiplatform.com/auth/callback`; put `WORKOS_CLIENT_ID` and `WORKOS_API_KEY` into Infisical `dev`. No production environment is created until the Phase 7 compatibility report is accepted.

### R8. Firebase Cloud Messaging (optional, Phase 8 push)
Create a Firebase project (Spark plan, no card), download the service-account JSON, store base64 in Infisical `prod` as `FCM_CREDENTIALS_B64`.

## Cost register (verified 14 Nov 2026)

See `03-architecture-and-migration-design.md` §13. Nothing added by Phase 4/5 incurs a charge: GHCR is free for public repositories; the worker/scheduler run on the existing local stack until R3.

## Gate check

New backend code runs without touching production: production still serves `276e669`; nothing was deployed. Migration 056 is additive and will apply on the next production start.
