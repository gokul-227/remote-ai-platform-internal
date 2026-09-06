# Start here — if you are an AI coding agent

This file exists specifically so that any AI coding agent (Claude, Codex, ChatGPT,
Antigravity, Gemini, or any other) can be handed this repository plus the source
code repository and immediately understand and operate on **Remote AI Platform**
end to end, without requiring anything further from the product owner beyond
what's described below.

**Read this file first, then read every file in `handbook/` before making any
change.** Don't skip either step, and don't rely on prior training knowledge about
what a typical project like this "usually" looks like — this handbook was written
by directly reading the actual current code and infrastructure, and several facts
about this specific project (its auth method, its exact deployment topology, its
zero-paid-infrastructure constraint) are unusual and will surprise you if you
assume a generic setup.

## The two repositories

| Repo | Contains |
|---|---|
| [`gokul-227/remote-ai-platform`](https://github.com/gokul-227/remote-ai-platform) | **All source code.** FastAPI backend, Next.js frontend, CI/CD workflows, infra-as-code. Nothing else — no business docs, no credential locations. |
| [`gokul-227/remote-ai-platform-internal`](https://github.com/gokul-227/remote-ai-platform-internal) | **This repo.** Everything else: how it's built, deployed, operated, and used — for humans and for you. No pull-request process here; changes commit directly to `main`. |

Clone or read both. The source repo alone does not tell you where anything is
deployed, what's live, or how to get the credentials you'll need — this repo is
required context, not optional background reading.

## What this product is, in one paragraph

Remote AI Platform (`remoteaiplatform.com`) is an AI-powered remote-engineering
job marketplace: it aggregates jobs from external boards, uses AI to parse
resumes into structured profiles, and computes explainable matches between
engineers and jobs. Three user types (engineers, companies, admins) share one
Next.js frontend and one FastAPI backend. It is a real product with real users
and real Stripe payments live in production — treat it accordingly (see
`handbook/01-technical-architecture.md` and onward for the full picture).

## Live URLs, right now

| Environment | Frontend | Backend API |
|---|---|---|
| Production | https://remoteaiplatform.com | https://remote-ai-platform-api.onrender.com |
| Development | https://dev.remoteaiplatform.com | https://remote-ai-platform-api-dev.onrender.com |

Backend health check on either: `GET /api/v1/health`. Full deployment topology
(hosting providers, service names/IDs, DNS, everything) is in
`handbook/03-deployment-and-infrastructure.md` — that document is the actual
source of truth for "what's live," since the source-code repo deliberately
doesn't say.

## How to get credentials — you should not need to ask a human

All real secret values (API keys, database URLs, OAuth secrets, everything) live
in **Infisical** (https://app.infisical.com), organization/project
`remote-ai-platform`, split into `dev` and `prod` environments. See
`handbook/04-secrets-and-credentials.md` for the full explanation of what lives
there and what still has to be separately configured in GitHub Actions/Render.

If you are an agent starting completely fresh with no Infisical access yet: ask
the product owner, once, for an Infisical Machine Identity Client ID + Client
Secret scoped to this project (Infisical → Access Control → Machine Identities →
Create Identity, Universal Auth). That is the only credential you should ever
need to request — once you have it, authenticate via Infisical's Universal Auth
API and read every other secret directly from there. Do not ask for individual
API keys one at a time; that Machine Identity is the single key that unlocks
everything else.

## Before you touch anything — read these, in order

1. `handbook/00-overview.md` — current status, what changed most recently, open
   decisions the product owner hasn't resolved yet
2. `handbook/01-technical-architecture.md` — how the backend/frontend/auth/AI
   layer are actually built (not how a similar project is typically built —
   this one has real, documented deviations from convention)
3. `handbook/02-tech-stack.md`
4. `handbook/03-deployment-and-infrastructure.md`
5. `handbook/04-secrets-and-credentials.md`
6. `handbook/05-cicd-pipeline.md` — includes real incidents from this project's
   own history worth not repeating (a memory-limit crash, a broken deploy
   pipeline, a fork-PR deploy-hijack risk that was closed)
7. `handbook/06-repository-guide.md`
8. `handbook/10-product-overview-and-business-architecture.md` through
   `handbook/14-admin-manual.md` — the business/product/user-facing side

## Ground rules that apply regardless of which agent you are

- **Zero paid infrastructure.** Every single piece of this project's
  infrastructure — Cloudflare, Render, Supabase, Resend, GitHub Actions,
  Infisical — is deliberately on a free tier. Do not introduce a paid service or
  upgrade a plan without the product owner's explicit sign-off, even if a paid
  tier would obviously solve a problem faster.
- **Branch model**: `prod` (live production) ← `dev` (integration) ← feature
  branches. Never push directly to `prod` or `dev` — always a PR, and PRs into
  `prod` must come from `dev` (enforced by CI, not just convention).
  Merging into `dev`/`prod` auto-deploys to the matching environment — verify
  the deploy actually succeeds (check the live health endpoint) rather than
  assuming a merge is the end of the job.
- **Never commit a secret value** to either repository. Real values live in
  Infisical only.
- **Verify claims against the live code and running services**, not against
  what a document (including this one) says was true as of when it was
  written — this handbook is dated and was accurate at the time, but the
  product keeps shipping. If something you observe contradicts a document
  here, trust the live system and update the document.
- Real users, real payments, real production system — the same engineering
  care (testing, security-mindedness, not breaking what already works) that
  applies to any live commercial product applies here.
