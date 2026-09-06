# Secrets & Credentials

**No secret value appears in this document, or anywhere else in this repo.**
This explains *where* every secret actually lives and how to rotate each
category.

## Centralized secrets manager: Infisical

All real credential values for this project are stored in **Infisical**
(https://app.infisical.com), organization `remote-ai-platform`, project
`remote-ai-platform`, split across a `dev` and `prod` environment. This is the
single source of truth going forward — check there first for any credential's
current value, rather than hunting across GitHub/Render/Supabase/Cloudflare
dashboards individually.

Infisical was chosen over AWS Secrets Manager (a paid service that would break
this project's standing zero-paid-infrastructure constraint) specifically
because it's open-source with a genuinely free Cloud tier sufficient for a
project this size (5 machine identities, 3 projects, 3 environments — no card,
no trial period). Doppler was the other free-tier option considered; Infisical
was preferred for being open-source with a more generous machine-identity
allowance for programmatic access.

## Where secrets still physically need to live (Infisical doesn't replace these)

Storing a value in Infisical does not make GitHub Actions or Render read it
automatically — each platform still needs its own copy to actually function.
Infisical is the organized, searchable **source of truth for values**, not a
runtime substitute for:

1. **GitHub Actions repository secrets** — used during CI/CD (build/deploy
   secrets, E2E test service-role keys). Repo → Settings → Secrets and
   variables → Actions.
2. **Render environment variables** — used at runtime by the backend
   (database URL, JWT secret, Stripe keys, OAuth secrets, storage keys, etc).
   Set per-service in the Render dashboard, separately for prod and dev.
3. **Native dashboards of each third-party service** — Supabase, Cloudflare,
   Stripe, and Resend each also hold their own configuration (the keys they
   themselves issued, webhook signing secrets, OAuth provider registrations).
   These are the ultimate source of truth for anything copied elsewhere.

When you rotate a credential, update it in all the places it's actually used
(not just in Infisical) — Infisical keeping a stale value would be worse than
not having it recorded at all.

## Rotation quick-reference

| Credential | Rotate via |
|---|---|
| `DATABASE_URL` (prod + dev) | Supabase dashboard → Project Settings → Database → Reset password, then update the Render env var |
| `SUPABASE_SERVICE_ROLE_KEY` / anon / publishable keys | Supabase dashboard → Project Settings → API |
| `JWT_SECRET_KEY` (legacy custom-JWT path only) | Generate a new high-entropy value, update the Render env var — invalidates only legacy-path sessions, Supabase-issued sessions unaffected |
| `STRIPE_SECRET_KEY` / `STRIPE_PUBLISHABLE_KEY` (LIVE) | Stripe dashboard → Developers → API keys → Roll key |
| `STRIPE_WEBHOOK_SECRET` | Stripe dashboard → Developers → Webhooks → select endpoint → Roll secret, then update Render |
| Google / Microsoft / GitHub OAuth client secrets | Their respective developer consoles (Google Cloud Console, Azure Portal, GitHub OAuth Apps) |
| `RESEND_API_KEY` | Resend dashboard → API Keys |
| `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY` (Supabase Storage S3 keys) | Supabase dashboard → Project Settings → Storage → S3 access keys |
| `CLOUDFLARE_API_TOKEN` | Cloudflare dashboard → My Profile → API Tokens → Roll. **Update the GitHub secret immediately after rolling**, or the deploy workflows break. |
| Render API key (used for ad-hoc admin operations) | Render dashboard → Account Settings → API Keys — create new, delete old |

## What's NOT recommended

AWS Secrets Manager, Azure Key Vault, and similar cloud-native managed
secret stores were considered and are not used — all are paid services, which
conflicts with this project's standing zero-paid-infrastructure constraint
(every other piece of infrastructure — Cloudflare, Render, Supabase, Resend,
GitHub Actions — is deliberately kept on a free tier). Infisical's free tier
delivers the same centralized-secrets benefit at zero cost for a project this
size.
