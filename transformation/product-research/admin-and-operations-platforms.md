# Administrative and operational platforms — Jira Service Management, ServiceNow, Grafana, Datadog/New Relic, Backstage, Keycloak/Okta/Entra, WorkOS (research package, 14 Nov 2026)

Evidence labels: **[Documented]** official documentation pages fetched 14 Nov 2026; **[Inferred]**; no REA observation (none of these are installed or open; the owner's Jira site has no JSM space).

## Jira Service Management
- SLA model: goals grouped by priority and JQL, calendars, conditions (start/pause/stop), display formats, auto-close resolved requests/incidents, SLAs on incidents, portal timelines **[Documented index pages only; article bodies render client-side]**.
- Request types, queues, internal notes vs customer replies, assignment, escalation are the standard JSM concepts **[Inferred from navigation]**.
- Mapping: support desk with request types, queues (unassigned, mine, breached), SLA timers (first response, resolution) with pause on "waiting for customer", internal notes, customer replies, escalation to incident.

## ServiceNow (ITSM incident management)
- Incidents are logged, classified, prioritized, assigned to users/groups, escalated, investigated; major incidents follow a separate shorter-timescale procedure; dashboards/reports track status and service levels **[Documented overview]**.
- Mapping: internal `incidents` module: severity, status (open → acknowledged → mitigating → resolved → closed), assignee, linked tickets/services, timeline/notes, post-incident summary; change records out of scope initially; audit trail on every state change.

## Grafana Cloud
- Components: metrics/logs/traces/profiles backends, dashboards, alerting, SLOs, IRM (on-call, incidents), synthetics, k6; ingestion via Grafana Alloy (OTel distribution), direct OTLP, Prometheus-compatible **[Documented]**.
- Free tier: 10k metric series, 50 GB logs, 50 GB traces, 50 GB profiles, 14-day retention, 3 users, 3 IRM users, 500 k6 VU-hours, no card **[Documented pricing]**.
- Mapping: the API already exposes Prometheus metrics; add Alloy sidecar/container in the Compose stack to remote-write to Grafana Cloud once the owner creates the stack; dashboards for API latency, worker/queue depth, import outcomes; alert rules to email; the admin "Service health" screen links to Grafana rather than re-implementing it.

## Datadog / New Relic
- APM, error tracking, traces, service maps, alerts **[Common; official pages not fetched]**. Sentry already covers error tracking. No additional provider is added (plan: avoid redundant monitoring).

## Backstage
- Software catalog: entities described by YAML metadata in source control, ownership per component, default view "owned by my team", registration by URL/templates/static config **[Documented overview]**.
- Mapping: lightweight `service_catalog` in the admin app: components (web, api, admin, worker, scheduler, redis, postgres, storage, email, AI providers, job sources), owner, runbook link, health endpoint, dependencies; sourced from a checked-in `catalog.yaml` in the product repo plus runtime health.

## Keycloak / Okta / Entra ID
- Keycloak: realms, clients, realm/client/composite roles, groups, identity brokering with IdP mappers, OTP/passkeys/recovery codes, sessions with revocation and timeouts, user federation, user and admin event auditing **[Documented]**.
- Okta/Entra: SSO, MFA, conditional access, lifecycle, audit **[Common; not fetched]**.
- Mapping: least-privilege staff roles (Super Administrator, Operations Administrator, Customer Support, Trust and Safety Moderator, Engineering Operator, Read-only Auditor) as a `staff_roles` table with explicit owner provisioning; session revocation and admin audit events already exist and are extended.

## WorkOS AuthKit (identity candidate)
- Pricing: first 1M MAU free; SSO/Directory Sync per connection ($125 each for 1–15); staging environments free; only production billed; MFA listed as included **[Documented pricing]**. Card requirement for production not stated on the page **[Unknown]**.
- Capabilities page fetched was the getting-started page (installer CLI, device auth flow); user import/migration and JWKS specifics must be read in later Phase 7 work **[Unknown here]**.
- Mapping: Phase 7 compatibility report; dual-validation (accept Supabase and WorkOS issuers during transition), identity mapping on `users.auth_subject` + verified email; never delete Supabase users; rollback = disable WorkOS issuer.

## Acceptance criteria (admin application, Phase 15)
- Every admin endpoint checks a staff role server-side; public signup never yields a staff role; auditor role can read everything and change nothing (tests).
- Support SLA timers computed server-side and shown breached/at-risk; incidents link to tickets and services; feature flags are persisted with audit reason; queue/worker/import monitoring reads real tables (Phase 5).
- Dashboard numbers are database aggregates with timestamps; no placeholder values.
