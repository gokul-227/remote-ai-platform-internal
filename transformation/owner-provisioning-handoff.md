# Provider provisioning and release handoff

This is an owner action register, not evidence that these services are provisioned.
The owner authorized free tiers with a card where required, direct dev/prod promotion,
and local implementation with provider provisioning deferred to the owner.

| Service | Owner action | Verification before cut-over |
|---|---|---|
| OCI | Create tenancy and free compute in home region; supply SSH access securely | VM capacity/quota, private Redis, persistent volumes, restart and restore drill |
| Neon | Create isolated development database; store URL in Infisical | Migration from empty DB, representative restored data, SSL and pool behavior |
| R2 | Enable billing and private bucket; scoped S3 credentials in Infisical | Upload, short-lived download, bucket CORS, unauthorized access rejection |
| WorkOS | Create staging AuthKit project and configure redirect URLs | Real login/recovery/revocation; verified identity mapping; dual-issuer rollback |
| Upstash | Create Redis database; store REDIS_URL securely | TLS connectivity, command quota, outage fallback; worker broker load feasibility |
| Grafana Cloud | Create free stack and scoped ingestion credentials | Metrics/log delivery, retention limits, alert delivery |
| FCM | Create project and web application if push is enabled | Browser permission, registration, token rotation and revocation |

Never paste credentials into chat or commit them. The existing provider configuration remains authoritative until its replacement passes verification. Recheck current quotas and billing controls in each provider console before enabling anything.

## Release order

1. Finish implementation and integration checks on dev.
2. Capture tested commit SHA and existing deployed SHA.
3. Export a database backup and verify a restore into an isolated database; retain object inventory and recovery procedure.
4. Apply additive migrations against staging first; compare row counts and key relations.
5. Verify provider connections and real authentication with synthetic accounts.
6. Deploy tested images and frontend SHA; inspect version and readiness endpoints.
7. Run member/staff authorization and file upload/review smoke journeys.
8. Enable worker scheduling only after heartbeat and single-scheduler lock checks; disable GitHub import scheduling at cut-over to avoid duplicate execution.
9. Observe errors, queue age, storage, quotas and costs. Roll back application SHA if release gates fail; do not automatically downgrade schema containing new production data.

## What blocks complete infrastructure migration

Provider accounts/credentials, an actual verified backup restoration, live login/OAuth/recovery,
and deployed worker/database/object-store evidence are required. Source code, local tests and
runbooks alone do not establish completion of phases 4, 6, 7, 16 or 17.
