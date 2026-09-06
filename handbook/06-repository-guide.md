# Repository Guide

There are two repositories. This is what belongs in each, and why.

## Public repo — `gokul-227/remote-ai-platform`

Everything that makes the product actually run: the FastAPI backend
(`apps/api`), the Next.js frontend (`apps/web`), CI/CD workflow definitions
(`.github/workflows/`), infrastructure-as-code for local development and
deployment (`infra/`), and the product's own README and contributor-facing
`CLAUDE.md`. This is *all* it contains — no business documentation, no
credential locations, no historical planning material.

This repo is public because there is no strong reason for the source code
itself to be private, and public repos get unlimited free GitHub Actions
minutes — which several parts of this project's zero-cost infrastructure
design depend on (the keep-Render-warm cron, the scheduled job-source sync,
and CI itself). Whether it should stay public remains an open question — see
"Open decisions" in `00-overview.md`.

## Private repo — this one (`remote-ai-platform-internal`)

Everything intentionally kept out of the public repo: the full handbook (this
folder). Nothing here is needed to build or run the product — it exists to
document *how the product is operated*, for people who need that context but
shouldn't need public repo access to get it, and to keep operational/business
detail (service identifiers, account ownership, business architecture) out of
a repo anyone on the internet can read. No credential values are stored here —
those live in Infisical (see `04-secrets-and-credentials.md`).

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

## How this handbook is organized

```
handbook/
  00-overview.md                          — start here
  01-technical-architecture.md            — how the system is built
  02-tech-stack.md                        — what it's built with
  03-deployment-and-infrastructure.md     — where it actually runs
  04-secrets-and-credentials.md           — how secrets are managed (never values)
  05-cicd-pipeline.md                     — how code ships, and lessons learned
  06-repository-guide.md                  — this document

  10-product-overview-and-business-architecture.md — what the product is, business flow
  11-feature-list.md                      — complete feature catalog by persona
  12-user-manual-engineers.md             — step-by-step guide for engineers
  13-user-manual-companies.md             — step-by-step guide for companies
  14-admin-manual.md                      — step-by-step guide for the platform admin
```

Files `00`–`06` are the technical/operational half (how it's built and run);
`10`–`14` are the business/product half (what it does and how to use it). This
is the entire contents of the repo — no other folders, no historical archive,
no duplicate documentation. If something in the product changes, update the
relevant file here rather than adding a new dated document alongside it.
