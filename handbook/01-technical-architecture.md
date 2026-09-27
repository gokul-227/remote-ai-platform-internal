# Technical architecture

**As of 2026-09-27**, verified against the `prod` branch after the go-live hardening
(`fix/go-live-p0-containment`). Where this document and the code disagree, the code wins — update this file.

## System overview

```mermaid
flowchart LR
    Browser["Browser<br/>Next.js 16 app on Cloudflare Workers (OpenNext)"] -- REST --> API["FastAPI on Render (Oregon, US)"]
    Browser -- "email code / Google / Microsoft / GitHub" --> Auth["Supabase Auth (EU)"]
    API -- "verify ES256 tokens via JWKS" --> Auth
    API --> PG[("Supabase Postgres (eu-west-1)")]
    API --> S3[("Supabase Storage (S3 API)<br/>private resume bucket")]
    API --> Redis[("Hosted Redis<br/>rate limits · cache · chat fan-out")]
    API -- LiteLLM --> AI["Free-tier AI chain<br/>Groq gpt-oss-120b → gpt-oss-20b → Gemini → Cerebras → OpenRouter → Mistral"]
    Resend["Resend SMTP"] <-- "sign-in codes" --- Auth
    GHA["GitHub Actions cron (6h)"] -- "POST /api/v1/jobs/sync" --> API
    API --> Boards["RemoteOK · Arbeitnow · Remotive · USAJobs · The Muse"]
```

## Frontend (`apps/web`)

- The product UI is `src/figma/*`: a client-rendered, **hash-routed** app inside a thin Next.js shell
  (`src/app/page.tsx` renders it with `ssr: false`). Routes are the `REG` map in `App.tsx`.
- **Shareable resource URLs:** `#engineer/<id>`, `#jobdetail/<id>`, `#company/<id>`, `#contractsign/<id>`,
  `#group/<id>`. `readRoute()` syncs the id with the storage key each screen reads. Old path URLs
  (`/jobs/<id>` …) are redirected by `src/app/[...slug]/page.tsx`, keeping the id.
- **Role-aware navigation** (`rap_shell.tsx`): visitors see public discovery + Sign in/Join; professionals
  see Community/Careers/Workspace/Account; organisations see Community/Hiring/Workspace/Account; admins see all.
- Data access: `useApi` (in `live.ts`) over the axios client (`src/lib/api.ts`). `useApi` exposes
  `total` from `X-Total-Count` for pagination. TanStack Query is still a dependency but not used by the UI.
- Session: Supabase session in the browser; the access token is copied to localStorage for the API
  client. On 401 the client refreshes via Supabase **only if the request carried a session**; logout calls
  Supabase `signOut` (local, or global for "sign out everywhere") and clears `rap-*` UI keys.
- Linted (0 errors; `no-explicit-any` is a tracked warning in `src/figma`), type-checked (no `@ts-nocheck`),
  WCAG 2.1 AA axe scans in E2E. `Modal` is an accessible dialog (focus trap, Escape, focus restore).
- Build provenance: `/api/version` and `/health/version` report `NEXT_PUBLIC_BUILD_SHA`, env, build time and
  deployment id injected by the deploy workflow.

## Backend (`apps/api`)

Modular monolith, one package per domain under `app/domains/`: `auth, engineers, companies, jobs,
applications, saved_jobs, matching, search, network (connections + messaging + WS), notifications, social,
groups, projects, marketplace (tasks/offers/submissions), contracts, payments, trust, quality, analytics,
billing (entitlement scaffolding), admin`. All mounted under `/api/v1`.

### Identity and authorization

- **Supabase is the only sign-in path.** `authenticate_bearer_token` verifies the token (ES256/RS256 via the
  project JWKS, audience `authenticated`, issuer `<SUPABASE_URL>/auth/v1`), maps `sub` → `users.auth_subject`
  (first sign-in links an existing row by verified email or creates one), rejects inactive users, and rejects
  tokens issued before `users.sessions_revoked_at` (sign-out-everywhere). The API never issues tokens.
- Role (`ENGINEER` = professional, `COMPANY` = organisation, `ADMIN`) lives only in the `users` table;
  users may switch between ENGINEER/COMPANY, never self-assign ADMIN.
- Resource checks live in each router (contract parties, project membership, company ownership, …).
  Hidden profiles (`is_public=false`) 404 to everyone except owner, admins and companies they applied to;
  the public projection excludes resume, salary floor and the AI review.
- Self-service privacy: `GET /auth/me/export` (every row referencing the user, JSON) and
  `DELETE /auth/me` (typed confirmation; cascades; removes the stored resume). Admins can't self-delete.

### Contracts and money

- `contracts/lifecycle.py` enforces: milestones change only on ACTIVE (fully signed) contracts, in order
  PENDING → IN_PROGRESS → DELIVERED → APPROVED (professional starts/delivers; client approves or sends back);
  terms frozen after the first signature; only the client adds milestones; PAID is never set by hand.
- **Payments are switched off** (`MARKETPLACE_PAYMENTS_ENABLED=false`) on every fund/capture endpoint; refunds
  stay open. The Stripe escrow adapter exists but there is no payout path. The owner's plan: freelance work is
  paid bank-to-bank between the parties (recorded, not processed), and Stripe Checkout is used only for
  platform plans (Engineer Pro €9 / Company Pro €29) in the final phase.

### AI

- LiteLLM only. `AI_PROVIDER=auto` builds the chain from `AI_FREE_TIER_CHAIN`, skipping providers without
  their own key; one `AI_TOTAL_BUDGET_SECONDS` budget across all models; no hidden retries.
- Per-user monthly allowance (`AI_FREE_MONTHLY_TOKENS`) and platform daily cap (`AI_GLOBAL_DAILY_TOKENS`);
  429 with a clear message when exceeded; usage recorded per user in `ai_usage_logs`.
- Resume upload stores the file (private bucket, presigned 15-min download URLs), extracts text off the
  event loop, parses within `RESUME_PARSE_BUDGET_SECONDS`, fills only **empty** fields and reports
  `ai_parse_status` (parsed / failed / no_text / quota_exceeded).
- Match scores are deterministic (`matching/service.py::score`); AI is used for explanations/reviews only.

### Jobs

- Five public aggregators, all professions. Text is repaired (mojibake) and NFC-normalised; job type and
  level are never invented (`unspecified` / null); `salary_period` is explicit. `GET /jobs` returns
  `X-Total-Count` and a stable order.

### Operations

- Health: `/health/live`, `/health/ready` (DB + Redis), `/health/dependencies` (+ storage + AI chain),
  `/health/version` (git SHA). Public health errors are generic in production.
- Metrics: Prometheus exporter off unless enabled by env; request counter labelled by route template.
- Logs: structlog JSON with request ids; Cloudflare Workers Logs on; Sentry on both sides when DSNs are set.
- Rate limits: per route per client, Redis-backed with in-memory fallback.

## Data model notes

Postgres is the source of truth. Migrations `001`–`036`; the go-live ones: `033` sessions_revoked_at,
`034` ai_usage_logs.user_id, `035` job data quality (salary_period, reset invented defaults), `036`
Supabase-only auth (keycloak_id → auth_subject, drop password_hash/token_version/password_reset_tokens).
All foreign keys to `users.id` cascade (account deletion relies on it). Redis is disposable.
