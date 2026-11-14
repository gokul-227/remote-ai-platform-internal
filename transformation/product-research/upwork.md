# Upwork — freelancing and contracts (research package, 14 Nov 2026)

Evidence labels: **[Observed]** signed-in Chrome tab snapshot (owner's account, read-only); **[Documented]** Upwork Help Center (several help URLs returned 403 to anonymous fetch; summaries below come from search-result abstracts of help articles 211063268, 360023544173, 211063618 and public marketing pages); **[Inferred]**; **[Unknown]**.

## Product overview
Two-sided marketplace: clients post jobs, talent submits proposals, parties contract on hourly or fixed-price terms with milestones, work is submitted and approved. Payments/escrow are core to Upwork and explicitly **out of scope** for Remote AI Platform.

## Feature inventory
- Talent home ("Find work"): tabs Best matches / Most recent / Saved jobs / Invites (count); Filters; job cards with posted time, proposals bucket ("Fewer than 5"), hourly/fixed, experience level, estimated duration and hours/week, description, skill chips, client payment-verified flag, client rating, client spend, client country; per-card Save and Job feedback; "Apply now" **[Observed]**.
- Primary nav for talent: Home, Messages, Opportunities, Contracts, Finances, notifications **[Observed]**.
- Profile sidebar: visibility (Public), completeness %, Work Now / Open for work / Boost toggles, Connects balance, Preferences, Proposals, Project Catalog, Direct Contracts **[Observed]**.
- Client marketing: "Project workspace — messages, files, and status in one view", team workspace with roles and permissions (Business Plus), curated shortlists **[Observed, public page]**.
- Proposal review (client): message, shortlist, hire, decline, archive **[Documented]**.
- Offers: client may withdraw/change until accepted; talent has 7 days to respond; acceptance creates a contract; proposed contract can't be edited but can be withdrawn and resubmitted **[Documented]**.
- Milestones: planned upfront or added later; one active/funded at a time; milestone change requests can be proposed and withdrawn before acceptance; changes to submitted work are requested in the contract workroom **[Documented]**.

## Screens and navigation
Feed-style job list with saved/invites tabs; contract "workroom" per contract (messages + activity timeline + milestones) **[Documented/Inferred]**.

## Detailed workflows
1. Discover → Save → Apply (proposal with cover letter, terms, optional milestones) → client review (shortlist/decline/archive/message) → interview via messages → offer → accept → contract → milestone submit → approve or request changes → complete.
2. Invitation flow: client invites talent (Invites tab) → talent accepts/declines → proposal.

## Roles and permissions **[Inferred]**
Client (owner), client team member (Business Plus roles), talent, agency member. Only contract parties see the workroom.

## Data model hypotheses **[Inferred]**
JobPost(type, budget, duration, experience, skills, client stats), Proposal(status: submitted/shortlisted/declined/archived/withdrawn, terms), Invite, Offer(expires_at), Contract(type, status), Milestone(status: pending/active/submitted/changes_requested/approved), Submission, ChangeRequest, Review, Message room.

## Observed interaction patterns
- Transparent competition signal ("Proposals: Fewer than 5").
- Client trust signals on every card (payment verified, rating, spend, country).
- Preference and availability toggles live beside the feed.

## Official documentation
- Review proposals: https://support.upwork.com/hc/en-us/articles/211063268
- Propose changes to a milestone: https://support.upwork.com/hc/en-us/articles/360023544173
- Add or edit milestones: https://support.upwork.com/hc/en-us/articles/360000990268

## Observation record
Chrome tab "Upwork" (find-work/best-matches), 14 Nov 2026. Client names, job texts and the owner's tax banner are not retained.

## Implementation opportunities
- Add "proposals so far" bucket, client trust signals (verified organisation, completed contracts count, average review) to freelance brief cards from real data.
- Add Saved briefs and Invites tabs to the freelance feed (invitations exist in offers; expose as a tab).
- Contract workroom = unified project workspace (Phase 9) with activity timeline.
- Offer expiry (7 days) and withdraw-before-acceptance rule already partly exist; verify and document.

## Acceptance criteria
- Full lifecycle via API with server-side state rules; invalid transitions 409; outsider 404/403.
- Milestone change requests: propose/withdraw/accept/reject with one active milestone enforced.
- No payment, escrow or money-release language anywhere in the UI.
