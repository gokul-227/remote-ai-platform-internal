# Launch contract (Phase 02)

Drafted 2026-09-27 from source at `07867833` + Phase 01/02 branches. Items marked **Proposed** need the owner's
confirmation; everything else describes how the product behaves in code today.

## Who it is for, and the core loop

| Persona | Role (internal) | Core journey at launch |
|---|---|---|
| Visitor | — | Browse/search/filter remote jobs → read a job → go to the source site (imported) or sign in to apply (direct) |
| Professional | `ENGINEER` | Sign in (email code / Google / Microsoft / GitHub) → profile (manual or resume import, reviewed before saving) → visibility → save / apply / withdraw → see application status and invitations |
| Organisation | `COMPANY` | Sign in → organisation profile → post / edit / close a job → review applicants and AI match → invite / shortlist / decline → message |
| Admin | `ADMIN` | Moderate reports, verify, hide jobs, run job sync, view AI usage and audit |

**Imported vs direct jobs.** Almost all live jobs (100/100 in the 2026-09-27 sample) are imported from Arbeitnow,
RemoteOK, Remotive and The Muse. Imported jobs are applied to **on the source site** — the API now refuses in-platform
applications to them (409), and the UI opens the source link or says there is none. Only jobs an organisation posts
here accept in-platform applications, and only while open (`e9fdc44`).

## Account model (current)

- One sign-in identity ↔ one app user. Business role lives in the app `users` table, never in the identity token.
- A user can switch between professional and organisation at any time (`PATCH /auth/role`, audited). Authorization
  follows the **current** role.
- One user owns at most one organisation profile (`company_profiles.user_id` unique). **There are no multi-member
  organisations**: no teammates, no org roles. Anything described as "team" must not be implied in copy.
- **Proposed:** keep this for launch; add organisation membership (owner/admin/member) only when a customer needs it (ENT-01).

## Surface visibility at launch

| Surface | Launch | Why |
|---|---|---|
| Jobs, search, job detail, saved, applications, matches | **Visible** | Core loop; covered by E2E |
| Profiles, resume import (AI with manual fallback), visibility | **Visible** | Core loop |
| Organisation profile, post job, candidates | **Visible** | Core loop |
| Network, messages, notifications | **Visible** | Needed to act on applications/invitations |
| Feed, groups | **Visible — Proposed** | Implemented with report/moderation; could be hidden if moderation capacity is a concern |
| Contracts + milestones (no money) | **Visible — Proposed** | Lifecycle enforced server-side; no payment states implied |
| Projects board, task marketplace, work log, quality review, "Hire a professional" | **Visible — Proposed**, or hide behind a flag if the owner prefers a narrower beta | Real APIs, but broad surface to support |
| Payments, escrow, earnings, payouts | **Gated off** (`MARKETPLACE_PAYMENTS_ENABLED=false`, 503 server-side, UI says "Payments aren't live yet") | PAY-01; re-entry gate = Phase 13 + owner/legal approval |
| Subscriptions (Pro plans) | **Not built** (entitlements model only) | Owner's final phase; Phase 13 |
| Disputes, payout methods, MFA/device list, open task bidding | **Not offered**, not shown | Known gaps (progress.md) |

## Honesty rules (enforced by tests)

- No escrow / held-funds language, no personal-picks wording on unpersonalised lists, no "NK+" statistics, no SOC 2/ISO
  claims, no guaranteed outcomes — `apps/web/src/figma/claims.test.ts`.
- Legal pages do not describe payments as processed or escrowed — `legal.test.tsx`.
- Match-score explanation in Help equals the weights in `matching/service.py` (40/25/15/8/7/3/2).

## Language, region, support, accessibility

- English only. Operator in the EU (Germany); jobs are global remote roles — the job's own location/eligibility text is
  authoritative.
- **Proposed support capacity:** one owner, email `contact@remoteaiplatform.com`, reply target 2 business days, abuse
  reports reviewed within 1 business day. Mailbox delivery still unverified (OPS-02).
- Accessibility target: WCAG 2.2 AA on primary flows (axe scans in E2E; manual pass in Phase 05).

## Decisions needed from the owner

1. Confirm the three **Proposed** visibility rows (community, contracts, projects/marketplace) or pick which to hide.
2. Confirm the support targets above.
3. Confirm "no multi-member organisations at launch".
