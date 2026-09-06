# Secrets & Credentials

**No secret value appears in this document, or anywhere else in this repo.**
This is a map of *where* each category of secret lives and how the pieces fit
together — for the actual current, maintained, row-by-row map (which system
holds each specific credential and how to rotate it), see
`credentials/REFERENCE.md` in this repo, which this document intentionally does
not duplicate line-for-line so there is a single place to keep updated.

## How secrets are managed today

There are three places secrets live, by role:

1. **GitHub Actions repository secrets** — used during CI/CD: values needed to
   build and deploy (Cloudflare API token/account ID, Supabase keys needed at
   build time, service-role keys used by the E2E test suite to provision test
   users, a service account used by the scheduled job-sync workflow). Set under
   the public repo's Settings → Secrets and variables → Actions, split between
   "Secrets" (write-only, never visible again after creation) and "Variables"
   (visible to anyone with write access — used for non-sensitive config like
   the Cloudflare zone ID).
2. **Render environment variables** — used at runtime by the backend: the
   database connection string, JWT signing secret (legacy path), Stripe keys,
   OAuth client secrets, Resend API key, storage access keys, and every other
   value the running FastAPI process needs. Set per-service in the Render
   dashboard, separately for the prod and dev backend services.
3. **Native dashboards of each third-party service** — Supabase, Cloudflare,
   Stripe, and Resend each hold their own configuration natively (API keys they
   issue, webhook signing secrets, SMTP credentials, OAuth provider
   registrations) in their own respective dashboards. These are the ultimate
   source of truth for any secret that also gets copied into GitHub Actions or
   Render as an environment variable.

There is currently no centralized secrets manager — each of the three places
above is authoritative for its own slice, and `credentials/REFERENCE.md` is the
map a human uses to find or rotate any given value across all three.

## Recommendation: centralized secrets management

The repo owner has asked about a centralized secrets manager (the kind of thing
AWS Secrets Manager provides) to replace the current three-places-plus-a-map
approach.

**AWS Secrets Manager specifically is not recommended for this project as it
stands**: it is a paid service, and this project operates under a standing
zero-paid-infrastructure constraint (every current piece of infrastructure —
Cloudflare, Render, Supabase, Resend, GitHub Actions — is on a free tier by
deliberate design, as documented throughout this handbook). Introducing a paid
secrets manager would break that constraint for a benefit — centralized secret
storage — that the current map-based approach already delivers at zero
incremental engineering cost, given the project's current size.

**If that zero-paid-infrastructure constraint is ever relaxed**, two free-tier
alternatives exist that would do the same job at no cost for a project this
size, and are worth evaluating at that time rather than defaulting straight to
a paid option:

- **Infisical** — open-source, has a free tier suitable for a small team,
  integrates with GitHub Actions and can sync into Render.
- **Doppler** — has a free "Developer" tier with similar integrations.

This is intentionally left as a decision for the repo owner to make later, once
there is a concrete reason to move off the current approach (for example, more
people needing controlled access to different subsets of secrets, or secrets
sprawling across enough services that the manual map in
`credentials/REFERENCE.md` becomes hard to keep current). It is not implemented
today, and nothing in this handbook should be read as recommending it be
implemented now.
