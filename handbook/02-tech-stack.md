# Tech Stack

Versions below were read directly from `apps/api/pyproject.toml` and
`apps/web/package.json` on the `prod` branch on 2026-09-06 — not estimated or
carried forward from memory. Where a version is prefixed `^`, that is the exact
range as pinned in the file (npm's caret ranges); everything under `apps/api` is
pinned to an exact version.

## In one paragraph, for a non-engineer

The product is a website (built with a modern web framework called Next.js) that
talks to a backend service (built with a Python framework called FastAPI). User
data lives in a managed database (Postgres, hosted by a company called Supabase,
which also handles login/sign-up). AI features (resume parsing, job enrichment)
call out to third-party AI models through an abstraction layer, so the specific
AI provider can be swapped without rewriting code. Payments, when real money is
involved, go through Stripe, the same payment processor used by most modern
online businesses. Everything is hosted on free or near-free tiers of
established cloud providers (Cloudflare, Render, Supabase) rather than
custom-managed servers.

## Backend (`apps/api`)

| Category | Technology | Version |
|---|---|---|
| Language / runtime | Python | 3.11 (pinned in `pyproject.toml` and the Docker base image) |
| Web framework | FastAPI | 0.141.1 |
| ASGI server | Uvicorn | 0.32.1 |
| Web framework internals | Starlette | 1.6.0 (pinned explicitly, ahead of FastAPI's own transitive range, to pull in fixes for several CVEs) |
| ORM | SQLAlchemy (async) | 2.0.36 |
| Postgres driver | asyncpg | 0.30.0 |
| Migrations | Alembic | 1.14.0 |
| Validation | Pydantic / pydantic-settings | 2.10.3 / 2.6.1 |
| Auth/JWT libraries | python-jose, PyJWT | 3.5.0, 2.13.0 |
| Password hashing (legacy path only) | passlib[bcrypt] | 1.7.4 |
| HTTP client | httpx | ≥0.28.0 |
| Background tasks | Celery (+ Redis transport) | 5.4.0 |
| Task monitoring | Flower | 2.0.1 |
| Object storage client | boto3 | 1.35.75 |
| AI abstraction | LiteLLM | 1.84.0 |
| Resume text extraction | pypdf, python-docx | 6.16.1, 1.1.2 |
| Caching / rate limiting | redis[hiredis] | 5.2.1 |
| Metrics | prometheus-fastapi-instrumentator | 8.1.0 |
| Structured logging | structlog | 24.4.0 |
| Payments | stripe (Python SDK) | 11.4.1 |
| Error monitoring | sentry-sdk[fastapi] | 2.68.1 (no-op until `SENTRY_DSN` is configured) |

Dev/test-only: pytest 9.0.3, pytest-asyncio, pytest-cov, ruff 0.8.3 (lint), mypy
1.13.0 (types), pre-commit 4.0.1, factory-boy/faker (test data).

## Frontend (`apps/web`)

| Category | Technology | Version |
|---|---|---|
| Framework | Next.js | 16.3.4 |
| UI library | React / React DOM | 19.2.4 |
| Language | TypeScript | ^5 |
| Styling | Tailwind CSS | v4 (`@tailwindcss/postcss`), hand-rolled design system, not a component library |
| Data fetching | TanStack Query | ^5.66.0 |
| Forms/validation | react-hook-form + zod | ^7.54.2 / ^3.24.2 |
| HTTP client | axios | ^1.7.9 |
| Auth/DB client | @supabase/supabase-js | ^2.115.0 |
| Icons | lucide-react | ^0.475.0 |
| Error monitoring | @sentry/nextjs | ^10.73.0 |
| Cloudflare deployment adapter | @opennextjs/cloudflare | ^1.20.6 |
| Cloudflare CLI | wrangler | ^4.129.0 |
| Test framework | Vitest + Testing Library + jsdom | ^5.0.0 / ^16.3.3 / ^30.0.1 |

## Database & storage

- **Postgres** — hosted by Supabase, one project for production and a fully
  separate one for the dev environment. The backend connects through Supabase's
  session-mode connection pooler (the direct `db.*.supabase.co` host is
  IPv6-only and unreachable from Render).
- **Object storage** — Supabase Storage, accessed via boto3's S3-compatible
  client (not the MinIO SDK) so the same code path works against local MinIO in
  development and Supabase Storage in production.
- **Redis** — used for caching and the distributed side of rate limiting;
  currently a single instance shared by both the dev and prod environments (a
  documented free-tier tradeoff, see `03-deployment-and-infrastructure.md`).

## AI

- **Abstraction**: LiteLLM — application code never imports a provider SDK
  directly.
- **Configured providers**: Groq (`llama-3.1-8b-instant`, primary in
  dev/prod as documented) with local Ollama models (`qwen2.5`,
  `qwen2.5-coder`, `deepseek-coder`) as fallback. The exact live production
  provider/model is an environment-variable choice (`AI_PROVIDER`/`AI_MODEL`)
  — see `04-secrets-and-credentials.md` for where that's configured; this
  handbook does not restate secret-adjacent config values.

## Payments

- **Stripe** — real, live-mode integration in production only (manual-capture
  PaymentIntents as an escrow primitive); a deterministic no-op sandbox provider
  everywhere else (always in the dev environment).

## Auth

- **Supabase Auth** — the live identity provider: email OTP (passwordless) plus
  Google, Microsoft, and GitHub OAuth.
- A legacy self-issued JWT system and a separate custom Google/Microsoft OAuth
  flow still exist in the backend codebase but are not used by the live
  frontend — see `01-technical-architecture.md` for the precise, verified
  detail.

## Monitoring & observability

- **Sentry** — two separate projects, one for the frontend (`@sentry/nextjs`),
  one for the backend (`sentry-sdk[fastapi]`); backend Sentry is a genuine no-op
  with zero overhead until `SENTRY_DSN` is set.
- **Prometheus** metrics exposed at `/metrics` on the backend
  (`prometheus-fastapi-instrumentator`).
- **structlog** for structured, request-ID-tagged backend logs.

## CI/CD & hosting

- **GitHub Actions** — all CI and deploy automation; see
  `05-cicd-pipeline.md` for the full pipeline.
- **Cloudflare Workers** (via the OpenNext adapter) — frontend hosting.
- **Render** — backend hosting (free-tier Docker web service).
- **Cloudflare** — DNS and CDN; domain registered at **Porkbun**.
- **Resend** — transactional email (OTP codes, notifications).
- Full detail on where each of these actually runs today is in
  `03-deployment-and-infrastructure.md`.

## Monorepo tooling

- **npm workspaces** + **Turborepo** at the repo root, orchestrating `apps/api`
  and `apps/web` (plus currently-empty `packages/config`, `packages/shared`,
  `packages/ui` placeholders for future shared code).
