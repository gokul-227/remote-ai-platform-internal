# Job discovery and source policy checkpoint — 5 November 2026

Scope: existing adapters in the product checkout. This is a source/documentation review and local test record, not evidence of successful production ingestion or an assertion of rights over all internet job listings. No paid provider account or subscription was created.

## Important findings and implemented boundaries

- Remotive permits attributed display but prohibits syndication to Google Jobs and using its listings as a gate for collecting signups or email addresses. Its official README recommends at most four data fetches per day. The production scheduler now selects it at six-hour intervals; manual runs remain available. Public application links must work without registration.
- Himalayas permits public job-board use with a visible source link and unchanged applicationLink. It prohibits forwarding to Google Jobs, including through JobPosting structured data. Its browse API now recommends opaque cursors; offset is deprecated.
- Working Nomads has a publicly readable JSON endpoint, but its current terms prohibit using the service to build a competing product. The adapter now requires explicit configured permission, default false.
- Adzuna's public API is not an indefinitely free commercial feed. Current terms limit evaluation and require an agreement for ongoing commercial/lead-generating use. The adapter now requires confirmed licence configuration, default false, in addition to its keys.
- The Muse requires application registration for production use. Production now disables the adapter when no registered API key is supplied; testing remains possible without a key.
- All four of REMOTIVE, HIMALAYAS, WORKINGNOMADS and ADZUNA are excluded from search-engine notifications and sitemap entries/counts. This intentionally stays conservative even if a permission flag is later enabled; indexing policy needs its own review.

## Existing-source matrix

| Source | Official reference inspected | Observed permission and remaining work | Implementation state |
|---|---|---|---|
| Remote OK | [API legal notice](https://remoteok.com/api) | Requires named attribution and a follow backlink; logo needs written permission. Preserve original listing URL. | Existing adapter. Parent handles visible source attribution and direct apply. |
| Arbeitnow | [Job Board API](https://www.arbeitnow.com/blog/job-board-api) | Official public no-key API intended for job-board integrations. Owner must retain upstream attribution and review any separate account-specific agreement. | Existing paged adapter retained. |
| Remotive | [Official API README](https://github.com/remotive-com/remote-jobs-api) | Attributed on-site display; no Google Jobs redistribution; no signup gate to see listing. No more than four scheduled fetches/day. | Existing adapter, scheduler throttled; search syndication blocked. |
| The Muse | [API documentation](https://www.themuse.com/developers/api/v2) | API is public, but use beyond testing requires a registered app. Terms link could not be read during this pass; no claim of unrestricted redistribution. | Production disabled without THEMUSE_API_KEY; page index fixed to zero-based first page. |
| USAJOBS | [API access terms](https://developer.usajobs.gov/apirequest/index), [OPM developer overview](https://www.opm.gov/developer/) | OPM describes expanding reach to commercial boards. Requires issued API credentials and compliance with applicable use limits. | Existing key-gated adapter retained; actual key/account state unverified. |
| Jobicy | [Developer API and fair-use documentation](https://jobicy.com/jobs-rss-feed) | Explicitly permits commercial boards and redistribution. Public apply URL must remain the Jobicy URL. API has a seven-day discovery window; absence does not establish closure. Commercial direct ATS URLs cost money and were not enabled. | Migrated to current free /api/jobs endpoint and cursor pagination. |
| Himalayas | [Remote Jobs API](https://himalayas.app/api) | Explicit attributed display permission; unchanged applicationLink; no Google Jobs/JobPosting redistribution; remove expired jobs. | Cursor pagination replaces deprecated offset; search syndication blocked. Closed-job validation remains a launch task. |
| Working Nomads | [Terms](https://www.workingnomads.com/terms-and-conditions), [public endpoint](https://www.workingnomads.com/api/exposed_jobs/) | Public accessibility does not override competing-product restriction. Obtain written permission before activation. | Default disabled with WORKINGNOMADS_PERMISSION_CONFIRMED=false. Existing records preserved. |
| Hacker News | [Official API](https://github.com/HackerNews/API), [Algolia endpoint documentation](https://hn.algolia.com/api) | Public API existence verified; API software licence does not establish blanket ownership of employers' post content. Retain original HN post link; review content redistribution scope. | Existing Who Is Hiring adapter retained; no scraping or private data access. |
| Greenhouse | [Job Board API](https://docs.greenhouse.io/job-board), [official overview](https://support.greenhouse.io/hc/en-us/articles/10568627186203-Greenhouse-API-overview) | Public job-board GET data supports career sites. This is not evidence of a blanket licence to resyndicate every employer's descriptions to third-party job engines. Use configured, permitted employers and original links. | Existing configured-board adapter; empty boards no longer misreported as outages. |
| Lever | [Official postings API](https://github.com/lever/postings-api/blob/master/README.md) | Designed for employer job-site integration and hosted application links. Employer-specific redistribution rights remain to be confirmed. | Existing adapter; explicit onsite/hybrid workplaceType now overrides incidental remote words. |
| Ashby | [Official public job-posting guide](https://docs.ashbyhq.com/using-the-lightweight-job-posting-api-to-list-openings-on-your-site) | Public API supports an employer's careers page and application links. No blanket third-party Google syndication licence established here. | Existing adapter; explicit isRemote=false now preserved. |
| SmartRecruiters | [Posting API](https://developers.smartrecruiters.com/docs/posting-api) | Public posting API for career-site integrations. Employer-specific redistribution permission remains unverified. | Existing adapter; explicit remote=false now preserved. |
| Adzuna | [API terms](https://developer.adzuna.com/docs/terms_of_service), [overview](https://developer.adzuna.com/overview) | Production commercial use needs appropriate licence; attribution requirements also apply. Free evaluation is not the requested zero-cost production solution. | Default disabled with ADZUNA_LICENSE_CONFIRMED=false and search syndication blocked. |
| Jooble | [REST API overview](https://jooble.org/api/about) | Officially supports showing job-search results on other websites. Issued key/account quota and any contractual conditions must be checked in the actual account. | Existing key-gated adapter retained; no new account or paid usage enabled. |

## Correctness and reliability changes

1. Cursor walks are bounded by requested record count, preserve opaque tokens, stop at the end and reject repeated/invalid cursors. Bad responses are not converted into a healthy empty feed.
2. Public ATS payloads validate list shape. A legitimately empty company board is a successful empty response; a board transport/shape failure is distinguished.
3. Explicit false remote flags win over location/title heuristics. Missing location is shown as unspecified, rather than asserting remote work.
4. A failed row save cannot trigger expiry of unseen inventory or log a fully successful source. Successfully saved rows remain preserved.
5. Search notifications collect IDs only after source transactions commit and exclude restricted source feeds.
6. CLI imports enable strict failure propagation, so the scheduled workflow's failure-counting logic can observe a failing source instead of receiving exit code zero for every run.
7. Search Console OAuth requests now use their requested read-only scope. Previously _google_token always issued the Indexing API scope, making Search Console reporting fail despite configured credentials.

## Local verification

Focused pytest commands use DEBUG=false because the host's inherited DEBUG=release is not a valid Pydantic boolean. This is an environment issue, not a committed secret or runtime setting change.

- Focused suites: test_public_job_boards.py, test_ingestion_outcomes.py, test_search_indexing.py, test_sitemap_entries.py, test_job_sync_source_filter.py.
- Final focused run: **56 passed in 4.47 seconds**.
- Ruff passed for the owned code/tests after correcting existing parameterization/assert formatting in the touched ingestion test file.
- These are mocked HTTP and SQLite tests. No production import, PostgreSQL concurrency run, source account acceptance, Google index inclusion or ranking is proven by them.

## Remaining launch work

- Verify source account permissions and actual production configured keys through the approved secret manager; do not paste values into documentation.
- Add closed-job reconciliation using Jobicy's status endpoint and Himalayas' per-job availability endpoint. The existing last-seen freshness rule is not proof that an upstream job closed.
- Review employer permissions for configured ATS boards and decide which are eligible for Google Jobs syndication.
- Apply any necessary takedown of existing uncleared listings through a reviewed administrative operation; this work preserves existing data and does not silently delete production records.
- Submit only eligible pages after deployment, validate Google's Rich Results output, and inspect actual indexing in Search Console. No search rank is guaranteed.

Rollback: revert the isolated adapter/service/runner/config changes through feature→dev→prod flow. Keep source permission gates and restricted-source exclusions in place unless a reviewed replacement policy is deployed. No schema migration is required.
