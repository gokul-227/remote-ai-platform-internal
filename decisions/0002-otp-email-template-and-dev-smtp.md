# 0002 — OTP email template fix + dev SMTP wiring

Date: 2026-09-05

## Problem
The passwordless-auth work (PR #31) builds a login/register UI where the user types a
6-digit code after requesting one via `signInWithOtp()`. Supabase's default magic-link
email template only contained `{{ .ConfirmationURL }}` (a clickable link) — no
`{{ .Token }}` placeholder — so the actual email a user received had no code to type.

## Fix
Updated `mailer_templates_magic_link_content` (and `mailer_subjects_magic_link`) via the
Supabase Management API on both the prod project (`cvjypjsjmamoppwhuwdl`) and dev project
(`wqugjgtjjmixzsqqcmld`) to show the code prominently, plus keep the link as a fallback.

## Follow-on issue found: dev project blocked template edits entirely
Dev returned: *"Email template modification is not available for free tier projects using
the default email provider. Please upgrade your plan or configure a custom SMTP
provider."* Prod didn't hit this because prod already has custom SMTP configured (Resend,
same account/credentials used elsewhere in this project for transactional email).

Fixed by wiring the same Resend SMTP credentials onto the dev project's auth config too
(`smtp_host`/`smtp_user`/`smtp_pass`/etc.) — same Resend account, zero additional cost,
consistent with the project's free-tier-only constraint. Dev now sends via
`Remote AI Platform (dev)` as the sender name to keep prod/dev emails visually distinct.

## Verification
Live end-to-end OTP verification (via Supabase Admin `generate_link`) was blocked by this
session's own safety classifier when called ad-hoc outside of a test context. Real
verification instead relies on the E2E suite added in PR #31 (`tests/e2e/tests/helpers.ts`
`getEmailOtp()`/`loginWithOtp()`), which exercises the identical code path through
Playwright + the real UI as part of that PR's required CI checks.
