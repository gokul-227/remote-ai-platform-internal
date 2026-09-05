# Phase 12 check: branded email addresses

2026-09-05. Verified, not fixed (nothing to fix without a domain decision).

**Finding**: no DNS verification records (`resend._domainkey.remoteaiplatform.com` TXT, domain-verification TXT, MX) exist for `remoteaiplatform.com`. Confirmed the Render-held `RESEND_API_KEY` is a send-only restricted key (can't query domain status via API, which is good security practice, not a gap). Conclusion: transactional email is currently sent via Resend's default shared sender, not a verified `@remoteaiplatform.com` address.

**To fix**: in the Resend dashboard, add `remoteaiplatform.com` as a domain, then add the DNS records Resend provides (SPF/DKIM/DMARC-style TXT records) via Cloudflare (already the DNS provider for this domain). Once verified, update the "from" address used in email-sending code to `hello@` or `no-reply@remoteaiplatform.com`. Not done in this pass since it needs a real DNS change on the live domain -- flagging as a quick, low-risk follow-up whenever convenient, not blocking anything else.
