# Decision: no dedicated Celery worker for now

**Date:** 2026-09-05
**Status:** Accepted
**Resolves:** Phase 0 report P0-2

## Context

Phase 0 audit found Render hosts exactly 2 services (prod + dev API), zero
Celery worker/beat processes. The `beat_schedule` (3 cron jobs) and any
`.delay()`-dispatched task have never run in production. Resume parsing was
the concrete casualty: `ResumeParserAgent` was only ever wired to a Celery
task nothing dispatches.

## Decision

Don't stand up a dedicated Celery worker service right now. Instead:

- **Per-request async work** (resume parsing): run inline, synchronously, in
  the same request. A single LLM call, already bounded by `LLMClient`'s own
  timeout/fallback handling — fits an inline pattern fine at current traffic.
  Implemented in PR #9.
- **Periodic scheduled work** (trending-skills refresh, stale-match
  recompute): follow the same GitHub-Actions-cron pattern already proven for
  job-source sync (`scheduled-job-sync.yml`), rather than a worker. Not yet
  implemented — flagged as a follow-up, not done in this pass.

## Why

Render's free tier funds exactly the 2 services already running. A third
always-on worker either costs real money or eats into the shared 750
instance-hour/month pool that keeps prod itself warm — not the pragmatic
call at this stage of the product, and avoids prematurely introducing
infrastructure complexity ahead of proven need (matches the "don't
prematurely introduce microservices" principle).

## Revisit when

- Request-scoped AI calls start meaningfully slowing down user-facing
  requests (resume upload blocking for several seconds becomes a real UX
  problem at higher volume)
- A scheduled task needs sub-hour granularity or heavier compute than a
  GitHub Actions job can reasonably do
- Budget allows a small paid worker instance
