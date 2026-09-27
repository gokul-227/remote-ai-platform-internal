# ADR-001 — Data fetching in the product UI

Status: accepted, 2026-09-27 (Phase 03).

**Context.** `apps/web/src/figma` fetches with a small `useApi` hook (axios, no cache library); `@tanstack/react-query`
is installed but unused by the product UI. `useApi` used to show the previous resource's data while a new one loaded or
failed, and did not scope data to the signed-in account.

**Decision.** Keep `useApi` for now; it is keyed by account + path + params (`b83639d`), so another resource's or
account's data is never shown. Do not migrate to React Query while routing, design and auth are also changing.
Revisit when a screen needs shared caching, optimistic updates or background refetch across screens.

**Consequences.** Paginated lists show their loading state between pages. Each screen owns its requests; there is no
cross-screen cache to invalidate (and none to leak across accounts).
