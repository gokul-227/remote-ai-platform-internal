# Phases 4 / 6 / 7 / 16 / 17 — readiness checkpoint

Assessment date: 2026-11-14. Live probes: 07:16:52–07:16:54 UTC. This is a bounded documentation assessment, not release approval.

**Release classification: Not ready, with explicit blockers. None of these five phases is complete.** Existing production continues running; this classification concerns acceptance of the transformation and its controlled migration.

## Authority, scope and baseline

Acceptance authority: workspace root `../../remote-ai-platform-implementation.md`, Part 12 phase gates, Parts 11/13/15 security/testing/cost requirements, Part 16 checkpoint format and Part 17 final Definition of Done. Read alongside `00-phase0-baseline-audit.md`, `03-architecture-and-migration-design.md`, `04-infrastructure-runbooks.md` and the internal handbook. Earlier statements are historical evidence, not completion proof. This checkpoint supersedes conflicting operational instructions in the older transformation documents.

Product checkout: `/Users/gokulr/Developer/GitHub/remote-ai-platform`, branch `feat/launch-discovery-hardening`, inspected HEAD `0d9f99f5f90dc032ec7aaf77092f70624c68ff7d` (local dev/origin-dev refs also pointed here). Internal checkout: `remote-ai-platform-internal`, `main`, starting HEAD `316041bf5490fbd388da4b45c2beb8342e11f245`, initially clean. These are timestamped observations, not assertions that remote branches remain unchanged.

Other agents have uncommitted changes in `apps/api/app/domains/auth/supabase_auth.py`, `apps/api/tests/test_multi_issuer_auth.py`, and, during inspection, notification preferences source/tests. They were not edited. CI for HEAD does not validate this moving worktree. Auth remediation belongs to the other agent; this assessment does not certify it.

Only transformation documentation was changed. No deploy, provider mutation, workflow dispatch/rerun, secret retrieval, database migration, restore, account creation, or production-user test was performed.

## Verified evidence

Product paths below are relative to the product repository.

| Evidence | Observation and limit |
|---|---|
| [CI 43569284261](https://github.com/gokul-227/remote-ai-platform/actions/runs/43569284261) | Success for `0d9f99f…`: backend lint/typecheck/tests, secret scan, frontend/admin build/tests, local Workers smoke, isolated Playwright E2E, migration/concurrency jobs all succeeded. Read job/step results; did not rerun tests or infer exact test counts. |
| `.github/workflows/ci.yml` | E2E uses mock Supabase. Migration job upgrades a fresh PostgreSQL 16 DB with two concurrent migrators and runs PostgreSQL concurrency tests. This is not a populated production-copy upgrade, downgrade drill, Neon integration, real WorkOS flow, or disaster recovery exercise. |
| [Dev deploy 43569969280](https://github.com/gokul-227/remote-ai-platform/actions/runs/43569969280) | Overall failure. Backend and web deploy jobs succeeded; admin failed at “Verify the console serves and is not indexable” after its Deploy step succeeded. Workflow-run metadata lists prod SHA `276e669…`; do not use that metadata alone to identify the deployed dev artifact. No root cause inferred from the step name. |
| [Production API version](https://api.remoteaiplatform.com/health/version) | HTTP 200, production SHA `276e66921c59750d4c956a263c88311c2c7e3d87`. |
| [Production readiness](https://api.remoteaiplatform.com/health/ready) | HTTP 200 but body `DEGRADED`: PostgreSQL `HEALTHY`, Redis `DOWN`, error `unavailable`. Reachable API is not healthy dependency coverage. |
| [Dev API version](https://remote-ai-platform-api-dev-eu.onrender.com/health/version) and [readiness](https://remote-ai-platform-api-dev-eu.onrender.com/health/ready) | SHA `0d9f99f…`, environment `staging`; DB/Redis healthy. Worker and scheduler `UNKNOWN`, each zero heartbeats. Does not establish independently operating imports. |
| Web `/health/version` on production and dev domains | Both returned HTTP 403 to this inspection client. Frontend deployed SHA not independently established by these probes; 403 is not proof of application outage. |
| `.github/workflows/publish-api-image.yml` | Source exists on inspected branch; querying workflow runs by this filename returned GitHub 404 “not found on the default branch.” Published image/digest and multi-architecture manifest remain unverified, not proven absent. |

## Root plan acceptance matrix

| Phase | Root deliverable and gate | Current status | Evidence required to close |
|---|---|---|---|
| 4 | Verified new development infrastructure; new backend works without interrupting production | **Partial implementation; provider/runtime gate open** | OCI dev capacity/account evidence; validated Compose; exact CI-tested image/digest; isolated dev secrets/data; TLS/network/egress checks; durable worker/scheduler verification; monitoring alerts; recoverable backup; continued production baseline. Current Render dev health does not prove OCI foundation. |
| 6 | Working development DB/object storage; existing production records and file access intact | **Partial reusable storage/backup code; modernization unverified** | Neon project and migrations on actual isolated Neon; R2 private buckets and credentials; signed upload/finalize and secure download tests; existing file compatibility; populated migration rehearsal; measured full restore including objects and deletion history. No production DB move initially. |
| 7 | Tested authentication migration proposal and safe implementation; no existing user loses access | **Backend scaffolding exists; evaluation and migration not accepted** | Other agent's auth fixes reviewed/tested at final SHA; real staging WorkOS sign-in/OAuth/refresh/recovery/revocation; identity conflict and role tests; existing-user preservation and WorkOS-only-user fallback rehearsal; accepted compatibility report. |
| 16 | Evidence-based release assessment; no unresolved critical launch blocker | **Assessment produced; readiness gate failed** | Resolve or explicitly scope every blocker below; final-SHA regression/security/accessibility/performance results, populated migration and restore drill, outage/cost evidence, successful web/admin/API deployment verification. Green CI alone insufficient. |
| 17 | Owner-authorized controlled migration and verified production deployment | **Not authorized by this task; not executed or accepted** | All prerequisite gates, explicit owner release authorization, verified backup/recovery, tested artifacts, quotas, auth continuity, compatible migrations, safe multi-persona smoke tests, observation window and rollback readiness. |

Root Part 17 professional, organisation and internal administrator journeys are **not accepted by this bounded review**. Existing CI is useful regression evidence, but no complete capability-by-capability mapping of workflow, persistence, authorization, error states, tests and docs was established here. Phases 8–15 cannot be silently treated as complete dependencies. No pilot or broader-production readiness claim is made. Payments remain outside this transformation scope; preserve existing functionality and do not enable paid fallbacks.

## Verified source gaps and release blockers

1. **Infrastructure networking and capacity:** `infra/docker/docker-compose.prod.yml` places worker and scheduler only on the `internal: true` network; API additionally joins `edge`. With external PostgreSQL and import providers, the worker/scheduler have no declared external network path. Validate and repair controlled egress before a VM deployment. Redis uses `allkeys-lru` with a bounded memory budget while also acting as broker: prove delivery/reconciliation under eviction or separate broker and cache policies. The full declared service memory limits exceed 1 GB before host overhead; the old E2 micro fallback is not a validated capacity plan.
2. **Runtime/provenance:** Compose and `.env.prod.example` default to `:latest`, while publish workflow emits SHA/branch tags. Require an explicit tested SHA plus resolved digest and architecture; do not rely on the default. API port 8000 is not host-published: the old host `localhost:8000` probe is invalid for this profile. Probe inside API or through Caddy. `curl -f` alone accepts degraded HTTP 200; assert DB, Redis and required heartbeats in JSON. Scheduler has no Compose healthcheck. No Alloy/backup service exists in this profile.
3. **Database/storage:** `app/core/storage.py` uses configurable S3-compatible `MINIO_*` settings, server-side PUT and presigned GET. Resume/deliverable routes provide existing private download paths. No general Files domain, signed PUT/finalize flow, or `R2_*` settings consumer was found in the inspected application. Merely adding the old runbook's R2 variable names does not wire storage. Implement or document the exact `MINIO_*` mapping and per-purpose bucket routing before enabling it.
4. **Migration hazards:** `start-production.sh` automatically runs `alembic upgrade head`. The inspected migration sequence reaches `060_search_trgm`; the old “056 applies next” statement is incomplete. Revision 060 creates GIN indexes without CONCURRENTLY; source comments about duration are not measured lock evidence. Require a populated rehearsal with lock timeout, write-impact measurements and old/new application compatibility. Actual production Alembic revision was not queried; an app SHA is not a schema inventory.
5. **Auth design drift:** migration `057_user_identities` creates a separate table keyed by `(provider, subject)`; it does not add the proposed `auth_subject_workos` column. Multi-issuer backend verification exists; browser sign-in remains Supabase (`apps/web/src/lib/supabase.ts`). Generic extra-issuer support does not prove AuthKit integration. Provider-specific issuer/JWKS/audience/verified-email semantics must be tested with actual staging tokens. Do not copy example config as a verified provider contract. Current auth work is uncommitted and not accepted here.
6. **Recovery gaps:** `ops/backup/backup.sh`, `restore.sh`, `objects.py` provide encrypted dump/object tooling and safety checks. No backup scheduling/restic implementation was found in inspected workflows or worker source; the old claim of an existing artifact backup workflow is not supported by this tree. Backup excludes Auth users/sessions, grants/roles, bucket policies and provider config. Restore reference verification currently queries resume URLs; it can warn about missing objects and still print success. Extend evidence to deliverables and every stored reference; warnings cannot count as a passing release drill.
7. **Operational readiness:** production Redis failure remains observed; dev worker/scheduler heartbeats are absent; dev admin deployment verification failed. No current account-level quota/billing proof, OCI/Neon/R2 runtime proof, full recovery measurements, provider-outage rehearsal, or production-like performance acceptance was established. These are open release gates, not assertions that providers/accounts do not exist.

## Provider account gates (owner supplies evidence; no provisioning in this task)

Earlier dated free-tier tables are planning inputs, not currently verified entitlements. Do not assume card-on-file, a public free tier, or a “free” product prevents overage. Record owner, environment, region, plan, enabled billing products, hard caps/alerts, retention and dated entitlement evidence without secrets. Exact current prices/allowances were not reverified in this assessment; therefore none is newly certified here.

| Provider / dependency | Required gate before use | Current evidence |
|---|---|---|
| Oracle Cloud | Owner confirms tenancy, region/capacity, VM architecture, compute/disk/network entitlement and reclaim policy; isolated dev host; SSH restricted, TLS ingress, private broker, required outbound paths, recovery host plan | Source/runbook only; account/capacity not inspected. Do not guarantee that scheduled work prevents idle reclamation. |
| Cloudflare DNS / Workers | Account/zone ownership, current routes, certificate mode, WAF/Turnstile entitlements and smoke results; record existing DNS and TTL before future changes | CI deploy records available; direct web probe blocked with 403. No DNS/WAF changes performed. |
| Cloudflare R2 | Explicit owner billing/overage decision; dev-only private buckets first; scoped token, CORS, lifecycle, upload and operation budgets; actual configured settings mapping | S3-compatible source available; R2 account and end-to-end storage unverified. |
| Neon | Separate dev project/branch; connection and TLS/pooler compatibility with asyncpg and Alembic; extensions/RLS/grants; storage/compute/connection/restore limits | Fresh PostgreSQL CI passes; actual Neon integration unverified. Keep production Supabase unchanged. |
| WorkOS | Staging app and redirect/logout allowlist, OAuth connections, actual claims, account recovery, revocation and billing boundaries; owner accepts compatibility report before production | Backend scaffold only; real staging flow not verified. |
| Supabase / Render | Preserve existing identities, DB, buckets and fallback deployment; prove backup access, storage policies, auth recovery and supported rollback artifacts | Production API version and DB health verified; identity/storage internals not inspected. |
| Redis / candidate Upstash | Investigate current failure; confirm TLS, Celery command/connection compatibility, memory/eviction, quotas and sustained scheduler traffic before selecting replacement | Production down; dev healthy; no provider selected or modified. |
| Grafana / Sentry / UptimeRobot | Account/retention/ingestion quotas, real alert delivery, API/queue/import/backup dashboards, alert owner | Instrumentation/design is not proof of configured alerting. |
| Infisical / GHCR | Least-privilege environment access; no prod export for dev; image pull and manifest/digest proof; secret handling and rotation plan | No secret access attempted; image publication lookup inconclusive as described above. |

## Migration and rollback acceptance requirements

These are requirements for a later authorized execution, not commands to run now. Do not migrate authentication and production PostgreSQL together.

1. **Inventory before mutation:** capture affected row/object counts, actual schema revision, old/new artifact digests, identity mapping coverage, config version references, DNS records/TTL and current health. Use synthetic or sanitized dev data. Do not copy production secrets or use production accounts as fixtures.
2. **Recoverability:** verify an encrypted off-provider backup, checksum, retention and separately accessible recovery key. Restore to a distinct empty target using source/target identity checks and explicit target confirmation. Reconcile counts, metadata, object hashes and all file references; reapply post-backup deletions. Preserve identity tombstones. Measure RPO/RTO and obtain owner-accepted thresholds before release; none are accepted yet.
3. **Database:** rehearse upgrades from the actual prior revision with representative populated data, RLS/roles/extensions and concurrent traffic. Review every pending revision. Test the previous application against the expanded schema. Prefer application rollback retaining additive schema; use a forward fix or verified restore when required. A `downgrade()` that drops new tables/columns can destroy new writes. Never automatically downgrade production as the default rollback.
4. **Objects:** copy rather than move/delete, retain source bucket and existing URLs/keys, verify object bytes/metadata and private access across roles, then change routing separately. Track uploads created after cutover so rollback cannot strand them; define synchronization or a write pause/reconciliation window. Signed URL expiry and revoked access must be exercised.
5. **Auth:** retain Supabase subjects/accounts and linked identity rows. Test conflicts, duplicate emails, concurrent linking, disabled/deleted users, revocation, role escalation and cross-tenant isolation. Rehearse return to Supabase for both previously linked and WorkOS-only users; simply removing an issuer can lock out the latter. Never use `RESET_IDENTITIES=1` as routine rollback: current restore code clears subjects and deletes tombstones. Recovery of a new identity project needs its own reviewed preservation procedure.
6. **Infrastructure/background jobs:** deploy dev first with isolated secrets, database, bucket and broker. Preserve Render fallback. Before any future scheduler cutover, prove restart/retry/outbox/dead-letter recovery and one active scheduling authority. Record the GitHub import schedule state; avoid simultaneous duplicate producers. Drain/reconcile in-flight work when reverting. Network and image fixes require implementation by a separate authorized task.
7. **Release decision:** coordinator freezes a final candidate SHA, collects successful checks including other agents' changes, and obtains explicit owner authorization. Define rollback triggers for login failures, authorization regressions, missing files, DB errors/locks, queue growth and unacceptable latency; set numeric thresholds and observation duration before cutover. Revert traffic/artifact/config only to a tested compatible fallback. Verify web, admin, API, identities, file access and imports after either deployment or rollback.

## Phase 16 remaining verification matrix

| Area | Existing evidence | Still required |
|---|---|---|
| Regression / frontend / admin | Finalized CI for inspected committed SHA | Repeat at final integrated SHA; complete root-plan multi-persona journey mapping |
| Security / authorization | Secret scan, CodeQL success and test jobs | Auth agent's final review; real-provider claim and revocation tests, cross-tenant/files/admin coverage mapped to requirements |
| Accessibility | Playwright job succeeds | Identify accessibility coverage/results for complete scoped journeys; green aggregate does not prove every screen |
| Performance / concurrency | PostgreSQL concurrency job succeeds | Representative load, budgets and index lock measurements on target sizing |
| Migration / backup | Fresh DB upgrades and backup safety tests | Populated upgrade/compatibility, full restore and identity/object recovery with measured RPO/RTO |
| Provider outage / cost | Live Redis degradation observed | Controlled isolated outage/reconnect/retry drill; account entitlements, usage caps and alert delivery |
| Deployment | Dev API SHA matches CI | Admin verification resolution, independently established web/admin versions, OCI image/network/runtime checks and authorized production release |

## Continuation record (root Part 16)

PHASE: 4 / 6 / 7 / 16 / 17

STATUS: Partial / Partial / Partial / Not ready / Not authorized or executed in this task.

CURRENT BRANCH / COMMIT SHA: Product `feat/launch-discovery-hardening` at `0d9f99f5f90dc032ec7aaf77092f70624c68ff7d`; internal `main` starting `316041bf5490fbd388da4b45c2beb8342e11f245`.

OBJECTIVES COMPLETED: Source-backed gate assessment, live read-only evidence, root acceptance mapping, provider gates and recovery requirements.

FILES CHANGED: This checkpoint; precedence/correction notes in `03-architecture-and-migration-design.md` and `04-infrastructure-runbooks.md` only.

DATABASE CHANGES / INFRASTRUCTURE CHANGES: None.

REA RESEARCH COMPLETED: None in this task.

TESTS EXECUTED: None locally; read existing CI job/step results, source, Git status and public GET health/version responses.

TEST RESULTS: Inspected SHA CI passed; deployment verification partially failed; runtime evidence above. No acceptance inferred for uncommitted changes.

SECURITY REVIEW: Documentation-level account continuity, file access, backup exclusions, broker/network and secret-separation risks; not a completed security audit.

RUNTIME VERIFICATION: Production API old SHA + Redis down; dev API current inspected SHA + no background heartbeats; web probes 403.

COST AND QUOTA IMPACT: No resources created; account-level cost/quotas remain gates.

KNOWN LIMITATIONS: No account-console inspection, authenticated journey execution, live schema inventory, image manifest proof, migration/restore/load/outage drill. Active concurrent source work may supersede snapshot.

ROLLBACK: Documentation changes can be reverted independently. Future technical rollback must meet the requirements above; none performed.

NEXT EXECUTABLE TASKS:

1. Auth agent finishes fixes and provides final committed test evidence; coordinator updates this checkpoint without treating current dirty files as CI-tested.
2. Infrastructure owner/engineer resolves Redis and admin verification findings, validates Compose egress/capacity/broker semantics and SHA image publication, and supplies account entitlement evidence. These actions need their own authorized execution scope.
3. Implement missing Phase 6 upload/finalize/configuration paths; owner supplies isolated Neon/R2 development accounts, then run provider integration and recovery drills under separate authorization.
4. Build Phase 7 real-provider compatibility report and account-continuity rollback rehearsal.
5. Coordinator closes the Phase 16 evidence matrix and root journey acceptance; only then seek a concrete Phase 17 owner release decision. Do not deploy from this documentation task.
