# AWS migration plan (Phase 17) — only after revenue/budget approval

Not started; nothing here is a commitment. Prerequisites: measured monthly usage, an approved AWS budget, and a reason
(reliability, residency, a customer requirement) the current providers can't meet.

1. **Portability first (no AWS yet):** storage already uses an S3 interface; database via SQLAlchemy/asyncpg;
   identity stays Supabase Auth (don't couple an identity migration to the infrastructure move).
2. **Account setup:** separate prod/non-prod accounts, IAM roles with GitHub OIDC (no static keys), budgets + alerts.
3. **Region:** eu-central-1 (Frankfurt) next to EU users and the EU data promise.
4. **Mapping:** Cloudflare DNS/CDN kept; API → ECS Fargate (or App Runner) behind ALB; PostgreSQL → RDS (single-AZ at
   first); objects → S3 private buckets; Redis → optional (ElastiCache only if needed); scheduled sync →
   EventBridge Scheduler; secrets → Secrets Manager.
5. **Rehearsal:** restore the latest backup into RDS/S3 with `ops/backup/restore.sh` (checksums, row counts), run E2E
   against it, measure latency.
6. **Cutover:** write freeze → final backup/restore → switch `DATABASE_URL`/storage env → smoke + journeys → observe;
   exactly one writer at all times; rollback = switch env back before new writes accumulate.
7. **Decommission** old resources only after the retention/rollback window agreed with the owner.
