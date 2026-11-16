# Release recovery — 16 November 2026

Candidate: `692b5042c33cd1df28fbd6208dcea99d77921241`. Production is still on rollback
`068d41b` pending exact-candidate CI and a subsequent health-gated deployment.

The earlier `885aa7d` rollout passed CI/deploy gates but live cold hub requests exposed
expensive fan-out and PostgreSQL sequential full-text scans. Rolled the application back;
additive schema076 and verified indexes remain. Broker policy corrected persistently to
noeviction; disposable query cache keeps allkeys-lru. No user/database/object data deleted.

## Fixes

- Count-only hub eligibility API and four concurrent lookups per directory, rather than
  dozens of complete salary/employer/skills aggregates.
- Server-side web requests use the API container directly.
- Keyword detail statistics aggregate projected rows from one capped window.
- Shared live-job predicate now matches partial index predicates. Read-only production
  EXPLAIN showed IS TRUE/IS FALSE taking ParallelSeqScan; bare boolean predicates take
  ix_job_posts_live_fts BitmapIndexScan. Local PostgreSQL regression enforces eligibility.
- Three PostgreSQL tests cover2,105/84,000synthetic rows and index selection; CI executes
  them alongside its ten existing concurrency tests. No10M-capacity claim.

## Isolated VM evidence

Restored latest encrypted/offsite backup into separate PostgreSQL17 and MinIO containers:
99tables,5objects/checksums/metadata/references verified. Migrated scratch schema to076.
Candidate API/web images used a private test network and separate Redis; the web also had
production-like outbound connectivity. No synthetic writes or load against live user data.
Restored job count85,009. Redis query caching disabled for the test API.

Observed one run (milliseconds; not p95/throughput certification):

| Request | Status | Time |
|---|---|---|
| Python eligibility count |200|204|
| Full unfiltered landing stats |200|304|
| Job list, two rows |200|48|
| First homepage render |200|1205|
| Remote-work hub |200|71|
| Work-from-home hub |200|77|
| Country hub |200|62|
| Subsequent homepage |200|81|

Three concurrent readiness samples remained healthy; maximum90ms. Scratch containers,
anonymous volumes and network removed by trap. VM evidence log:
`/tmp/rap-candidate-692-verify-egress.log`; helper `/tmp/rap-candidate-692-verify.sh`.
Earlier fully internal/no-egress harness delayed first-page rendering before API requests;
that network setup was not production-equivalent. Its failed result is retained separately,
not counted as acceptance. No specific telemetry/DNS cause is asserted.

Local code checks passed:237webunit,33focusedhub/landingunit,17SEOAPI,countsAPI,
9deployregressions,5focusedPlaywright,fullwebformat/TypeScript/lint,backendRuff/mypy.
Final692CI43870170740 still running at this checkpoint. Build-only43870170459 succeeded.
Do not promote before exact-candidate CI is green.

Owner gates persist: WorkOS activation/identity cutover, Search Console access/reporting,
provider terms/credentials, legal review and dependency-safe secret/provider retirement.
Representative85k evidence is not certification of the10M-job/1M-member target.
