# ADR-003 — Browser calls the API directly with a bearer token (no BFF for now)

Status: accepted, 2026-09-27 (Phase 03).

**Context.** The browser holds a Supabase session and sends `Authorization: Bearer` to the Render API; CORS is an
explicit allow-list. A same-origin backend-for-frontend (Worker proxy with HttpOnly cookies) would take tokens out of
JavaScript but adds CSRF defences, a proxy hop on the 10 ms-CPU Workers free tier, and a second place to get
authorization wrong.

**Decision.** Keep direct bearer calls. Mitigate token exposure with a strict CSP (no third-party scripts), no
`dangerouslySetInnerHTML` of user content, and sign-out that ends the Supabase session. Revisit if the app starts
rendering rich user HTML, adds third-party scripts, or an enterprise customer requires cookie-only sessions.

**Consequences.** Any XSS would expose the session token — CSP and output escaping stay launch-blocking controls (SEC-05).
