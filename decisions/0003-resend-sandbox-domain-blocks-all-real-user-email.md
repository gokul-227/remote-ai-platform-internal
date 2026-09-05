# 0003 — Resend sandbox domain blocks email to all real users (P0)

Date: 2026-09-05

## Finding
PR #31's E2E suite fails on every journey test at the OTP login step
(`locator('#code')` never becomes visible). Root cause is NOT in PR #31's code.

Supabase's SMTP is configured with `smtp_admin_email: onboarding@resend.dev` (Resend's
default sandbox sender). Resend restricts this sender to only deliver mail to the
account owner's own registered email address until a custom domain is verified.

Confirmed directly via `POST /auth/v1/otp`:
- `admin@workmesh.ai` (arbitrary recipient) → `500 {"error_code":"unexpected_failure","msg":"Error sending magic link email"}`
- the account owner's own email → succeeds

## Impact
This is a pre-existing production bug, not something PR #31 introduced. It just wasn't
caught before because every prior auth flow either bypassed real email sending (admin
API force-confirm) or didn't require a successful send to proceed. PR #31 is the first
flow where a real Supabase-triggered email send is mandatory on every login, so it's the
first thing to surface this.

**Right now, no real user other than the account owner can receive any transactional
email from the platform** — not an OTP code, not a registration confirmation, nothing.

## Fix (requires manual dashboard steps — blocked here by the same persistent Cloudflare
API/IP restriction noted in decision 0002/earlier email-routing work)
1. Add `remoteaiplatform.com` as a domain in the Resend dashboard
2. Add the DNS records Resend provides (SPF/DKIM/DMARC) via the Cloudflare dashboard
3. Once Resend shows the domain verified, update `smtp_admin_email` on both Supabase
   projects (prod `cvjypjsjmamoppwhuwdl`, dev `wqugjgtjjmixzsqqcmld`) from
   `onboarding@resend.dev` to a `@remoteaiplatform.com` address (e.g. `hello@`)

## Status
Blocked on user completing the Resend domain verification + DNS records. PR #31's code
itself is correct and fully verified otherwise (backend/frontend checks all green); its
E2E failure will resolve once this is fixed, no PR changes needed.
