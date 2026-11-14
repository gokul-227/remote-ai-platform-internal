# LinkedIn — professional identity and recruitment (research package, 14 Nov 2026)

Evidence labels: **[Observed]** read-only accessibility snapshot of the owner's signed-in Chrome tab via the chrome-devtools connector; **[Documented]** official LinkedIn Help; **[Inferred]** reasonable inference; **[Unknown]** internal detail not observed.

## Product overview
Professional network combining identity (profile), network (connections/follows), content (feed), jobs (search, apply, tracker, recruiter tools) and messaging. Observed primary navigation **[Observed]**: Home, Network, Jobs, Messaging, Notifications, Me, For Business, Learning. Global search box accepts natural-language queries ("Describe the job you want").

## Feature inventory (observed + documented)
- Profile card with headline, location, Premium badge, "Experience" quick-add, profile viewers count, analytics link **[Observed]**.
- Feed: "Start a post" with Video/Photo/Write article, feed ordering control ("Most relevant first"), per-post Follow, control menu, Hide post, reactions menu, Comment, Repost, Send, reaction counts, visibility icon ("Visibility: Global"), promoted posts, comment threads with Reply counts and "See more comments" **[Observed]**.
- Jobs home: "Jobs based on your preferences" (stated preference sentence + edit), "Jobs where you're more likely to hear back", "Recent job searches", "More jobs for you"; each card shows title, company, location + remote/hybrid/on-site, salary range when present, badges (Verified job, Promoted, Easy Apply, Viewed, Actively reviewing applicants, Be an early applicant, "N connections work here"), per-card Dismiss and More options **[Observed]**.
- Jobs sidebar: Preferences, Job tracker, My Career Insights, Post a job, Manage job posts **[Observed]**.
- Apply: Easy Apply (in-platform multi-step form → Review → Submit) vs Apply (redirect to employer site); submitted applications cannot be edited or withdrawn **[Documented: help a512388]**.
- Messaging overlay: conversation list, per-conversation thread, quick-reply suggestions, attach image/file, GIF, emoji, send options **[Observed]**.
- Saved items, Groups, Newsletters, Events in sidebar **[Observed]**.
- Recruiter pipelines/notes/tags: help page returned 404 anonymously; **[Unknown]** beyond public marketing.

## Screens and navigation
Top bar (7 primary items) + context sidebars. Jobs is a hub page with multiple ranked lists rather than a single search results page. Notification badges are counted per nav item ("Home, 1 new notification").

## Detailed workflows
1. Job discovery → card → detail (search-results URL with `currentJobId`) → Easy Apply / Apply → tracker. **[Observed+Documented]**
2. Preference-driven recommendations: user states job titles, remote scope, locations; the platform shows the sentence and lets the user edit it. **[Observed]**
3. Messaging from recruiter to candidate with templated quick replies ("Yes, interested…", "No thanks…"). **[Observed]**
4. Post creation with media/article, audience visibility shown on each post. **[Observed]**

## User roles and permissions **[Inferred from observed surfaces]**
Member, Premium member, job poster/recruiter (Post a job, Manage job posts), company page admin. Application records are private to applicant and poster.

## Data model hypotheses **[Inferred]**
Member, Profile(sections), Connection, Follow, Post, Reaction, Comment, Job(verified flag, apply mode, salary range, workplace type), Application(status, tracker stage), SavedJob, JobPreference, Conversation, Message, Notification.

## Observed interaction patterns
- Per-card dismiss with reason-free undo; "Viewed" state persisted on cards.
- Badges communicate hiring activity ("Actively reviewing applicants") and timing ("Be an early applicant").
- Quick replies lower friction in recruiter conversations.

## Official documentation
- Applying for jobs: https://www.linkedin.com/help/linkedin/answer/a512388
- Jobs you may be interested in: https://www.linkedin.com/help/linkedin/answer/a512279

## Observation record
Chrome tab "Jobs | LinkedIn" and "Feed | LinkedIn", 14 Nov 2026 ~02:10 CET, accessibility snapshot only. Personal names, message bodies and job identifiers seen in the snapshot are not retained in this repository.

## Implementation opportunities for Remote AI Platform
- Add a **Job tracker** view (saved → applied → interviewing → offer/closed) on top of `job_applications` + `saved_jobs`.
- Add stated **job preferences** record used by matching and shown as an editable sentence on the jobs home.
- Add job card badges from real data: "Easy apply" (internal) vs "Apply on source", "Viewed", "Early applicant" (application count < N), "Actively reviewing" (company activity within 7 days).
- Recruiter quick replies/templates in channels for application conversations.
- Organisation membership and roles (poster vs admin) — missing today.

## Acceptance criteria (Remote AI Platform)
- A professional completes onboarding, sets preferences, sees preference-driven recommendations, saves and applies; the tracker reflects each stage server-side.
- An organisation member with the recruiter role can publish/close jobs and move applicants through stages; a plain member cannot (403), and competing applicants cannot read each other's applications.
- Hidden profiles never appear in search, recommendations or public URLs.
