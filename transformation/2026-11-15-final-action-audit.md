# Claude final-action audit — 15 November 2026

**Result:** legal/SEO/scale source work is integrated at `db2a7d2`; remaining work includes verification, a documented responsive defect, capacity acceptance, and dependency-safe cleanup planning. Deployment/settings refresh alone does not close these actions.

Scope: read-only local repository/session audit by coordinator and two independent read-only agents. Only this report was written; no tests, remote queries, credential retrieval, deploy, settings change, or commit. Runtime statements below are checkpoint evidence, not freshly verified live facts.

## Authority and completed work

- Claude session `4599f00e-d18b-4eeb-87b3-e10c0847aecd.jsonl`: literal last assistant text, line 12890, is a rate-limit notice. Final substantive action list is line 12874; earlier commitments at 12833/12854 cover secret cleanup, production E2E, Jooble and weekly SEO reporting. Raw credential-bearing turns were not printed.
- Product `rap-runtime-observability`, `fix/vm-runtime-observability`, HEAD `db2a7d2`, clean at inspection. Legal `512f18d`, in-app policy links/runtime release gate `f09a56a`, SEO `95fafe3`, scale `aa2d40a`, deployment integration `0946c2a`, migration merge `db2a7d2` are retained in its history. Do not assign agents to reimplement or remerge these.
- [Completion checkpoint](2026-11-15-claude-worktree-completion.md:27) records combined backend 1038 passed/16 skipped, web 236 passed, filter review 58 passed, and fresh migration076 round-trip. Those are recorded results, not tests rerun by this audit. Claude's old six-policy in-app limitation is superseded by `f09a56a`.

Product paths below are relative to `../rap-runtime-observability/` from the internal repo root.

## Exact remaining actions beyond deploy/settings refresh and owner gates

| Action | Closure evidence / source |
| --- | --- |
| **Finish final-image browser and reconnect validation.** Recheck messaging hydration, meetings and complete member/staff journeys; record passing results or remaining failures at the integrated candidate. | `transformation/continuation.md:35–48` retains unobserved reruns after a signed-out messaging failure. `docs/development/e2e.md:3–4` describes isolated mock-provider tests. Source/unit totals do not establish full browser acceptance. Production smoke is a separate, later gate; never point the synthetic-user/staff mutation suite at production (`tests/e2e/tests/operations-console.spec.ts:4–8`). |
| **Resolve the documented staff-console tablet overflow.** Reproduce before changing source; fix and verify sign-in, Staff & roles and Feature flags at 820px. | `docs/product/feature-audit.md:51–54` records 84px overflow. `tests/e2e/tests/responsive.spec.ts:153–166` skips staff coverage unless the isolated admin URL is supplied. This audit did not reproduce the defect. |
| **Produce capacity acceptance rather than a capacity claim.** Use an isolated representative dataset; measure request p95/errors, throughput, memory, database CPU, catalogue refresh runtime and cold/fallback behavior. | `docs/operations/capacity.md:5–6,26–33,64–79,102–103` explicitly leaves full-size load and target-host throughput unverified. 10M jobs/1M members is a target only. |
| **Make capacity evidence reliable.** Include HTTP cursor traversal and explicit semantic-check thresholds; ensure benchmark failures are surfaced in acceptance and small-seed runs can initialize. | `tests/load/capacity.js:36–40,61–77` has HTTP thresholds but no `checks` threshold and browses by offset. `tests/load/scale/bench.py:54–55,87–92` assumes a row at offset100000 and catches query failures without failing the run. Warm medians are not end-to-end p95. These are tooling follow-ups, not evidence the application cursor is broken. |
| **Close remaining acceptance evidence explicitly.** Record populated upgrade/write-lock and old/new compatibility checks for new indexes; scoped accessibility and isolated outage/retry/reconnect results. | `docs/operations/capacity.md:15–24` warns about blocking index creation. The older `2026-11-14-phase-readiness-checkpoint.md:86–94` matrix is not closed by the newer unit/fresh-migration results. Classify these as unverified gates, not newly observed failures. |
| **Complete Claude's promised dependency review before any secret retirement.** Prepare a setting-name-only keep/replace/retire list and reconcile stale checkpoint claims. | `ops/vm/render-env.sh:16–23` still consumes `/render-api`; `docs/operations/secrets-and-config.md:29–38` still uses transitional Supabase, MINIO-named storage and Sentry-named GlitchTip settings. Names alone cannot establish obsolescence. Actual deletion/rotation is separate operational work and may depend on auth cutover/fallback expiry. |

## Already-known release-dependent checks — still need evidence

After the separately authorized deploy/settings refresh: verify successful final-SHA CI, deployed web/admin/API versions and image provenance, schema076, worker/scheduler heartbeats, isolated job-cache and catalogue freshness; check legal pages/security.txt, sitemap shard latency, indexable landing pages and actual Cloudflare cache behavior. Then capture successful Jooble source ingestion (not merely key presence) and weekly SEO report delivery. Jooble is scheduled once daily; avoid wasting its request budget. Evidence: completion checkpoint `:24,27–35`; `apps/api/app/domains/tasks/service.py:68–69`; `docs/operations/seo.md:119–150`.

## Owner/external gates — excluded from independent code completion

Adzuna permission; France Travail/Careerjet/USAJOBS credentials; optional Reed/Findwork integrations (conditional, not promised core work); Search Console API/property access and Cloudflare HTML-cache rule; legal jurisdiction/response commitments and annual security.txt renewal; WorkOS activation/account continuity, Google/LinkedIn approvals, monitoring endpoint/webhook and branch-protection administration, exposed-key rotation, and post-fallback service retirement. Refreshing settings alone does not grant Search Console access or enable its report flag. Sources: session lines12833–12874; `docs/operations/seo.md:127–150`; `docs/operations/cutover-history.md:15–25`; `transformation/continuation.md:16–21`.

Do not resurrect superseded initial OCI/database/file/backup provisioning blockers: newer cutover and restore records document VM migration and a successful restore (`docs/operations/cutover-history.md:10–13`; `docs/operations/backups-restore.md:48–51`). Current Compose also has worker egress and a no-eviction broker separated from the query cache. These local records do not independently certify current runtime health or close the remaining acceptance gates.

## Parallel completion handoff

1. **Browser/UI agent:** isolated final-image journeys, reconnect evidence, tablet defect; no production mutation suite.
2. **Scale agent:** capacity-tool corrections, representative load and populated migration/compatibility evidence; no production load.
3. **Coordinator/operations agent:** dependency-only cleanup inventory, evidence reconciliation, and later release/report/import checks when operationally authorized; preserve owner gates.

The two audit agents finished their read-only reviews; the execution lanes above remain future work. This audit does not authorize or claim their implementation.

Final status check: only this report is untracked in the internal repo. Product `infra/docker/deploy.sh` became modified during the audit after the initial clean check; concurrent work was left untouched and is outside the committed `db2a7d2` assessment.
