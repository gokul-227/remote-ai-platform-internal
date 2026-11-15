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

## R3b. VM data layer: PostgreSQL 16, nightly backups, rehearsal copy (done 15 Nov 2026)

Product branch `infra/vm-postgres` (commit `305ed67`, not pushed). VM: OCI A1 Oracle Linux 9.8 aarch64, 4 OCPU, 22 GB, `opc@79.76.118.213`. Nothing serves public traffic; DNS unchanged; Render/Supabase untouched (source read with `pg_dump` and `SELECT` only).

**Disk.** `lsblk`: boot volume 46.6 GB, VG `ocivolume` fully allocated (root 29.5 GB xfs + `/var/oled` 15 GB). There is no free extent to grow root without resizing the boot volume in OCI (Always Free allows up to 200 GB total); not done. Root after setup: 17 GB used / 13 GB free.

**Secrets (Infisical prod `/automation`, shared):** `VM_POSTGRES_PASSWORD` (40 random alphanumerics), `VM_DATABASE_URL` (`postgresql+asyncpg://rap:…@postgres:5432/remote_ai_platform`, valid inside the compose network).

**Files on the VM:** `/opt/rap/repo` (rsync of the product working tree, without web/admin/node_modules/.venv/.git), `/opt/rap/.env` (opc, 600: `infisical export --env prod --path /render-api` + `POSTGRES_PASSWORD`, `RAP_IMAGE=rap-api:local`, `API_PUBLIC_HOST=api-oci.remoteaiplatform.com`), `/opt/rap/backup.env` (root, 600; names in `ops/backup/vm/backup.env.example`), `/opt/rap/venv` (python3.12 + boto3), `/var/backups/rap` (root 700, newest 3 local archives).

```bash
# from the Mac (repo root)
rsync -az --delete -e "ssh -i ~/.ssh/oci_rap" --exclude node_modules --exclude .venv --exclude .git \
  --exclude __pycache__ --exclude .next --exclude .claude --exclude apps/web --exclude apps/admin ./ opc@79.76.118.213:/opt/rap/repo/
# on the VM
sudo dnf -y install python3.12 python3.12-pip rsync && sudo python3.12 -m venv /opt/rap/venv && sudo /opt/rap/venv/bin/pip install boto3
cd /opt/rap/repo/infra/docker
C="docker compose --env-file /opt/rap/.env -f docker-compose.prod.yml"
$C config -q && $C up -d postgres redis          # api/worker/scheduler/caddy NOT started
docker exec rap-prod-postgres-1 psql -U rap -d remote_ai_platform -c 'show shared_buffers'
sudo ss -ltnp | grep -E ':5432|:6379'            # must print nothing: no host ports
cd /opt/rap/repo/apps/api && docker build --target production -t rap-api:local .   # GHCR image has no arm64 manifest
```

PostgreSQL 16.16, config `infra/docker/postgres/postgresql.conf` (shared_buffers 4GB, effective_cache_size 12GB, work_mem 32MB, maintenance_work_mem 1GB, max_connections 100, random_page_cost 1.1, parallel 4/2), data checksums on, volume `rap-prod_postgres_data`, network `rap-prod_internal` (internal: true). Redis stays local in the compose (Celery broker polling would exhaust the free Upstash quota); Render keeps Upstash.

**Rehearsal copy.** Production Supabase is **PostgreSQL 17.6**: `pg_dump` 16 refuses a 17 server, so the dump was made with `postgres:17` as plain SQL and loaded into 16 after removing `SET transaction_timeout` (17-only) and `CREATE SCHEMA public`/`COMMENT ON SCHEMA public`:

```bash
# Mac -> VM over stdin; the URL is written to a 600 file and deleted afterwards
infisical secrets get DATABASE_URL --projectId 237deba5-… --env prod --path /render-api --plain --silent | sed 's/+asyncpg//' | ssh … 'cat > /var/tmp/rehearsal/src.url'
docker run --rm -e SRC="$(cat src.url)" -v $W:/w postgres:17 pg_dump --format=plain --no-owner --no-privileges --schema=public --file=/w/db.sql "$SRC"
/opt/rap/venv/bin/python ops/backup/dumpcounts.py < db.sql | LC_ALL=C sort > expected.tsv
sed -e '/^SET transaction_timeout/d' -e '/^CREATE SCHEMA public;/d' -e '/^COMMENT ON SCHEMA public /d' db.sql > db16.sql
docker exec -i rap-prod-postgres-1 psql -U rap -d remote_ai_platform -v ON_ERROR_STOP=1 --single-transaction < db16.sql
$C run --rm --no-deps -T api alembic upgrade head   # (docker run --env-file keeps dotenv quotes; use compose)
```

Results: source 453 MB on disk (Supabase incl. system), dump 266 MB / 74 s, restore 26 s, VM DB 230 MB. Row counts: **83/83 tables equal, 54,033 rows** (expected from the dump snapshot vs `count(*)` on the VM). Alembic `060_search_trgm` → `068_backup_runs` in 21 s; all 83 tables unchanged afterwards, 10 new empty tables (`backup_runs`, `feature_flag_overrides`, `file_activity`, `file_nodes`, `file_shares`, `file_stars`, `file_versions`, `incidents`, `push_subscriptions`, `staff_roles`). `users.auth_subject` not touched. Source extensions not recreated on the VM (Supabase-only): `pg_cron`, `pg_net`, `pgcrypto`, `supabase_vault`; no public DDL references them. RLS is enabled on 83 tables with no policies (owner bypass, same as today). Plaintext dump deleted after the run.

**Nightly backup.**

```bash
sudo install -m 600 -o root -g root /dev/stdin /opt/rap/backup.env   # piped from Infisical on the Mac, see backup.env.example
sudo cp /opt/rap/repo/ops/backup/vm/rap-backup.{service,timer} /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now rap-backup.timer   # 02:30 UTC, Persistent
sudo systemctl start rap-backup.service && journalctl -u rap-backup.service -n 30
```

Bucket `rap-prod-backups` (created by `offsite.py ensure-bucket` through the S3 API; private: anonymous GET returns 404), prefix `db/`, newest 14 kept by script pruning (OCI's S3 API has no lifecycle). Note `MINIO_ENDPOINT` in Infisical has no scheme: `backup.env` uses `https://` + it. First run: `remote-ai-platform-20261115T143708Z.tar.enc`, 85 MB, 93 tables, 5 objects, 27 s. `BACKUP_ENVIRONMENT=rehearsal` until cut-over (then `production`). The report POST to `https://api.remoteaiplatform.com` fails (HTTP error) until production runs migration 068; the run is unaffected.

**Restore drill** (archive fetched from the bucket, scratch DB `restore_drill`, objects into temporary OCI buckets with the source bucket names, never Supabase; all removed afterwards):

```bash
python ops/backup/vm/offsite.py fetch <name> $D
docker exec rap-prod-postgres-1 psql -U rap -d postgres -c 'CREATE DATABASE restore_drill'
S3_ENDPOINT=$ARCHIVE_S3_ENDPOINT … TARGET_DATABASE_URL=postgresql://rap:…@postgres:5432/restore_drill BACKUP_FILE=$D/<name> DRY_RUN=1 ops/backup/restore.sh </dev/null
CONFIRM_TARGET='<identity>' … ops/backup/restore.sh </dev/null
```

Fetch 2 s; restore verified in 18 s: archive + 9 inner checksums ok, 93 tables row counts ok, 5 objects ok, references ok. Found and fixed: `restore.sh` with `PG_IMAGE` created only the first extension (`docker run -i` read the rest of the extension list).

**Open for the coordinator:** API image for arm64 (GHCR only has amd64; built locally as `rap-api:local`), boot-volume resize if more than ~13 GB free is needed, production migration to 068 (for backup reporting), switch `BACKUP_ENVIRONMENT=production` and re-dump from Supabase at the cut-over (the VM copy is a point-in-time rehearsal and will go stale; restore into an emptied DB).
## R3c. Websites off Cloudflare Workers onto the VM (built 15 Nov 2026; cut-over NOT done)

Why: the free Workers plan's per-request CPU limit was exceeded daily by Next.js server rendering. Cloudflare becomes DNS + proxy/CDN only (no Workers in production).

Product branch `feat/web-on-vm` (from `integrate/all`). What it adds:

| Piece | Where | Image (GHCR, multi-arch arm64+amd64) |
|---|---|---|
| Next.js standalone server (`output: "standalone"`, `node apps/web/server.js`, :3000) | `apps/web/Dockerfile` | `ghcr.io/gokul-227/remote-ai-platform-web:<sha>` / `:prod` |
| Staff console static export on a tiny Caddy (:8080) | `apps/admin/Dockerfile`, `apps/admin/docker/Caddyfile` | `ghcr.io/gokul-227/remote-ai-platform-admin:<sha>` / `:prod` |
| API (unchanged Dockerfile, now also arm64) | `apps/api/Dockerfile` | `ghcr.io/gokul-227/remote-ai-platform-api:<sha>` / `:prod` |
| Edge Caddy + `caddy-dns/cloudflare` + `caddy-ratelimit` | `infra/docker/caddy/Dockerfile` | `ghcr.io/gokul-227/remote-ai-platform-caddy:latest` |
| Hostnames, TLS, compression, rate limit | `infra/docker/Caddyfile` | |
| Services `web` (127.0.0.1:3000), `admin` (127.0.0.1:3002), custom `caddy` | `infra/docker/docker-compose.prod.yml` | |
| Pull, migrate, up, health gate, rollback | `infra/docker/deploy.sh` | |
| CI/CD | `.github/workflows/deploy-vm.yml` | |

`deploy-frontend.yml` no longer deploys Workers (renamed "Deploy backend (Render)"; it only triggers the Render API deploy until the API moves). `release.yml` now follows "Deploy production (VM)". The **dev** site still deploys to the `remote-ai-platform-dev` / `remote-ai-platform-admin-dev` Workers via `deploy-frontend-dev.yml` (OpenNext and wrangler stay in the repo for that only).

GitHub settings (set 15 Nov): secret `VM_SSH_KEY` (= `~/.ssh/oci_rap`), variables `VM_HOST=79.76.118.213`, `VM_SSH_KNOWN_HOSTS` (ssh-keyscan of the VM). Optional: `VM_SSH_USER` (default `opc`), `VM_EDGE_LIVE=true` after cut-over (enables the through-Cloudflare smoke test).

### How a deploy works
1. CI passes on a push to `prod` → `deploy-vm.yml` builds web/admin/api/caddy per architecture on native runners (`ubuntu-24.04` + `ubuntu-24.04-arm`), merges multi-arch manifests, tags `<sha>` and `prod` (caddy also `latest`). A manual dispatch builds any branch; it deploys only from `prod` with `deploy` ticked and only the tip of prod.
2. Deploy job (environment `production`): scp `docker-compose.prod.yml`, `Caddyfile`, `deploy.sh`, postgres conf to `/opt/rap/repo/infra/docker/`; `docker login ghcr.io` with the job token (logged out afterwards); `deploy.sh <sha>`.
3. `deploy.sh`: rewrites `RAP_IMAGE`/`WEB_IMAGE`/`ADMIN_IMAGE` in `/opt/rap/.env` to `<sha>`, `compose pull`, `compose run --rm --no-deps api alembic upgrade head`, `compose up -d --no-deps api worker scheduler web admin` (postgres/redis never recreated; caddy only when `EDGE_ENABLED=true`). Health gate (up to 150 s): `127.0.0.1:3000/api/version` reports `<sha>`, `/` and `/robots.txt` 200, admin on `127.0.0.1:3002` serves the console, API `/health/ready` 200.
4. Gate fails → tags reset to the previous SHA (`/opt/rap/state/previous`), `up -d`, gate re-run; the job fails. Migrations are forward-only (not reverted) — keep them backward compatible. Manual rollback on the VM: `bash /opt/rap/repo/infra/docker/deploy.sh --rollback`.

Note: the VM runs `postgres:16` (R3b) while the compose file on `integrate/all` says `postgres:17`; `deploy.sh` never touches postgres, but a bare `docker compose up -d` would try to recreate it — do not run one until that is reconciled. Uptime Kuma already holds 127.0.0.1:3001, hence admin on 3002.

### Cut-over (coordinator)
Prerequisite: a green `deploy-vm.yml` run on `prod` (web/admin/api healthy on loopback). Then put the Cloudflare token into `/opt/rap/.env` (opc, 600) and start the edge:
```bash
# from the Mac: token to the VM over stdin, never printed
infisical secrets get CLOUDFLARE_API_TOKEN --projectId 237deba5-a70e-4571-8529-26597d6e4631 --env prod --path / --plain --silent \
  | ssh -i ~/.ssh/oci_rap opc@79.76.118.213 't=$(cat); grep -v "^CLOUDFLARE_API_TOKEN=" /opt/rap/.env > /opt/rap/.env.new; echo "CLOUDFLARE_API_TOKEN=$t" >> /opt/rap/.env.new; mv /opt/rap/.env.new /opt/rap/.env; chmod 600 /opt/rap/.env'
# on the VM (API_PUBLIC_HOST stays api-oci.remoteaiplatform.com until the API moves)
echo EDGE_ENABLED=true >> /opt/rap/.env
cd /opt/rap/repo/infra/docker && C="docker compose --env-file /opt/rap/.env -f docker-compose.prod.yml"
$C pull caddy && $C up -d --no-deps caddy && $C logs -f caddy   # wait for "certificate obtained successfully" per host
```
The token needs **Zone → DNS → Edit** on `remoteaiplatform.com` (DNS-01 TXT records); check with `curl -s -H "Authorization: Bearer $T" https://api.cloudflare.com/client/v4/user/tokens/verify` and its permissions in the dashboard. If it lacks DNS:Edit, create a dedicated token (store in Infisical, put it in `.env` as `CLOUDFLARE_API_TOKEN`) or use a **Cloudflare Origin CA** certificate instead: Dashboard → SSL/TLS → Origin Server → Create certificate (hosts `remoteaiplatform.com`, `*.remoteaiplatform.com`, 15 years) → save as `/opt/rap/certs/origin.pem` / `origin.key` (600), mount `/opt/rap/certs:/etc/caddy/certs:ro` on caddy, drop `acme_dns` from the Caddyfile and add `tls /etc/caddy/certs/origin.pem /etc/caddy/certs/origin.key` to the `(common)` snippet and the www block. Origin CA certs are trusted only by Cloudflare — fine because every hostname stays proxied.

Verify the origin before touching DNS (bypasses Cloudflare):
```bash
for h in remoteaiplatform.com www.remoteaiplatform.com admin.remoteaiplatform.com; do
  curl -sS -o /dev/null -w "$h %{http_code} %{redirect_url}\n" --resolve $h:443:79.76.118.213 https://$h/; done
curl -s --resolve remoteaiplatform.com:443:79.76.118.213 https://remoteaiplatform.com/api/version
curl -s --resolve remoteaiplatform.com:443:79.76.118.213 https://remoteaiplatform.com/robots.txt | grep -x "Allow: /"
curl -sI --resolve remoteaiplatform.com:443:79.76.118.213 -H 'Accept-Encoding: zstd' https://remoteaiplatform.com/ | grep -iE 'content-encoding|content-security'
```

DNS (Cloudflare, zone remoteaiplatform.com). Apex, www and admin are currently **Workers Custom Domains**, whose DNS records are Worker-managed and cannot be edited directly:
1. SSL/TLS → Overview → encryption mode **Full (strict)**. Edge Certificates → Always Use HTTPS on, minimum TLS 1.2.
2. Workers & Pages → `remote-ai-platform` → Settings → Domains & Routes → remove `remoteaiplatform.com` and `www.remoteaiplatform.com` (and any routes); `remote-ai-platform-admin` → remove `admin.remoteaiplatform.com`. Do this immediately before step 3 (the hostnames are down in between).
3. DNS → Records → add, all **Proxied**, TTL Auto:
   - `A  @      79.76.118.213`
   - `A  www    79.76.118.213`
   - `A  admin  79.76.118.213`
   - `api-oci` already points at the VM (R4); leave it.
   - `api`: **unchanged** (Render) until the API cut-over; then replace the Render CNAME with `A api 79.76.118.213` (Proxied), set `API_PUBLIC_HOST=api.remoteaiplatform.com` in `/opt/rap/.env`, `$C up -d --no-deps caddy`, and remove the Render job in `deploy-frontend.yml`.
   Equivalent API calls (`$T` token with DNS Edit + Workers Scripts Edit; `$Z` zone id, `$A` account id — Infisical prod `/`):
   ```bash
   curl -s -H "Authorization: Bearer $T" "https://api.cloudflare.com/client/v4/accounts/$A/workers/domains" | jq '.result[]|{id,hostname,service}'
   curl -s -X DELETE -H "Authorization: Bearer $T" "https://api.cloudflare.com/client/v4/accounts/$A/workers/domains/<id>"   # per production hostname
   for n in remoteaiplatform.com www admin; do curl -s -X POST -H "Authorization: Bearer $T" -H 'Content-Type: application/json' \
     "https://api.cloudflare.com/client/v4/zones/$Z/dns_records" --data "{\"type\":\"A\",\"name\":\"$n\",\"content\":\"79.76.118.213\",\"proxied\":true,\"ttl\":1}" | jq .success; done
   curl -s -X PATCH -H "Authorization: Bearer $T" -H 'Content-Type: application/json' "https://api.cloudflare.com/client/v4/zones/$Z/settings/ssl" --data '{"value":"strict"}'
   ```
4. Verify through Cloudflare: `https://remoteaiplatform.com/api/version` (git_sha = deployed SHA), `/remote-jobs`, a `/jobs/<id>`, `/sitemap.xml`, `/robots.txt`; `https://www.remoteaiplatform.com/x` → 301 to the apex; `https://admin.remoteaiplatform.com/` (Operations console, `X-Robots-Tag: noindex`). Then set GitHub variable `VM_EDGE_LIVE=true`.
5. Optional hardening: restrict 80/443 in the OCI security list / firewalld to Cloudflare ranges (https://www.cloudflare.com/ips/).

Rollback of the cut-over (while the Workers still exist): delete the three A records and re-add the Workers custom domains (Workers → Domains & Routes → Add custom domain).

### Deleting the production Workers (after ≥ 7 days on the VM without rollback)
```bash
export CLOUDFLARE_API_TOKEN=… CLOUDFLARE_ACCOUNT_ID=…   # Infisical prod /
npx wrangler delete --name remote-ai-platform --force
npx wrangler delete --name remote-ai-platform-admin --force
```
Keep `remote-ai-platform-dev` and `remote-ai-platform-admin-dev` (dev still on Workers). Delete GitHub variable `PROD_FRONTEND_WORKERSDEV_URL`. When dev moves too: remove `@opennextjs/cloudflare`, `wrangler`, `open-next.config.ts`, both `wrangler.jsonc`, the `cf:*` scripts, CI's "Cloudflare Workers runtime smoke" step and `deploy-frontend-dev.yml`; update `tools/architecture/catalog.yaml` (web is still described as Workers, evidenced by `apps/web/wrangler.jsonc`) and run `make docs-sync`.

### Verified on the VM (15 Nov 2026; images built natively on arm64, run on loopback only, checked over an SSH tunnel)
Web image 352 MB. `/` 200 (0.15 s), `/remote-jobs` 200 (93 KB server-rendered, 0.97 s), `/jobs/<id>` 200 with the job title in `<title>`, `/sitemap.xml` 200 (index of static + jobs sitemaps), `/robots.txt` `Allow: /` + sitemap, `/privacy` 200, `/api/version` environment production. CSP/HSTS/X-Frame-Options from `next.config.ts` present; `/_next/static/*` `public, max-age=31536000, immutable`. Admin image: console HTML, deep links fall back to the SPA shell, `robots.txt` Disallow, `X-Robots-Tag: noindex`, zstd. Edge Caddyfile passes `caddy validate` in the custom image. Test containers removed; nothing new listens publicly.
