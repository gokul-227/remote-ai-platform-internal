# Overview

**As of 2026-09-06.** This handbook is the technical/operational half of the final
documentation handoff for Remote AI Platform. It was written by reading the actual
codebase and CI/CD configuration on the `prod` branch as it stands today, not by
copying forward older documentation — several of the facts below (auth method,
domain count, CI pipeline, hosting) changed within the last 48 hours and older docs
in this repo no longer described them correctly.

## What this product is

Remote AI Platform (remoteaiplatform.com) is an AI-powered remote engineering
marketplace. It aggregates remote software engineering jobs from external job
boards, uses AI to turn uploaded resumes into structured engineer profiles, and
computes explainable matches between engineers and jobs. Three kinds of user —
**engineers** (job seekers), **companies** (employers), and **admins** — share one
web application and one backend API. A longer-term goal, partially built, is to
support paid contract work end-to-end (escrow payments, milestones, contracts)
rather than being a pure job board.

This document does not restate the product's features from a user's point of
view — see the companion business/product handbook for that. This one explains
how the system is built, how it is deployed, and how to operate it.

## Current status (2026-09-06)

- **Live in production** at `remoteaiplatform.com`, serving real users.
- **Authentication was overhauled today**: password-based login has been replaced
  with passwordless email one-time-code (OTP) sign-in, plus Google, Microsoft, and
  GitHub OAuth, all via Supabase Auth. See `01-technical-architecture.md` for the
  precise current state, including a legacy code path that still exists but is no
  longer the active flow.
- **A real-time messaging/notifications bug was fixed today**: WebSocket
  connections had been silently broken for every real user since the switch to
  Supabase-issued tokens (they were still being checked against the old
  password-based token verifier). This is now fixed and working.
- **A security audit covering the whole codebase was completed today** across six
  focused passes (auth/IDOR/WebSockets, payments/storage, CORS/headers/rate-limits,
  injection/SSRF/XSS, file-upload/AI/PII, and CI/CD/dependencies). A summary of
  findings is in `05-cicd-pipeline.md`.
- **A production incident happened and was resolved today**: after merging the
  day's changes, the backend started crashing on boot from an out-of-memory error
  on Render's free tier (root cause: a redundant database-migration re-run on every
  startup, doubling the app's import-time memory footprint). Fixed and merged; the
  fix is live on both `dev` and `prod` — see `03-deployment-and-infrastructure.md`
  for detail.
- A real Stripe integration for escrow payments exists alongside a no-op sandbox
  provider, gated behind a feature flag so it is only active where explicitly
  configured (production only; the dev environment deliberately never touches
  real Stripe).
- **All real secret values now live in Infisical**, a centralized free-tier
  secrets manager (organization/project `remote-ai-platform`, `dev`/`prod`
  environments) — see `04-secrets-and-credentials.md`. This replaced a
  scattered "check GitHub/Render/Supabase/Cloudflare individually" approach.
- **Deploy automation was fixed today**: Render's own GitHub-integration
  auto-deploy had been silently unreliable (merges sat undeployed for hours
  with no visible error). Both `dev` and `prod` now deploy the backend via an
  explicit Render deploy-hook call from the same GitHub Actions pipeline that
  already deploys the frontend — see `05-cicd-pipeline.md`.
- **This repository itself was rewritten today** from a pile of dated,
  historical working documents into the current single handbook described
  below, and an `AGENT-ONBOARDING.md` was added at the repo root specifically
  so any AI coding agent — not just a human — can pick up both repositories
  and operate on the product without further input from the product owner.

## Who should read what in this handbook

| Document | Audience | Content |
|---|---|---|
| `../AGENT-ONBOARDING.md` | AI coding agents | Start here if you're an agent, not a person — everything needed to operate on this product cold |
| `01-technical-architecture.md` | Engineers, technical founder | How the system is built: backend domains, auth, AI, real-time, payments |
| `02-tech-stack.md` | Everyone | What it's built with, in plain language with technical detail available |
| `03-deployment-and-infrastructure.md` | Whoever operates this next | Exactly where everything runs today |
| `04-secrets-and-credentials.md` | Whoever operates this next | Where every credential lives (never the values) |
| `05-cicd-pipeline.md` | Engineers | How code gets tested and shipped, and lessons from real incidents |
| `06-repository-guide.md` | Everyone | Why there are two repos and what belongs in each |

## Open decisions

Things the codebase and infrastructure support either answer to, deliberately
left for the product owner to decide rather than resolved unilaterally:

1. **Legacy auth consolidation** — a self-issued custom-JWT auth path and an
   unused custom OAuth broker still exist in the backend alongside the live
   Supabase-based auth (see `01-technical-architecture.md`). Delete now, or
   leave until there's a concrete reason to touch it?
2. **Legal content identity fields** — the Impressum page needs a real legal
   name/address (or a decision to wait until a company entity is formed)
   before a lawyer reviews the AI-drafted legal pages.
3. **Pricing/plans** — the billing domain has entitlement-scaffolding models in
   code, but zero real prices are set; the current stance is "everything free,
   gate features and charge later."
4. **Public repo visibility** — whether `gokul-227/remote-ai-platform` should
   become private, pending confirmation nothing about staying public creates
   real risk (see `06-repository-guide.md`).

## A note on accuracy

Every claim in this handbook was checked directly against the `prod` branch of the
public repository (`github.com/gokul-227/remote-ai-platform`) as it existed on
2026-09-06. Where something could not be verified confidently from source, that is
stated explicitly in the relevant document rather than guessed at.
