# CI/CD Pipeline

Verified directly against `.github/workflows/` on the `prod` branch of the
public repo on 2026-09-06.

## What runs on every pull request

`ci.yml` runs on every push to `prod`/`dev` and on every pull request, with four
jobs:

- **Backend** — Ruff lint, mypy type check, then the real test suite run
  inside the Docker Compose `test` service (not a mocked environment).
- **Frontend** — `npm ci` from the repo root (this is an npm-workspaces
  monorepo; the lockfile lives at the root, not inside `apps/web` — running
  `npm ci` from `apps/web` fails), then lint, `tsc --noEmit`, and a production
  build.
- **E2E** — Playwright, run against a full `docker-compose` stack (backend +
  frontend + Postgres + Redis + MinIO), with `AUTH_PROVIDER=supabase` explicitly
  set so CI exercises the same auth path that is actually live in production,
  not the legacy local-only path. Demo data is seeded, and a matching Supabase
  Auth identity is created for the CI admin fixture account, since the frontend
  now authenticates exclusively through Supabase. Individual E2E flows that
  need a real sign-in (registration, the admin journey) create and confirm
  users directly against the Supabase Auth API rather than reading a real
  inbox, since there is no inbox available in CI.
- **Migration validation** — runs `alembic upgrade head` against a fresh
  Postgres 16 instance, catching migration errors independently of the rest of
  the test suite.

Beyond `ci.yml`, three more workflows run on pull requests / pushes as security
gates: **CodeQL** (`codeql.yml`, static analysis for common vulnerability
classes), **gitleaks** (`gitleaks.yml`, secret-scanning), and **dependency
review** (`dependency-review.yml`, flags newly-introduced vulnerable
dependencies on a PR). All CI jobs run with explicit least-privilege
`permissions: contents: read` — none of them need to push commits or otherwise
write to the repo.

## Branch flow enforcement

`enforce-branch-flow.yml` runs on every pull request targeting `prod` and
rejects it unless its source branch is exactly `dev` — the promotion path is
feature branch → `dev` → `prod`, with no exceptions, enforced by CI rather than
relying on people remembering the convention.

## What happens on merge to `dev` vs `prod`

- Merging to **`dev`**: `ci.yml` runs again on the merge commit; Render's git
  integration auto-deploys the dev backend service; `deploy-frontend-dev.yml`
  deploys the dev Cloudflare Worker once CI succeeds on that push.
- Merging to **`prod`**: same shape — Render auto-deploys the prod backend
  service on the push; `deploy-frontend.yml` deploys the prod Cloudflare Worker
  once CI succeeds on that exact push event.

Both frontend deploy workflows trigger on `workflow_run` (CI completing) rather
than directly on `push`, specifically so a broken merge to a protected branch
never gets deployed just because it merged — the deploy only fires if CI
actually passed on that commit.

## Real incidents from today, worth learning from

**1. A fork PR could have hijacked a production deploy using this repo's real
secrets.** The frontend deploy workflows originally triggered on any `CI`
`workflow_run` whose head branch matched `prod`/`dev`. `CI` itself also runs on
`pull_request`, including from forks — and for a PR-triggered run, the "head
branch" GitHub reports is the *contributor's own branch name in their fork*. An
attacker could have named a fork branch literally `prod`, opened a PR, and once
that PR's CI run completed, produced a `workflow_run` event matching
`branches: [prod]` — letting the deploy job check out and deploy the attacker's
own fork commit using this repository's real Cloudflare, Sentry, and Supabase
secrets. **Fixed** by restricting both deploy workflows to only act on
`workflow_run` events where `github.event.workflow_run.event == 'push'` (a push
event can only originate from someone with actual write access to this
repository, never from a fork's pull-request run), with a redundant
`head_repository` check kept as defense in depth.

**2. An npm version bump silently broke the frontend build.** npm 12 changed
its default behavior around install scripts (blocking them unless explicitly
allowlisted), which broke the Cloudflare Workers deploy pipeline with no clear
signal beyond a failed install step. Fixed by explicitly allowlisting the
required install scripts.

**3. A redundant migration re-run caused a memory-limit crash on every
boot.** Described in full in `03-deployment-and-infrastructure.md`: an
in-process Alembic migration run during application startup, on top of the
migration Render already performs separately, pushed memory usage over Render
free tier's 512MB ceiling on every fresh process start — crashing the backend
on both `dev` and `prod` shortly after a large batch of changes merged. Fixed
by removing the duplicate in-process run. This blocked deploying several other
already-merged fixes (including the WebSocket auth fix) until it was resolved,
which is the practical lesson: a startup-time regression on a memory-constrained
free tier can block an otherwise-unrelated release train, so changes that add
work to application startup deserve real scrutiny on this project specifically.

## For whoever maintains this next

- Treat the free-tier 512MB RAM ceiling on Render as a real constraint on every
  change that touches application startup or adds a dependency, not a one-time
  historical note — see incident 3 above.
- The branch-flow and fork-deploy protections exist because this repository is
  public; removing them (or making the public repo private — see "Open
  decisions" in `00-overview.md`) would change the actual risk calculus here
  and is worth revisiting together with that decision.
- CodeQL, gitleaks, and dependency-review were added specifically as part of
  today's security audit, alongside the fork-hijack fix — they are new, not
  long-standing.
