# Overview

**As of 2026-09-27** (first written 2026-09-06). This handbook is the technical/operational half of the final
documentation handoff for Remote AI Platform. It was written by reading the actual
codebase and CI/CD configuration on the `prod` branch as it stands today, not by
copying forward older documentation — several of the facts below (auth method,
domain count, CI pipeline, hosting) changed within the last 48 hours and older docs
in this repo no longer described them correctly.

## What this product is

Remote AI Platform (remoteaiplatform.com) is an AI-powered remote work marketplace for **all
professionals and organisations**. It aggregates remote jobs from public job boards, uses AI to turn
uploaded resumes into structured profiles, and computes explainable matches between professionals and
jobs. Three kinds of user — **professionals** (role `ENGINEER` in code), **organisations** (role
`COMPANY`) and **admins** — share one web application and one backend API. Contracts with milestones are
supported; payment between the parties happens outside the platform.

This document does not restate the product's features from a user's point of
view — see the companion business/product handbook for that. This one explains
how the system is built, how it is deployed, and how to operate it.

## Current status (2026-09-27)

- **Pre-launch hardening done** (branch `fix/go-live-p0-containment`, see `../go-live/`): the product is
  for **all professionals and organisations**; Supabase is the only sign-in path; money movement is
  switched off; contracts follow an enforced lifecycle; hidden profiles stay hidden; users can export and
  delete their own data; AI runs on a free-tier chain (Groq first) with per-user token allowances;
  jobs page through all results with repaired text; links are shareable; navigation is role-aware.
- **Removed old architecture:** Keycloak, the self-issued password/JWT auth, Celery workers/beat, demo
  seeding with known-password accounts, Vercel config.
- **Tests:** backend 352 (pytest), frontend 29 (vitest), 29 browser journeys + WCAG AA scans (Playwright,
  hermetic stack with a mock Supabase Auth) — all run in CI before any deploy.
- **Deploys** are serialized, pinned to the CI-tested commit, and verified by the SHA each service reports.
- **Payments** (final phase, owner decision): freelance work paid bank-to-bank between the parties;
  Stripe Checkout only for platform plans (Engineer Pro €9 / Company Pro €29) with usage limits.
- **Pending owner items:** Impressum identity (shown as "coming soon"), key rotation after launch,
  restoring branch protection after launch. See `../go-live/owner-checklist.md`.

## Open decisions

1. Operator identity for the Impressum/privacy notice (currently "coming soon") — needed before charging.
2. Stripe plans and checkout — final phase.
3. Public repo visibility — unchanged.

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

## A note on accuracy

Every claim in this handbook was checked directly against the `prod` branch of the
public repository (`github.com/gokul-227/remote-ai-platform`) as it existed on
2026-09-06. Where something could not be verified confidently from source, that is
stated explicitly in the relevant document rather than guessed at.
