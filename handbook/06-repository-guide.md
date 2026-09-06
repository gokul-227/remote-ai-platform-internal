# Repository Guide

There are two repositories. This is what belongs in each, and why.

## Public repo — `gokul-227/remote-ai-platform`

Everything that makes the product actually run: the FastAPI backend
(`apps/api`), the Next.js frontend (`apps/web`), CI/CD workflow definitions
(`.github/workflows/`), infrastructure-as-code for local development and
deployment (`infra/`), and the product's own README and contributor-facing
`CLAUDE.md`. As of the repo-hygiene pass completed today, this is *all* it
contains — historical planning documents, screenshots, deployment topology
details, and credential locations were deliberately moved out (see below), and
the root `CLAUDE.md` was trimmed to a short bootstrap file.

This repo is public because there is no strong reason for the source code
itself to be private, and public repos get unlimited free GitHub Actions
minutes — which several parts of this project's zero-cost infrastructure
design depend on (the keep-Render-warm cron, the scheduled job-source sync,
and CI itself). Whether it should stay public is listed as an explicitly open
question in this repo's `REQUIRED_FROM_YOU.md`, pending confirmation that
nothing about staying public creates real risk.

## Private repo — this one (`remote-ai-platform-internal`)

Everything intentionally kept out of the public repo: this handbook, decision
records (`decisions/`), dated point-in-time reports (`reports/`), and a map of
where every credential lives (`credentials/REFERENCE.md`) without ever storing
an actual secret value. Nothing here is needed to build or run the product —
it exists to document *how the product is operated*, for people who need that
context but shouldn't need public repo access to get it, and to keep
operational/business detail (real service IDs, account ownership, internal
decision-making) out of a repo anyone on the internet can read.

There is no pull-request process on this repo's `main` branch — changes are
committed directly, reflecting that this repo is documentation and operational
record, not shipped product code subject to code review.

## Why the split exists

A real commercial product with paying users and real payment processing
shouldn't have its business and operational documentation — live
infrastructure identifiers, account ownership, credential locations, internal
decisions about pricing or auth architecture — sitting in a public repository
just because its source code is public. Keeping source code open (for the
Actions-minutes and transparency benefits above) and operational documentation
private are two separate, compatible decisions; this split is how the project
implements both at once.

## How this handbook itself is organized

```
handbook/
  00-overview.md                     — start here
  01-technical-architecture.md       — how the system is built
  02-tech-stack.md                   — what it's built with
  03-deployment-and-infrastructure.md — where it actually runs
  04-secrets-and-credentials.md      — how secrets are managed (never values)
  05-cicd-pipeline.md                — how code ships, and lessons learned
  06-repository-guide.md             — this document
  assets/                            — supporting screenshots/diagrams, if any
```

A separate, business/product-facing handbook (feature lists, user-facing
walkthroughs, business architecture) is maintained alongside this one and is
out of scope here by design — this handbook stays focused on the
infrastructure- and code-facing side of the product.
