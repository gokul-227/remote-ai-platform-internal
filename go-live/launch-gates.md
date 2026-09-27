# Launch gates (Phase 15)

Status on 2026-09-27 after phases 00–14 (code on stacked local branches; not yet pushed or deployed).
Release rule: no open P0 for a launched surface.

| Gate | Evidence needed | Status |
|---|---|---|
| G01 Provenance | Web + API report the tested SHA; migrations at head | Deploy workflows assert SHA and smoke-test after deploy. Pending first deploy of this work |
| G02 Secrets/admin | Rotation evidence; branch protection | **Open**: keys were pasted into chat again on 2026-09-27 (incl. Stripe live) — rotate all; restore `enforce_admins`, add Secret scan + CodeQL as required checks |
| G03 Identity/authz | Real auth smoke; cross-user negatives | Code: API-01, DATA-01, IDOR audit fixed + tested. **Open**: real OTP/OAuth smoke in staging (dev API down) |
| G04 Privacy/rights | Export, deletion, notice aligned | Code done (DATA-01/02, retention guard, notice). Needs `SUPABASE_SERVICE_ROLE_KEY` on Render |
| G05 Legal/support | Operator identity; mailbox receipt | **Open**: LEG-01 facts; OPS-02 mailbox test; legal review |
| G06 Core journeys | All exposed journeys | 36 E2E journeys pass (hermetic). Real-provider staging pending |
| G07 Recovery | Encrypted backup + timed restore of production | Tooling rehearsed locally. **Open**: first production backup + restore drill |
| G08 CI/staging/release | Required checks, working staging, rollback | CI hardened; **open**: dev environment (paused Supabase), branch protection |
| G09 A11y/performance | Automated + manual a11y; budgets | Automated WCAG 2.2 AA on all signed-in screens; perf measured (cache hit 2 ms, miss ~1 s at transatlantic RTT). Manual screen-reader pass + Render region decision open |
| G10 Operations/cost | Alerts, quotas, owner | Canary alerts; runbooks. Sentry alert rules + quota ledger open |
| G11 Payment truthfulness | Money movement off | Verified off (503); PAY-02 fixed |
| G12 Public exposure/SEO | Indexing, canonical, honest claims | Job pages, 404s, sitemap, staging noindex, claims test |

**Verdict:** engineering-ready once pushed and CI-green; **not yet limited-beta ready** (G02, G05, G07 open); not
public-market, paid-launch or AWS-migration ready. Staged release proposal: invite-only beta (10–30 people) after G02/
G05/G07 close; widen after two weeks without Sev 1–2 incidents.
