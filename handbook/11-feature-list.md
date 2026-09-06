# Feature List

*Built from the actual frontend routes (`apps/web/src/app/**/page.tsx`) and backend domains
(`apps/api/app/domains/*`) as they exist today — not from old planning documents. Anything not fully
working is called out explicitly rather than presented as finished.*

Legend: **Working** = real, end-to-end, usable today. **Partial** = built but with a real gap or rough
edge. **Placeholder** = visible in the UI but not backed by real functionality yet.

---

## Engineer features

| Feature | What it does | Status |
|---|---|---|
| Passwordless sign up / sign in | Create an account and sign in with a one-time email code — no password to set or remember — or with one click via Google, Microsoft, or GitHub. See the User Manual for the exact flow. | Working |
| AI resume import | Upload a PDF/DOCX resume during onboarding or from your profile; AI extracts your headline, primary role, skills, experience, and location so you don't retype your resume. | Working |
| Manual profile builder | Fill in headline, role, bio, skills, experience, education, projects, rate, and links (GitHub/LinkedIn/portfolio) by hand instead of, or in addition to, AI import. | Working |
| Profile completeness score | A visible 0–100 score and checklist (bio, 3+ skills, experience, a project, resume, rate) nudging you to finish your profile — a fuller profile matches better. | Working |
| Job search & filters | Search and filter open remote roles by keyword, location, job type, experience level, remote-only, and salary range. | Working |
| Explainable AI match score | Every job you view shows a computed match score (skills 40%, experience 25%, role fit 15%, timezone 8%, availability 7%, rate 3%, remote-preference 2%) with a plain-language rationale — not a black box. | Working |
| Save jobs | Bookmark jobs to review later without applying immediately. | Working |
| Apply to jobs | Submit an application (with an optional cover note) to any open role; track its status as it moves through Submitted → Reviewing → Shortlisted → Accepted/Rejected. | Working |
| Withdraw an application | Pull back an application you've submitted, as long as a company hasn't already accepted or rejected it. | Working |
| Direct invites from companies | A company can invite you to apply directly to a role they think you fit — it shows up in your applications list as an invitation. | Working |
| Personal recommendations dashboard | A dedicated page surfacing your top AI-ranked job matches. | Working |
| Direct messaging | One-to-one conversations with companies (or other members) once you're connected — real-time via WebSocket. | Working |
| Professional network / connections | Send, accept, and manage connection requests with other engineers and companies, similar to a professional network. | Working |
| Social feed | A LinkedIn-style feed: post updates/project news/achievements/articles, like, comment, and share; filter by "All," "Connections," or "Media & Links." | Working |
| Groups | Join or create interest-based groups, post inside them, and manage membership roles. | Working |
| Notifications | In-app notifications for application status changes, contract offers, messages, and escrow/payment events, plus a live unread-count badge. | Working |
| Global search | Search across jobs, companies, and people from one search bar. | Working |
| Digital contracts | Once a company decides to hire you, review contract terms (rate, scope, milestones) and sign digitally. | Working |
| Escrow-backed payments | See funds held in escrow for your contract milestones, and get paid out to your platform wallet once a company approves and releases a milestone. | Working |
| Wallet & transaction history | A running view of escrow held, total earned, and full transaction history. | Working |
| "Trending Skills" panel (feed and sidebar) | A widget intended to show which skills are rising in demand across the platform. | **Coming soon (honestly labeled)** — the backend job that would compute this (`refresh_trending_skills`) is a documented no-op stub with no real logic behind it yet, gated off by a `FEATURE_TRENDING_SKILLS` flag. Rather than showing fabricated numbers, the UI displays a plain "Coming soon — we're still building this" message in its place — an intentional, transparent placeholder, not a bug. |
| Code/submission quality review | An AI endpoint exists to evaluate code submissions and give a quality report. | **Partial** — the backend (`/quality/evaluate`, `/quality/review-code`) is implemented, but there's no dedicated engineer-facing page for it in the current frontend routes; it's not part of the everyday engineer workflow today. |

## Company features

| Feature | What it does | Status |
|---|---|---|
| Company sign up / sign in | Same passwordless email-code auth (plus Google/Microsoft/GitHub one-click sign-in) as engineers — see the Company Manual. | Working |
| Company profile | Set up organization name, industry, size, location, website, and description so candidates know who they're applying to. | Working |
| Post a job | Publish a new open role with title, description, required skills, experience level, budget range, and remote preference. | Working |
| Manage job postings | Edit, pause/reactivate, or view all of your organization's postings in one dashboard. | Working |
| Ranked, explainable candidates | For any job you post, see engineers ranked by the same explainable match engine engineers see, so you're not scanning an unfiltered pile. | Working |
| Review applications | See everyone who applied to your jobs, with their profile summary (headline, skills, experience, profile score) surfaced inline. | Working |
| Move applications through a pipeline | Change an application's status (Reviewing → Shortlisted → Accepted/Rejected) with real state-transition rules enforced server-side. | Working |
| Invite candidates directly | Proactively invite a specific engineer to apply to one of your open roles. | Working |
| Message candidates | Direct, real-time messaging with any candidate you're connected to. | Working |
| Browse the engineer/company directories | Explore public engineer and company profile listings. | Working |
| Digital contracts | Draft a contract for a specific worker (rate type, amount, scope, milestones), send it, and track it through Offered → Signed → Active → Completed/Terminated. | Working |
| Escrow-backed payments | Fund an escrow payment (via Stripe, or a sandbox provider for testing) for a project/task, then release or refund it once milestones are approved. | Working |
| Payments dashboard | Wallet balance, escrow held, and full transaction history across all your contracts. | Working |
| Company/network feed & groups | Same social feed, groups, and connections features as engineers. | Working |

## Admin features

*(covered step-by-step in the Admin Manual — summarized here for completeness)*

| Feature | What it does | Status |
|---|---|---|
| Platform stats dashboard | Live counts of registered engineers, companies, jobs, and total users. | Working |
| System health monitor | Live up/down status for core services (auth, jobs API, database, Redis, etc.) with latency. | Working |
| AI usage & cost monitoring | Aggregated LLM call counts, token usage (prompt/completion), estimated cost in USD, and a breakdown by feature. | Working |
| Job-source sync monitoring | Per-source (RemoteOK, Arbeitnow, Remotive, USAJobs, The Muse) status of the last automated import run, with success/failure and job counts. | Working |
| User management | List, search, suspend/reactivate, change the role of, or permanently delete any user account. | Working |
| Job management | Pause/reactivate or permanently delete any job posting; bulk re-clean legacy job text formatting. | Working |
| Moderation queue | Review and act on user-submitted reports (hide a job, suspend a user, or dismiss) from one queue. | Working |
| Audit log | An immutable, searchable log of every sensitive admin action (status changes, role changes, deletions) for accountability. | Working |
| Activity log | A general feed of platform administrative activity. | Working |
| Feature-flag visibility | The backend can report which platform feature flags are currently on/off (e.g. trending skills, AI matching). | **Partial** — the API endpoint exists and works, but there's no panel for it in the admin dashboard UI yet, and flags can only be changed via server configuration, not clicked on/off from the console. |

## Shared / cross-cutting features

| Feature | What it does | Status |
|---|---|---|
| Notifications | Real-time in-app notifications (via WebSocket) for messages, application updates, contract events, and payments, for every persona. | Working |
| Messaging | One-to-one real-time conversations, available to engineers and companies alike once connected. | Working |
| Professional network | Connections, a public engineer directory, and a public company directory. | Working |
| Global search | One search surface across jobs, people, and companies. | Working |
| Groups | Community spaces any user can create, join, and post in. | Working |
| Social feed | The shared LinkedIn-style feed described above, available to both personas. | Working |
| Saved items | Bookmarking for jobs today (engineers); no equivalent for companies bookmarking candidates yet. | Partial |

---

## A note on "coming soon" honesty

The product has a good example of doing this right: the **Trending Skills** widget (on the social feed
and in the right-hand sidebar across several pages) doesn't exist yet as real functionality — the backend
task that would compute it, `refresh_trending_skills`, is a documented no-op stub with no logic behind
it — but rather than faking numbers to look finished, the UI plainly shows "Coming soon — we're still
building this" in that panel's place. It's gated behind a `FEATURE_TRENDING_SKILLS` flag so the real
feature can be switched on deliberately once it's built, instead of a half-working version going live by
accident. This is the standard worth holding every other in-progress feature to.
