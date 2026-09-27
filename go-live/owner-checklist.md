# Owner checklist

Things only the owner can do, or has chosen to do later. Last updated 2026-09-27.

## Now / at launch

- [ ] **contact@remoteaiplatform.com delivery.** Cloudflare rule "contact@ forward" exists and the
      destination is verified. Send one email from another account to contact@remoteaiplatform.com. If it
      doesn't arrive: Cloudflare dashboard → remoteaiplatform.com → Email → Email Routing → **Enable**
      (adds the MX/SPF records). A Resend test to the address was suppressed, so check with a real mailbox.
- [x] Supabase prod auth: email rate limit, redirect URL allow-list without localhost (done by owner).
- [x] Groq key in Render prod + dev and in Infisical.

## After launch

- [ ] **Branch protection:** restore `enforce_admins` on `prod` and `dev`, keep required CI checks, 0
      approvals (solo maintainer). Snapshots: `protection-prod.json`, `protection-dev.json`.
- [ ] **Rotate every key pasted into the chat session** (Cloudflare, Stripe live, GitHub, Resend, Render,
      Infisical, Figma, Groq, Supabase access token). Update Infisical first, then Render / GitHub secrets.
- [ ] Optional: free keys for Gemini, Cerebras, OpenRouter, Mistral → Infisical + Render. The chain picks up
      each provider automatically once its key is set (`AI_PROVIDER=auto`).
- [ ] Optional: restore the paused dev Supabase project (`wqugjgtjjmixzsqqcmld`) if the dev site should
      allow sign-in.
- [ ] Legal review of Terms/Privacy.

## Final phase (deferred by owner)

- [ ] **Impressum identity:** name, address, VAT/tax status in `apps/web/src/figma/legal_content.ts`
      `OPERATOR` (shows "Coming soon" until then).
- [ ] **Stripe plans:** Professional Pro €9 / Organisation Pro €29 per month via Stripe Checkout (individual
      account, German tax ID; check Kleinunternehmer/VAT treatment with a tax adviser before charging).
      Free limits: 2 job posts, 50 applications/month, 2 freelance posts; AI allowance for Pro.
- [ ] **Freelance payments:** bank-to-bank between the parties; payout details visible only to the contract
      counterparty; no platform or owner bank details anywhere.
