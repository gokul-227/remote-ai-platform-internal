# Required from you — step by step

Updated 2026-09-05 (evening).

## 🔴 P0 — blocks real users right now, found while debugging PR #31's failing CI
**No real user except you can receive any email from the platform.** Resend is still on
its sandbox sender (`onboarding@resend.dev`), which only delivers to the account owner's
own inbox until a custom domain is verified. Confirmed directly: OTP send to a test
account → 500 error; OTP send to your own email → works. Full writeup: `decisions/0003-...md`.

**Fix (needs you, dashboard-only, I'm blocked by the same Cloudflare API/IP issue as the email-routing task):**
1. Resend dashboard → add `remoteaiplatform.com` as a domain
2. Add the DNS records Resend gives you (SPF/DKIM/DMARC) via the Cloudflare dashboard
3. Tell me once verified — I'll switch Supabase's sender email on both projects

This is also *why* PR #31's E2E suite fails — not a bug in that PR's code (see below).

## Done
- ✅ All 12 phase PRs merged
- ✅ Sentry PR **#29** merged — both DSNs wired (Render env vars backend, GitHub secret + deploy workflow frontend)
- ✅ Dev storage isolation: dev's own Supabase S3 keys set on Render
- ✅ Legal content PR **#30** — real drafted Privacy/Terms/Impressum content, "AI-drafted, pending lawyer review" banner kept
- ✅ Supabase OTP email template fixed on **both** prod and dev projects — the magic-link/OTP email now shows a typeable 6-digit code (`{{ .Token }}`), not just a click-link. Dev project also needed custom SMTP wired (Resend, same account as prod) since Supabase free-tier projects block template edits on the default mailer — done.
- ⏭️ Upstash Redis for dev: skipped — free tier caps at 1 DB/account, upgrading needs a payment method (against the no-spend rule). Dev keeps sharing prod's Redis, low-risk per decision 0001.

## Still needed: merge these PRs (in order)
1. **#28** — `dev → prod` promotion (ships everything live, closes remaining Dependabot alerts) — still open
2. **#31** — https://github.com/gokul-227/remote-ai-platform/pull/31 — passwordless email OTP (replaces password login entirely) + GitHub OAuth login. Backend/frontend fully verified (ruff/mypy/pytest 249/249, tsc/lint/vitest/build all green). **E2E is currently failing — root cause is the Resend P0 above, not this PR's code.** It'll pass on its own once that's fixed (just re-run the job, no code change needed). Also still needs the GitHub OAuth App steps below before merging.
3. **#32** — https://github.com/gokul-227/remote-ai-platform/pull/32 — Facebook-style blue/white rebrand. Uses `#0552CC` light / `#4C9AFF` dark rather than Facebook's literal `#1877F2` — that exact hex only scores 4.23-4.24:1 contrast on white, under the 4.5:1 AA bar, so it was darkened slightly to stay accessible while keeping the same hue. Zero new axe-core violations, lint/tsc/vitest/build all clean, screenshots confirm light+dark mode both look coherent. Fully independent of #31, can merge in either order.

## Action needed from you before merging #31

**1. Create the GitHub OAuth App** (I can't do this — needs your GitHub account):
   - Go to https://github.com/settings/developers → "New OAuth App"
   - Application name: `Remote AI Platform`
   - Homepage URL: `https://remoteaiplatform.com`
   - Authorization callback URL: `https://cvjypjsjmamoppwhuwdl.supabase.co/auth/v1/callback`
   - Create it, copy the **Client ID**, generate and copy a **Client Secret**
   - Repeat for a second OAuth App using the dev project's callback URL if you want GitHub login to work on dev too: `https://wqugjgtjjmixzsqqcmld.supabase.co/auth/v1/callback`

**2. Enable the GitHub provider in Supabase** (needs the Client ID/Secret from step 1 — tell me the values and I'll wire them in via the Management API, same as I did for Google/Microsoft):
   - Prod project (`cvjypjsjmamoppwhuwdl`) and dev project (`wqugjgtjjmixzsqqcmld`)

Once those two are done, #31 is ready to merge — no password login will exist anymore, only email OTP + Google/Microsoft/GitHub.

## Decisions pending your answer (just reply, no research needed)
1. **Auth system consolidation** (Supabase-native + legacy custom-JWT `service.py` path) — "do it" or "later"? Lower urgency now that the dead custom-OAuth code (`oauth.py`) was already found unused and removed.
2. **Accessibility contrast colors** (2 design tokens, ~180 usages, needs a brand-color call) — "darken them" or "leave as-is"? May get superseded by the Facebook-blue rebrand once that lands.
3. **Legal content identity fields** — Impressum needs your real name/address (or wait until a company is formed) before the lawyer review pass.
4. **Pricing/plans** — architecture built (billing PR), zero real prices set. Whenever you're ready to define tiers/limits/prices.
5. **Cloudflare Email Routing** (business inboxes hello@/support@/security@/privacy@ → your Gmail) — still blocked on my end by a persistent Cloudflare IP restriction on this sandbox. Manual dashboard steps already given earlier — let me know once it's set up, or ask again for the steps.
