# Remote AI Platform — Internal

**Private. Never make this public, never fork it publicly, never add a collaborator who shouldn't see live infrastructure details.**

This repo holds everything intentionally kept out of the public `remote-ai-platform` repo: live URLs, account/project identifiers, deployment topology, and a map of where every credential actually lives. It does **not** contain raw secret values (API keys, passwords, tokens) — those live only in Render/Supabase/Cloudflare/Stripe/GitHub's own encrypted stores. This repo tells you *where* to find or rotate them, not what they currently are.

## Contents

- `docs/DEPLOYMENT_TOPOLOGY.md` — current live architecture, real URLs, service/project IDs (prod + dev)
- `docs/DEPLOYMENT_ZERO_COST.md`, `docs/PRODUCTION_CERTIFICATION.md`, `docs/deployment.md` — historical deployment docs migrated from the public repo (some content predates the Cloudflare Workers migration — cross-check against DEPLOYMENT_TOPOLOGY.md for current state)
- `credentials/REFERENCE.md` — every credential in use: what it's for, where it's stored, how to rotate it

## Ground rules

1. Never commit an actual secret value here. If you paste one in by accident, rotate it immediately (assume it's compromised the moment it touches git history, private repo or not) and scrub the commit.
2. Keep `docs/DEPLOYMENT_TOPOLOGY.md` current — it's the source of truth for "what's actually live right now," since the public repo deliberately doesn't say.
3. Access to this repo should be limited to people who need to operate production. Review collaborator access periodically.
