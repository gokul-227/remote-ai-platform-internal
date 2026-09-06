# 0004 — Backend OOM-crashes on boot, blocking today's deploy to dev/prod (P0)

Date: 2026-09-06

## Status: under active investigation

## What happened
After merging all 8 PRs from today's batch (#31, #33-#40), Render's auto-deploy for
both `dev` and `prod` started failing:
- Frontend deploy: fixed separately (npm 12's install-scripts blocking, see PR #41).
- Backend deploy: **fails with "Out of memory (used over 512Mi)"** ~20-30 seconds after
  a fresh process boots and successfully connects to the database. Confirmed reproducible
  on both branches, including a case with zero other instances running concurrently — so
  this is not a deploy-swap-overlap artifact, it's a real single-instance memory problem.

## Also observed (possibly related, possibly separate)
`dev`'s OLD (pre-batch, yesterday's) build started OOM-crash-looping today around 13:08
UTC, ~18 hours after its last successful deploy with zero new code — suggesting either a
genuine runtime memory leak that grows with uptime, or external noisy-neighbor pressure
on Render's shared free tier. Under investigation alongside the startup-spike question.

## Impact
- `prod` is NOT currently down — Render correctly refused to swap to the broken new
  build and kept serving yesterday's old build. But prod cannot receive ANY of today's
  fixes (including the real fix for completely-broken production WebSockets, PR #37)
  until this is resolved.
- Explicitly ruled out as a fix: suspending prod's working instance and redeploying. We
  confirmed a *single* fresh instance also OOMs with nothing else running, so this would
  just cause a full outage with no working replacement. Do not attempt this.

## Also fixed today, unrelated bug caught along the way
`validate_production_settings()` (added in PR #39) correctly fails closed on
`DEBUG=true` in any Render-hosted environment (Render always sets `RENDER=true`
regardless of `APP_ENV`) — but the dev/staging Render service had `DEBUG=true` set
intentionally for verbose local debugging. Since dev is also public-facing over the
internet, verbose tracebacks there are a real info-disclosure risk too, so we fixed this
by setting `DEBUG=false` on dev's Render env vars (via Render API) rather than loosening
the check. Sentry already captures full error detail for debugging without exposing
tracebacks to clients.

## Next steps
An agent is investigating the actual OOM root cause (dependency-bump memory increase vs.
a genuine leak vs. lifespan/startup-time heavy work) and will open a P0 hotfix PR against
`dev`. Do not merge/deploy anything further to prod until this lands and is verified.
