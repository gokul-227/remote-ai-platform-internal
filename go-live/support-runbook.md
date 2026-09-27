# Support and administration runbook

Channel: `contact@remoteaiplatform.com` (SUPPORT_EMAIL). Delivery to a real inbox is still unverified (OPS-02).
Proposed targets: reply within 2 business days; abuse reports and appeals within 1 business day.

| Request | What to do | Tool |
|---|---|---|
| "Delete my account" with signed contracts/payments | Self-service answers 409 by design. Suspend the account, delete profile content and resume by hand, keep contract/payment rows until the retention policy says otherwise; confirm in writing | Admin → Users (suspend) |
| Data export | Self-service: Settings → Privacy → Download my data | — |
| Correct data | User edits their profile; otherwise admin edits via Users | Admin |
| Moderation appeal | Read the report and decision (never reveal the reporter). Reverse if wrong: re-activate user / un-hide job; removed posts can't be restored (tell the user) | Admin → Reports |
| Suspended user asks why | The API tells them they are suspended and where to appeal; answer with the decision category, not the reporter | Admin → Reports |
| Account-merge / "different sign-in method" (409) | Verify the person controls both identities (reply from both addresses); then link manually in the DB (set `users.auth_subject`); log it | SQL, audited note |
| Blocked by someone | Blocks are private; say only that they can't contact that member | — |
| Job sync stale (canary) | Run scheduled-job-sync manually; check admin sync log | GitHub Actions |

Rules: least data access; no impersonation; every sensitive action leaves actor/reason/time (admin actions are logged
automatically; for manual SQL add a note here). Organisation "verified" badges can only be set in the database today —
don't grant any until the badge's meaning is written down.
