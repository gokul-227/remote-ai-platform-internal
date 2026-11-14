# Read-only runtime verification — 5 November 2026

Observed at approximately **22:30–22:32 Europe/Berlin** (21:30–21:32 UTC).
No deploy, merge, production write, credential retrieval or job import was performed.

## Current routing and versions

GitHub repository variables (public URLs only) identify:

- PROD_API_BASE_URL: https://api.remoteaiplatform.com
- PROD_API_URL: https://api.remoteaiplatform.com/api/v1
- DEV_API_BASE_URL: https://remote-ai-platform-api-dev-eu.onrender.com

| Surface | Result |
|---|---|
| Production frontend /api/version | HTTP 200; production; git SHA `1df5b240b86181546943f37cfeb1650d7396d66a`; build time `2026-10-26T01:06:38Z`; deployment `40252514754-1` |
| Production API /health/version | HTTP 200; production; same git SHA |
| Production API /health/ready | HTTP 200; HEALTHY; database 179 ms and Redis 73 ms in this single sample |
| Production API /api/v1/jobs?limit=1 | HTTP 200; one public listing returned; X-Total-Count **43,163** at this sample |
| Development API /health/version | HTTP 200; same git SHA; environment reports `staging` |
| Development API /health/ready | HTTP 200 in this sample |
| Historical production onrender.com /health/version | HTTP 404; X-Render-Routing: no-server |

The historical onrender.com routing failure is not a production outage. The active
custom domain answers correctly. Onboarding and deployment handbook routing have
been corrected to the currently verified endpoints.

## Frontend asset delivery

A fresh curl GET of https://remoteaiplatform.com/ returned 7,407 bytes of HTML.
All **11 script URLs referenced by that HTML** returned HTTP 200 and content type
text/javascript through curl, including app/page and app/layout chunks. The
layout chunk embeds https://api.remoteaiplatform.com as its public API endpoint.

Python urllib's default client received HTTP 403 on the homepage and category
page during the same pass. Curl succeeded. This client-dependent result does not
establish broken production assets or a blanket crawler block; actual browser
and Google URL Inspection behavior still need verification. No claim of a full
browser journey is made from fetching scripts.

## CI and ingestion observations

The GitHub runs API reports:

- [Latest production deploy](https://github.com/gokul-227/remote-ai-platform/actions/runs/40252514754): success, 26 October; its tested SHA matches both live services.
- [Production canary](https://github.com/gokul-227/remote-ai-platform/actions/runs/42080357106): success, 5 November.
- [Scheduled source sync](https://github.com/gokul-227/remote-ai-platform/actions/runs/42074245032): success, 5 November.
- [Dev database keep-awake](https://github.com/gokul-227/remote-ai-platform/actions/runs/42034131903): failed; the readiness curl hit its 90-second timeout (exit 28). Development readiness succeeded during this later sample; this proves recovery at the sample, not that the prior failure was harmless.

Production /health/ingestion reported recent successful fetches for 13 sources,
with USAJOBS and ADZUNA most recently DISABLED. This is the deployed old source
policy: Working Nomads and unregistered Muse safeguards in the local change are
not deployed yet. API source health is history, not proof of redistribution
rights or correctness of every listing. No overdue identity erasures were
reported by this endpoint.

## Limits and launch verdict

Public production routing and deployed SHA provenance are verified. The new local
implementation has not been deployed. Authentication, multi-user journeys,
provider consent, new source-policy behavior, actual Google indexing/ranking,
backup/restore and production capacity remain outside this read-only check.

The response timestamps and job count are point-in-time observations; repeat
checks against the tested release before any go-live claim.
