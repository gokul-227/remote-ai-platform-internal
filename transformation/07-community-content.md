# 07 — Community content: external reading and connection cold start

Date: 2026-11-15 · Branch: `feat/community-content` (public repo) · Status: built, not deployed

## The question

The Feed (member posts) and Connections are empty on a new platform. Can they be filled from
outside sources for free, the way jobs are imported?

**Short answer.** Posts and connections cannot be imported honestly. A post is something a member
wrote, and a connection is something two members agreed to. Importing either would mean fake
authors or other people's contact graphs. What we can do:

1. **Feed:** add a separate, clearly labelled **"From around the web"** tab. It holds attributed
   links (title, short excerpt, link) from public sources whose terms allow it. Nothing there
   poses as a member post.
2. **Connections:** add an honest cold-start: "People you may know" built from on-platform
   signals only, a personal invite link, and empty states that guide people to communities,
   companies and their profile.

## Non-negotiables applied

- No fake users, no posts attributed to people, and no fake or imported connections.
- No scraping of LinkedIn, Facebook, X, Reddit or any site whose terms forbid it. Nothing from
  GitHub (owner decision: no GitHub for content or jobs).
- No contact-book uploads. No storage of other people's personal data. We don't store authors'
  names: attribution is to the source and the original link.
- Every external item is labelled "External", names its source, and links to the original.
  Items are removable (per item by staff; per source by withdrawing permission, which purges
  them). They are never indexed: `X-Robots-Tag: noindex, nofollow`, and they appear on no SEO
  page.

## Sources assessed

Rate limits and terms below reflect each source's published documentation as of 2026-11. Re-check
them before enabling a source in production. Where a figure could not be confirmed from a primary
page during this research, the table says so.

| Source | Access | Terms / licence summary | Rate limits | We store | We do not store | Decision |
|---|---|---|---|---|---|---|
| **DEV Community (Forem) API** `GET https://dev.to/api/articles?tag=remote` | Public, no key for read endpoints | Authors own their articles. The API is offered publicly, and `description` is the author's own listing summary. robots.txt (checked 2026-11-15) does not disallow `/api/articles`. | None stated in the API reference (checked). We fetch at most 4 times a day, 30 items each. | title, `description` (cut to 280 chars), canonical URL, DEV URL as the discussion link, tags, published_at | body_markdown/body_html, author name, avatar, reactions | **Enabled by default** |
| **Hacker News Search (Algolia)** `GET https://hn.algolia.com/api/v1/search_by_date?tags=story&query=remote work` | Public, no key | Titles and links to public stories. `story_text` (Ask HN bodies) is the member's own writing and is not copied. robots.txt returns 404, which means allowed. | Algolia documents 10,000 requests/hour per IP. We fetch 4 times a day. | title, story URL (or the HN item URL for Ask HN), HN discussion link, created_at | story_text, usernames, points, comments | **Enabled by default** |
| **Public RSS/Atom feeds** of engineering and remote-work blogs (first entry: GitLab blog `https://about.gitlab.com/atom.xml`) | Public feeds | Publishers syndicate feeds for headline display, but copyright stays with them. Terms differ per publisher. | Be polite: twice a day, with robots.txt honoured. | title, `<summary>`/`<description>` (stripped, 280 chars), link, categories, date | `content:encoded` / full text | **Registered but paused** (`permission_confirmed=false`). Staff confirm each feed's terms in the console before it runs. Add more feeds the same way. |
| **Stack Exchange API** | Key optional | Content is CC BY-SA 4.0. Attribution must name the author and link the author's profile and the post. ShareAlike applies to excerpts. | 300 requests/day per IP without a key, 10,000/day with a key, and the `backoff` field must be honoured (from the SE docs; their site blocked our fetch during this research, so re-verify). | — | — | **Deferred.** Proper attribution requires storing author names, which conflicts with "no personal data". Revisit with title + link only, plus a licence link. |
| Reddit | API | Data API terms restrict commercial and redistribution use and need an agreement | — | — | — | **Rejected** |
| LinkedIn, X, Facebook | — | Terms forbid scraping and redistribution | — | — | — | **Rejected** (owner rule) |
| GitHub (trending, discussions) | — | — | — | — | — | **Rejected** (owner rule: no GitHub for content) |
| Medium and newsletter platforms | RSS | Per-publication terms. Medium feeds carry full text. | — | — | — | Not added. Can be added as paused feeds if a publication allows it. |

## What was built

### External reading (`app/domains/social/news.py`)

- **Table `news_items`** (migration `072_news_and_invites`, additive, with RLS): id, source, url,
  `url_hash` (unique, sha256 of the canonical URL), discussion_url, title (300), excerpt (300),
  tags, published_at, fetched_at.
- **Canonicalisation and dedup:** http(s) only; lower-case host; fragment, tracking parameters
  (`utm_*`, `fbclid`, `gclid`, `ref`…) and trailing slash removed. The same article from DEV and
  HN is stored once.
- **Safety:** robots.txt is checked before every fetch (5xx or an unreachable robots.txt means
  don't fetch). Feeds are parsed with lxml with entity resolution, DTD loading and network
  access disabled (tested against an XXE fixture). Bodies are capped at 2 MB, with 50 items per
  run.
- **Scheduling:** each source is a `job_sources` row named `NEWS_*`, so staff use the existing
  console (`/admin/background/sources`) to pause it, change its cron, run it now, or
  confirm/withdraw permission. `due_sources` sends `NEWS_*` rows to the Celery task
  `news.import_source`. Progress is recorded in `job_import_checkpoints`, with failures,
  retries and dead letters handled as for jobs.
- **Retention:** every import prunes items fetched more than 30 days ago.
- **Removal:** staff `DELETE /api/v1/social/external/{id}`. Withdrawing a source's permission
  deletes everything that source contributed.
- **API:** `GET /api/v1/social/external` is public and paginated. It returns only items from
  enabled, permission-confirmed sources, with `{source: {label, homepage}}` attribution and
  `external: true`, and sends `X-Robots-Tag: noindex, nofollow`.
- **UI:** the Feed has two tabs, "Member posts" and "From around the web". External items show
  an "External" tag, the source name (linked), the title linked to the original, "Read on
  <source>", and an optional "Discussion" link. All links go through `safeHref` and open with
  `rel="noopener noreferrer nofollow"`. A banner says these are not posts by members. There
  are no reactions or comments on external items.

### Connections cold start (`app/domains/network/router.py`)

- **`GET /connections/suggestions`** builds "People you may know" from:
  - same organisation (owner or active member);
  - shared **public** groups (private groups are never used);
  - two or more shared skills on **public** profiles;
  - applications to the same role. This only re-orders people who already qualify through
    another signal and is **never shown as a reason**, so nobody learns where someone else
    applied.

  Excluded: you, anyone with an existing connection row in any status (so rejected requests
  aren't re-suggested), blocks in either direction, inactive accounts, staff, and hidden
  (`is_public=false`) profiles. The response contains no email addresses.
- **Personal invite links:** `POST /invites` returns the token once and stores only its sha256.
  Links last 30 days with 25 uses, members can create 5 a day (429 after that), and they can
  be listed and revoked. `GET /invites/resolve/{token}` is public and shows the inviter's name
  and headline. `POST /invites/accept/{token}` sends the inviter a **pending** connection
  request, which the inviter still approves. Blocks are respected, and using your own link is
  refused.
- **"Email" invites go through the member's own email app** (a `mailto:` link). The platform
  never collects or emails third-party addresses. That is why no unsubscribe list is needed,
  and why the existing outbox (which only ever emails registered users) stays clean. This is
  a deliberate change from the brief's "uses existing email service".
- **Empty states:** Network says "Start your network" and offers Browse professionals, Join a
  community and Browse companies. "People you may know" without signals explains where
  suggestions come from and offers Find a community, Browse companies and Add skills. An
  empty Feed links to People you may know, communities, companies and the web tab.
- **Data export:** `member_invites` is exported by `inviter_id`, without the token hash.

## Operating it

1. Deploy the migration, then the worker and scheduler. The scheduler registers `NEWS_DEVTO`
   and `NEWS_HN` as enabled, and `NEWS_RSS_GITLAB` as paused.
2. To add a feed, append a `NewsSource(kind="feed", permission_default=False)` in `news.py`.
   After reading the publisher's terms, a staff member sets `permission_confirmed=true` with a
   reason; the change is audited.
3. If a publisher objects, set `permission_confirmed=false`. Its items are purged at once.

## Not done / follow-ups

- Stack Exchange (see the table: attribution needs author names).
- Following companies: the platform has no follow model today. The empty states point to
  company pages and communities instead. A follow feature is a separate piece of work.
- Migration chain: `072_news_and_invites` revises `068_backup_runs` (the tip of
  `integrate/all`). If 069–071 land first, re-point `down_revision` when merging.
