# 08 — Google Contacts invite (owner runbook)

Feature: "Find people you know from Google" on My network. It ships switched off behind the
feature flag `contacts_import` (`FEATURE_CONTACTS_IMPORT=false`). Leave it off in production until
Google has verified the OAuth app for the contacts scopes.

## What it does (for the verification form and the privacy review)

- The member clicks the button. The API builds a Google consent URL with
  `https://www.googleapis.com/auth/contacts.other.readonly` and
  `https://www.googleapis.com/auth/contacts.readonly`, `access_type=online` (no refresh token),
  `include_granted_scopes=true`, PKCE S256 and a signed state tied to the member (10 min).
- Google sends the browser back to `https://<web origin>/contacts/callback`. That page posts the
  code to `POST /api/v1/contacts/google/import`. The API exchanges it server-side, reads only
  `emailAddresses` and `names` (People API `otherContacts` and `people/me/connections`, at most
  5,000 entries from each), revokes the token at once and holds the list only in memory.
- Matches: active, non-staff members with a public profile and no block either way. Hidden profiles,
  blocked members, staff and inactive accounts look exactly like non-members, so the response never
  shows whether an address has an account.
- Invites: only addresses the member ticks, at most 25 per rolling 24 hours, sent through our email
  provider with a "stop all invitations" link (confirm page, then the address hash goes on
  `invite_suppressions`). Suppressed addresses and existing accounts are skipped without telling
  the inviter. `contact_invites` keeps the plaintext address only during the send, then the
  SHA-256 hash only. Nothing else is stored.

## Configuration

| Setting | Where | Notes |
|---|---|---|
| `GOOGLE_OAUTH_CLIENT_ID` | Infisical → Render (API) | Web application client in Google Cloud |
| `GOOGLE_OAUTH_CLIENT_SECRET` | Infisical → Render (API) | Secret |
| `INVITE_SIGNING_KEY` | Infisical → Render (API) | Optional, 32+ random bytes. Signs state, contact tokens and stop links. If it is empty, a key is derived from the client secret, and rotating the secret then invalidates old stop links. Set it before launch. |
| `FEATURE_CONTACTS_IMPORT` | Render env or the staff console flag override | Turn it on only after verification |
| `CONTACT_INVITES_PER_DAY` | optional | Default 25 |
| `CORS_ORIGINS` | already set | The callback origin must be listed there |

## Owner steps in Google Cloud

1. Google Cloud console → pick (or create) the project → APIs & Services → Library → enable
   **Google People API**.
2. OAuth consent screen (Google Auth Platform → Branding):
   - App name "Remote AI Platform", support email, logo (120×120), and the app domain
     `remoteaiplatform.com`.
   - Application home page: `https://remoteaiplatform.com`
   - Privacy policy: `https://remoteaiplatform.com/privacy`. It now has a Google contacts paragraph
     with the Limited Use statement. Check that it is deployed before you submit.
   - Terms: `https://remoteaiplatform.com/terms`
   - Authorized domains: `remoteaiplatform.com`. Verify domain ownership in Google Search Console
     with the same Google account.
3. Data access → add the scopes `.../auth/contacts.other.readonly` and
   `.../auth/contacts.readonly`. Both are **sensitive** scopes. If you only want people the member
   has emailed, drop `contacts.readonly`: there is one fewer scope to justify, and the code accepts
   either.
4. Clients → create an OAuth client ID of type **Web application**:
   - Authorized redirect URIs: `https://remoteaiplatform.com/contacts/callback`, and
     `https://dev.remoteaiplatform.com/contacts/callback` for dev (use a separate dev client if you prefer).
   - Copy the client ID and secret into Infisical, which syncs them to Render.
5. While the app is unverified, keep the publishing status on **Testing** and add test users (up to
   100). They will see the "unverified app" screen, which is acceptable for testing.
6. Submit for verification (Verification Center → "Prepare for verification"):
   - Scope justification, for example: "Members can choose to find people they already know on Remote AI
     Platform. We read only names and email addresses once, in memory, to show which contacts
     already have a public profile, and to let the member pick contacts to invite. We do not store
     the contact list or the token (online access, revoked right after the read)."
   - **Demo video** (unlisted YouTube link) that shows: the consent screen with the app name and the
     scopes in the URL bar, the "Find people you know from Google" button, the Google consent, the
     results page (matches with Connect, the invite checklist), sending an invite, and the email
     with its stop link. Show the OAuth client ID in the URL during consent.
   - Answer the follow-up email from Google's Trust & Safety team. Sensitive-scope reviews usually
     take a few days to a few weeks. A CASA security assessment is needed only for *restricted*
     scopes, not for these.
7. After approval, set the publishing status to **In production** and switch on
   `FEATURE_CONTACTS_IMPORT` (env or staff console). Test once with a non-test-user account.

## Rollback

Turn the flag off. Every endpoint then returns 404 and the button is hidden. Data is limited to
`contact_invites` (hashes) and `invite_suppressions`. Keep the suppression list even after a rollback.
