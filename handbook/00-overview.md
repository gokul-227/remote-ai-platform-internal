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
  injection/SSRF/XSS, file-upload/AI/PII, and CI/CD/dependencies). Findings and
  fixes are described in `05-cicd-pipeline.md` and this repo's `reports/` folder.
- **A production incident happened and was resolved today**: after merging the
  day's changes, the backend started crashing on boot from an out-of-memory error
  on Render's free tier. Root cause and fix are described in
  `03-deployment-and-infrastructure.md` and `decisions/0004-render-oom-blocks-deploy.md`.
  As of this writing the fix is merged and live on both `dev` and `prod`.
- A real Stripe integration for escrow payments exists alongside a no-op sandbox
  provider, gated behind a feature flag so it is only active where explicitly
  configured (production only; the dev environment deliberately never touches
  real Stripe).

## Who should read what in this handbook

| Document | Audience | Content |
|---|---|---|
| `01-technical-architecture.md` | Engineers, technical founder | How the system is built: backend domains, auth, AI, real-time, payments |
| `02-tech-stack.md` | Everyone | What it's built with, in plain language with technical detail available |
| `03-deployment-and-infrastructure.md` | Whoever operates this next | Exactly where everything runs today |
| `04-secrets-and-credentials.md` | Whoever operates this next | Where every credential lives (never the values) |
| `05-cicd-pipeline.md` | Engineers | How code gets tested and shipped, and lessons from real incidents |
| `06-repository-guide.md` | Everyone | Why there are two repos and what belongs in each |

## A note on accuracy

Every claim in this handbook was checked directly against the `prod` branch of the
public repository (`github.com/gokul-227/remote-ai-platform`) and this repo's own
`docs/DEPLOYMENT_TOPOLOGY.md` and `credentials/REFERENCE.md`, as they existed on
2026-09-06. Where something could not be verified confidently from source, that is
stated explicitly in the relevant document rather than guessed at.
