# Credentials Reference

No secret values are stored in this file or anywhere in this repo. This is a map of what exists and where to go to view/rotate it. Treat every row as "compromised, rotate it" if it's ever pasted into a chat, ticket, or any git history (private repo included).

| Credential | Used for | Where it actually lives | How to rotate |
|---|---|---|---|
| `DATABASE_URL` (prod + dev) | Backend -> Postgres via Supabase pooler | Render env var (per service) | Supabase dashboard -> Project Settings -> Database -> Reset password, then update the Render env var |
| `SUPABASE_SERVICE_ROLE_KEY` / anon key (prod + dev) | Backend admin operations, frontend public client | Render env var + GitHub Actions secrets (`SUPABASE_SERVICE_ROLE_KEY`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`, `_DEV` variants) | Supabase dashboard -> Project Settings -> API |
| `JWT_SECRET_KEY` | Signs this app's own (non-Supabase) JWTs | Render env var | Generate a new high-entropy value, update Render env var -- invalidates all existing custom-JWT sessions (Supabase-issued sessions unaffected) |
| `STRIPE_SECRET_KEY` / `STRIPE_PUBLISHABLE_KEY` (LIVE) | Real payment processing | Render env var | Stripe dashboard -> Developers -> API keys -> Roll key |
| `STRIPE_WEBHOOK_SECRET` | Verifies incoming Stripe webhook signatures | Render env var | Stripe dashboard -> Developers -> Webhooks -> select endpoint -> Roll secret, then update Render |
| `GOOGLE_OAUTH_CLIENT_ID` / secret | Google sign-in | Render env var; client created in Google Cloud Console | Google Cloud Console -> APIs & Services -> Credentials |
| `MICROSOFT_OAUTH_CLIENT_ID` / secret | Microsoft sign-in | Render env var; app registered in Azure Portal | Azure Portal -> App registrations -> Certificates & secrets |
| `RESEND_API_KEY` | Transactional email | Render env var | Resend dashboard -> API Keys |
| `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY` | S3-compatible storage (Supabase Storage) | Render env var | Supabase dashboard -> Project Settings -> Storage -> S3 access keys |
| `CLOUDFLARE_API_TOKEN` | GitHub Actions deploys the frontend Worker | GitHub Actions secret | Cloudflare dashboard -> My Profile -> API Tokens -> Roll. **Update the GitHub secret immediately after rolling**, or `deploy-frontend*.yml` breaks. |
| `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_ZONE_ID` | Not secret, but kept as GitHub vars/secrets to avoid printing them in public workflow files | GitHub Actions secrets | See DEPLOYMENT_TOPOLOGY.md for the actual (non-sensitive) IDs |
| Render API key (personal, used for ad-hoc admin operations, e.g. via CLI) | Manual service/env-var management | Render dashboard -> Account Settings -> API Keys | Create a new one, delete the old one from the same page |
| `PROD_ADMIN_EMAIL` / `PROD_ADMIN_PASSWORD` (`ci-service@remote-ai-platform.internal`) | CI's scheduled job-sync auth | GitHub Actions secrets | Change the password via the app's own auth flow (or Supabase Admin API), update the GitHub secret |
| Keycloak client secret | Legacy config field only -- Keycloak itself is not deployed/used | Render env var (placeholder value, just satisfies a startup validation check) | N/A -- consider removing this config requirement entirely in a future cleanup |

## Where GitHub Actions secrets/vars are set

Repo: `gokul-227/remote-ai-platform` -> Settings -> Secrets and variables -> Actions. Split across "Secrets" (write-only, never visible again after creation) and "Variables" (visible to anyone with write access to the repo, used for non-sensitive config like the zone ID).
