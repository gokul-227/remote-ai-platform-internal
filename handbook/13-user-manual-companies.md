# User Manual — Companies

A step-by-step guide to using Remote AI Platform to hire remote engineers, based on the product as it
actually works today.

## 1. Creating your account

There's no password anywhere in this product. On the sign-up page, sign up the same way an engineer
would:

1. **Google, Microsoft, or GitHub** — one click, and you're signed in immediately. Nothing further to
   set up.
2. **Email code** — enter your name and email, agree to the Terms of Service, and click **Continue**. On
   the role-selection step, choose **"I am hiring / represent an organization"** and click **Send code**.
   Remote AI Platform emails you a one-time sign-in code; enter it and click **Verify & create account**.

There's no separate "business" sign-up flow, contract, or sales process today — the same account system
serves both engineers and companies, distinguished only by the role you pick during sign-up. Every future
sign-in works the same way: enter your email, click **Send code**, and type in the code emailed to you
(or use Google/Microsoft/GitHub again, whichever you used originally).

## 2. Setting up your organization profile

After your first sign-in, the onboarding wizard walks you through three short steps:

1. **Organization Identity** — organization name (required), industry, and organization size (1-10,
   11-50, 51-200, 201-500, or 500+ employees).
2. **Organization Overview** — location/headquarters, website URL, and a description of what your
   organization builds and what technologies you use. Candidates see this, so make it count.
3. **Confirm Organization Setup** — review everything, then click **Finish Setup & Go to Hiring
   Dashboard**.

You can edit any of this later from your **Company Profile** page.

## 3. Posting a job

From your **Company Dashboard**, go to **Post a Job** (or the "+ New Job" action). Fill in:

- Job title and description
- Required skills
- Experience level
- Budget/salary range
- Remote-work preference / location constraints

Once published, your job is immediately visible in the platform-wide job search, and the AI matching
engine starts scoring it against every engineer profile on the platform in the background — you don't
need to do anything extra to trigger matching.

Manage all of your postings — editing details or pausing a role you're no longer hiring for — from the
**Company Jobs** page.

## 4. Understanding your candidate matches

Open any of your job postings and go to its **Candidates** view. Instead of an unsorted pile of resumes,
you'll see engineers ranked by the same explainable match score engineers see on their end — driven by
skills overlap, experience fit, role fit, timezone/remote-preference fit, availability, and rate fit, with
a plain-language explanation attached to each score. Use this to prioritize who to review first, not as
the only signal — always read the full profile before deciding.

## 5. Reviewing applications

Go to your **Candidates** / applications view for a job to see everyone who has applied, along with a
summary of their profile (headline, primary role, skills, years of experience, profile completeness
score, and location) inline — you don't need to click into a full profile just to triage.

Move an application through the pipeline as you review: **Reviewing → Shortlisted → Accepted** or
**Rejected**. These transitions are enforced by the system (for example, you can't jump straight from
"submitted" to "accepted" without moving through the intermediate stages), which keeps your hiring
pipeline consistent.

If you find a strong candidate who hasn't applied to your role yet, you can **invite them directly** —
this creates the same kind of application record, marked as an invitation, so it shows up for them to
accept or decline.

## 6. Messaging candidates

Once you're connected with a candidate (or they've applied to your job), message them directly from the
**Messages** page — conversations update in real time.

## 7. Managing job postings

Your **Company Jobs** page is the single place to see every role you've posted, its current status, and
how many applications it has received. From there you can edit a listing or pause/reactivate it.

## 8. Hiring: contracts and escrow payments

When you're ready to formally hire someone, this is where the product moves from "marketplace" to
"working relationship":

1. **Create a contract.** From the **Contracts** page, draft a contract for the specific person you're
   hiring: title, scope of work, rate type and amount, currency, and one or more milestones (each with a
   title and dollar amount). This sends a **contract offer** to the worker.
2. **Both parties sign.** The worker reviews and digitally signs; once you've both signed, the contract
   status becomes **Active**.
3. **Fund escrow.** From the **Payments** page, create an escrow payment tied to the contract's project/
   milestone. In production this runs through **Stripe** — funds are authorized and held, not yet
   released, until you confirm the work is done. (A sandbox/mock payment provider is also available,
   used for testing rather than real transactions.)
4. **Review delivered work.** The worker marks a milestone as delivered; you review it.
5. **Approve and release.** Once satisfied, mark the milestone **Approved** and **release the escrow** —
   funds move from escrow to the worker's platform wallet. If something goes wrong before release, you
   can instead **refund** the held escrow back to yourself.
6. Your **Payments** dashboard shows your full transaction history and current escrow exposure across all
   contracts.

**Honest caveats about this part of the product:**
- The system does **not force you to go through the application/hiring pipeline before creating a
  contract** — a contract can technically be created for anyone, at any time. In practice, use it as the
  step *after* you've accepted someone through the normal applications flow.
- There is no automatic "mark contract as fully completed and archived" step tied to milestones — a
  contract simply accumulates milestone approvals, or you can manually **terminate** it. Treat the
  contract's milestone history as your record of what's been delivered and paid.
- This is a functional but lightweight system — it is not a substitute for your own accounting, tax, or
  legal documentation for a hire.
