# Remote AI Platform — Internal

**Private. Never make this public, never fork it publicly, never add a collaborator who shouldn't see live infrastructure details.**

This repo holds everything intentionally kept out of the public `remote-ai-platform` repo: live URLs, account/project identifiers, deployment topology, and a map of where every credential actually lives. It does **not** contain raw secret values (API keys, passwords, tokens) — those live only in Render/Supabase/Cloudflare/Stripe/GitHub's own encrypted stores. This repo tells you *where* to find or rotate them, not what they currently are.

## Contents

- `handbook/` — the final, polished technical/operational documentation package
  (architecture, tech stack, deployment, secrets, CI/CD, repo guide) written
  fresh from the current state of the code on 2026-09-06. Start at
  `handbook/00-overview.md`. A separate business/product handbook is maintained
  alongside this one and covers feature/user-facing documentation instead.
- `docs/DEPLOYMENT_TOPOLOGY.md` — the living, IDs-included operational
  reference for current live architecture, real URLs, and service/project IDs
  (prod + dev); kept current as the raw source of truth, with
  `handbook/03-deployment-and-infrastructure.md` as its polished narrative
  counterpart.
- `credentials/REFERENCE.md` — every credential in use: what it's for, where it's stored, how to rotate it
- `decisions/` — dated architectural/operational decision records (historical, not superseded by the handbook)
- `reports/` — dated point-in-time reports (historical, not superseded by the handbook)

The old `remote-ai-platform-docs/` dump (140 files moved verbatim out of the
public repo in an earlier cleanup pass) was removed on 2026-09-06 — its content
was a stale snapshot of the public repo's old docs, superseded by `handbook/`.

## Ground rules

1. Never commit an actual secret value here. If you paste one in by accident, rotate it immediately (assume it's compromised the moment it touches git history, private repo or not) and scrub the commit.
2. Keep `docs/DEPLOYMENT_TOPOLOGY.md` current — it's the source of truth for "what's actually live right now," since the public repo deliberately doesn't say.
3. Access to this repo should be limited to people who need to operate production. Review collaborator access periodically.
