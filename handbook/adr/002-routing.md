# ADR-002 — Routing: hash routes now, canonical public paths incrementally

Status: accepted, 2026-09-27 (Phase 03; implementation in Phase 12).

**Context.** The product UI is a hash-routed SPA (`#jobs`, `#jobdetail/<id>`) inside one Next.js page. Links are
shareable and refresh-safe, but crawlers and link previews see one generic page, and unknown paths return 200.

**Decision.** Keep hash routing for the authenticated app. Add real path routes only for public content — job detail
first, then organisation pages and opted-in public profiles — server-rendered with their own metadata, real 404/410,
and a sitemap. Old hash links keep working through a bridge. No router rewrite, no private content in static generation
or shared caches.

**Consequences.** Two routing styles coexist for a while; public path routes must call only public API endpoints.
