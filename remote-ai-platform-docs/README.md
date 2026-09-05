# Migrated docs from the public repo

Everything under this folder was moved out of the public `remote-ai-platform` repo on 2026-09-06, at the
repo owner's request ("move all the docs, md files, screenshots ... except source code"). Paths mirror
their original location in the public repo's working tree exactly, so e.g. `docs/architecture.md` here
was `docs/architecture.md` there.

Contents:
- Root-level planning/audit markdown: `API_MAPPING.md`, `DESIGN_SYSTEM.md`, `ENTERPRISE_GAP_ANALYSIS.md`,
  `ENTERPRISE_TRANSFORMATION_PLAN.md`, `IMPLEMENTATION_PLAN.md`, `PRODUCT_AUDIT.md`,
  `SCREEN_SPECIFICATION.md`
- `docs/` — the full former `docs/` tree: architecture/current-state/audit/runbook markdown, the
  `ui-audit/` UI-verification pass (including `ui-audit/screenshots/*.png`, ~80 screenshots of the seeded
  demo app across engineer/company/admin/public views — all fake/demo data, no real credentials visible),
  and `docs/scripts/blackbox_smoke_test.py` (a QA script, not part of the app's build/runtime)
- `screenshot-pages/` — the Playwright screenshot-capture tool (`tools/capture-screenshots.js`) used to
  produce the above screenshots; a manual QA aid, never wired into CI
- `CLAUDE.md.full-original` — the full ~193-line root `CLAUDE.md` as it stood in the public repo before
  this migration. The public repo now keeps a short bootstrap version pointing back here; this is the
  complete original for reference.

Some content here (e.g. `docs/architecture.md`, `docs/CURRENT_STATE.md`) is known-stale relative to the
code at time of writing — see the "Documentation caveats" section of `CLAUDE.md.full-original`. Nothing
here was rewritten or corrected during the move; it was relocated as-is.

Note: `docs/deployment.md` etc. already existing elsewhere in this repo's own `docs/` (outside this
folder) predate this migration and are a separately-curated set — see the top-level README's "Contents"
section. Where both exist, treat the top-level `docs/DEPLOYMENT_TOPOLOGY.md` as current and anything
under this `remote-ai-platform-docs/` folder as a historical snapshot.
