# REA evidence matrix — external products (2026-10-25)

REA `rea-agents@6.7.0`, MCP server `rea` connected in the VS Code Claude Code session.
Owned-fixture test: `analyze_javascript_application` on `apps/admin/src` → **succeeded**
(`ev_3dc92361…`, 8 files, 93 findings).

## What REA was and was not used for

REA was used through `capture_browser_scenario` (headless Chromium, no account, no cookies, one page
each, accessibility snapshot only) on **official public help/documentation pages**, and through
`analyze_javascript_application` on an **openly licensed** product artifact (Jitsi SDK, Apache-2.0).

REA was **not** used to decompile, extract bundles or source maps, or inspect the internals of any
proprietary product. Reasons: REA's own DISCLAIMER requires authorization per target ("public
availability … does not by itself establish permission"); the implementation plan §3 rules 7–9
forbid harvesting competitor code/source maps; the products' terms forbid it. Disabling Claude Code
permission prompts does not create third-party authorization. Minimum change to unblock deeper
analysis of a given product: written permission from that vendor (or a licence that allows it).

## Matrix

| Product | Feature | REA method | Target | Evidence | Verified (observed) | Inferred / uncertain | Authorization | REA status | Implementation | Tests |
|---|---|---|---|---|---|---|---|---|---|---|
| Slack | Channel notifications | capture_browser_scenario | slack.com help article 360056534254 | `ev_f501a80f…` | Per-conversation "All new posts" / "Just mentions"; mute turns notifications off entirely; reset to default; badge-every-unread and follow-every-thread options | — | Public help page | **Completed** | Channel notify level ALL/MENTIONS/NOTHING; mute also suppresses mentions (migration 050) | `test_channel_notification_levels` ✅ |
| Jira | Watching, comments | capture_browser_scenario | support.atlassian.com "Watch, share and comment on a Jira work item" | `ev_88539ad6…` | Watchers notified on comment/edit/transition; optional auto-watch on items you comment on or update; @mentions notify; comment reactions; activity sort; voting | Default of auto-watch preference not stated | Public help page | **Completed** (first URL 404, `ev_e6807d33…`) | Commenting/updating auto-watches; watchers notified about comments | `test_commenting_watches_and_watchers_hear_about_comments` ✅ |
| GitHub | Code deliverable integration | capture_browser_scenario | docs.github.com "About creating GitHub Apps" | `ev_1d0acc8e…` | Apps installed on accounts with access to specific repositories; built-in webhooks; narrow permissions; user-to-server vs installation tokens | — | Public docs (open-source docs) | **Completed** | Design confirmed for the deferred GitHub App; commit-SHA deliverables already shipped | Existing `test_deliverables.py` ✅ |
| Zoom-equivalent (Jitsi) | Meeting integration | analyze_javascript_application | `@jitsi/react-sdk@1.4.4` (Apache-2.0) | `ev_3a648486…` (26 files, 98 findings) | Iframe via `external_api.js` on a configurable domain; props `roomName`, `jwt`, `userInfo.displayName`, `configOverwrite`, `invitees`; recording via JWT/JaaS | Moderation needs a JWT-capable (non-free) host | Licence permits | **Completed** | Join link pre-fills display name; organiser add/remove participants; reschedule | `test_meeting_participants_and_reschedule` ✅ |
| Zoom | Participant management | capture_browser_scenario | support.zoom.com KB0066860 | `ev_9d43826c…` | Wrong article (French voice-commands page) | — | Public help page | **Partial** (no usable finding) | — | — |
| Upwork | Invitations, proposals | capture_browser_scenario | support.upwork.com article 211063018 | `ev_489b685c…` | Cloudflare "Performing security verification" page only | — | Bot protection | **Attempted but blocked** (external service restriction; not bypassed) | From public-doc research (web search of Upwork help/community): decline with reason, client withdraws invitation, proposals withdrawn/resubmitted → invitation expiry (7 d, chosen by us), decline reasons, cancel invitation, proposal edit before review | `test_invitation_*`, `test_proposal_revision_only_before_review` ✅ |
| Brave | Safe demo viewing | capture_browser_scenario | brave.com/shields | `ev_efb46cce…` | Page loaded (URL only captured) | — | Public page | **Partial** | Existing: http(s)-only demo links, private hosts refused, `noopener` | Existing deliverable tests ✅ |
| Facebook | Reactions | not run | — | — | — | — | Meta terms prohibit automated access without permission | **Not attempted — blocked by authorization** | From public-doc research (news coverage of the reactions launch): fixed reaction set, one per person, changeable → our own professional reaction set | `test_reactions_*` ✅ |
| LinkedIn | Profiles, jobs | not run | — | — | — | — | LinkedIn User Agreement prohibits bots/automated access | **Not attempted — blocked by authorization** | No new work (existing profiles/jobs/applications) | — |
| Uber | Matching, ratings | not run | — | — | — | — | Not attempted this round | **Not yet attempted** | Existing explainable matching | — |
| Microsoft Word/PowerPoint/Excel | Document deliverables | not run | — | — | — | — | Not attempted this round | **Not yet attempted** | Existing secure upload/version/review | — |

## Status legend used

REA research completed / partially completed / attempted but blocked / not yet attempted are kept
separate from "public-documentation research completed", "feature implemented", "feature tested" and
"deployed and verified". Nothing in this round is deployed: PR #122 (→ dev) awaits CI and the owner's merge.
