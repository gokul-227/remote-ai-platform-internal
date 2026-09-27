# Product Overview & Business Architecture

*Last verified against the live codebase: 2026-09-27.*

## What this is

Remote AI Platform is a marketplace that connects **remote professionals of every field** with **companies and
organisations hiring remotely**, with artificial intelligence doing two specific jobs in the middle:

1. **Reading resumes.** When an engineer uploads a resume, an AI model extracts their skills, work
   history, and headline automatically, instead of asking them to type it all in by hand.
2. **Explaining matches.** Instead of a black-box "you're 87% compatible" score, the platform computes a
   match between an engineer and a job from real, visible factors (skills overlap, seniority fit, role
   fit, timezone, remote-work preference, and rate) and shows *why* a match scored the way it did.

Around that AI core sits a fairly complete professional-network product: company and engineer profiles,
job search and applications, a LinkedIn-style social feed, direct messaging, groups, saved jobs,
notifications, and — for the hiring side — digital contracts with milestones once someone is
actually hired.

## Who it's for

**Engineers ("Professionals")** come to the platform to build a profile once (with AI doing the heavy
lifting from their resume), discover remote roles that are genuinely relevant to them, understand at a
glance why a role is or isn't a good fit, apply, and manage the process — messaging, tracking application
status, and eventually signing a contract — in one place. Payment is made directly between the parties.

**Companies** come to the platform to post open remote roles, see qualified candidates surfaced and
ranked automatically rather than sifting through hundreds of unfiltered resumes, review and manage
applications, message candidates directly, and — for roles that convert to a hire — issue a contract with
milestones and pay the professional directly (for example by bank transfer).

**Admins** (the platform operator, today effectively the founder) come to the platform to keep the lights
on: see how many engineers/companies/jobs exist, watch the automated job-import pipeline for failures,
keep an eye on what the AI features are costing in LLM tokens, manage user accounts, and act on anything
flagged for moderation.

## The three personas, in one sentence each

| Persona | What they come to do |
|---|---|
| **Engineer** | Get discovered and matched for real remote roles with minimal manual busywork. |
| **Company** | Post a role once and get a ranked, explainable shortlist instead of a resume pile. |
| **Admin** | Keep the marketplace, the AI pipeline, and the job-import pipeline healthy. |

## End-to-end business flow

This diagram shows the actual, current mechanics of how a job moves from being posted to a completed,
paid engagement — not an idealized future state. Where a step exists in the code but isn't fully wired
end-to-end for a normal user, that's called out directly underneath.

```mermaid
flowchart TD
    A[Company posts a job\nvia Post a Job form] --> B[Job stored as JobPost]
    A2[Automated aggregators pull public\nremote listings every 6h:\nRemoteOK, Arbeitnow, Remotive,\nUSAJobs, The Muse] --> B

    B --> C[AI Job Enricher\nnormalizes skills / seniority\nfor aggregated listings]
    C --> D[Matching Engine scores every\nEngineer profile against the job:\nskills, experience, role, timezone,\nremote fit, rate — with a plain-language\nrationale for the score]

    E[Engineer signs up,\nuploads resume] --> F[AI Resume Parser extracts\nskills / experience / headline]
    F --> G[Engineer profile]
    G --> D

    D --> H[Engineer sees ranked,\nexplainable job matches\non their dashboard]
    D --> I[Company sees ranked,\nexplainable candidates\nfor their job]

    H --> J[Engineer applies,\nor Company invites the engineer directly]
    J --> K[Application: SUBMITTED to REVIEWING\nto SHORTLISTED/ACCEPTED or REJECTED]
    I --> J

    K -->|Company decides to hire| L[Company issues a Contract\nwith rate + milestones]
    L --> M[Both parties digitally sign\ncontract goes ACTIVE]
    M --> O[Engineer delivers a milestone]
    O --> P[Company approves the milestone]
    P --> Q[Company pays the professional directly\noutside the platform]

    K -.->|messaging available at every stage| R[Direct messages\nbetween engineer and company]
```

**What's real vs. what's aspirational in this flow, honestly:**

- Job posting, AI enrichment for aggregated jobs, resume parsing, the explainable match score,
  applications (submit / invite / status transitions), messaging, contracts, digital signing and the
  enforced milestone lifecycle are implemented end-to-end. Money movement is switched off
  (`MARKETPLACE_PAYMENTS_ENABLED=false`).
- The "Company invites an engineer directly" path re-uses the same application record as a normal
  apply — there's no separate outreach/sourcing product yet.
- Contracts are a working feature set, but they are **not gated on an
  application actually reaching ACCEPTED** — a company can create a contract for any user it knows the ID
  of. In practice today it's used as the natural next step after a hire decision, but the system doesn't
  enforce that sequencing.
- There is no in-product concept of "project completion" closing out a contract automatically — a
  contract moves through milestone approvals, but marking a contract COMPLETED vs. leaving it ACTIVE
  indefinitely is a manual/informal step today.

## Business model — the honest current state

**Everything on the platform is free to use today.** There is no active billing system, no subscription/
plan model, and no paywall anywhere in the product — every feature described in this handbook (AI
matching, unlimited job postings, unlimited applications, messaging, contracts) is available to
every engineer and company account at no cost.

The codebase does now contain the first piece of *scaffolding* for future monetization — a `billing`
domain with `Plan` and `Subscription` data models and an entitlement-checking helper — but it is
explicitly architecture-only: no plan names, prices, or usage limits have been decided, the helper isn't
called from a single existing endpoint, and there's no billing API mounted for the frontend to talk to at
all. Nothing about today's product experience is gated by it.

This is a deliberate, temporary decision, not an oversight: the plan is to prove out usage and value
first, then introduce paid plans and feature gating later, with the groundwork above ready for that switch
to be flipped. The natural places to meter are visible in the product already — job-posting volume for
companies, advanced search/matching depth, and priority placement are the likely candidates — but none of
that is decided or wired up today, and this document should not be read as describing a finished pricing
model. Anyone evaluating the business today should treat "free for everyone, monetization is scaffolded
but not yet active" as the accurate, current answer.

**Decided for the final phase (owner, 2026-09):** freelance/contract work is paid **bank-to-bank between
the parties**; the platform never holds that money. The platform's own revenue comes from Stripe Checkout
plans: **Professional Pro €9/month** and **Organisation Pro €29/month**. Free accounts keep up to 2 job
posts, 50 applications a month and 2 freelance posts; Pro lifts these and includes an AI token allowance.
Until that ships, everything stays free and the escrow code is switched off.
