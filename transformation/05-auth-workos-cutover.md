# Sign-in migration: Supabase Auth to WorkOS AuthKit, cut-over and rollback (15 Nov 2026)

> Status: the code is built on branch `feat/workos-auth` in the product repo. It is not merged and not deployed. Nothing in production has been changed. This runbook does not authorize deployment. The coordinator runs the real user import and the cut-over.

Owner decision: after cut-over WorkOS is the **only** sign-in provider and Supabase Auth gets deleted. The code lets you switch Supabase off completely with configuration alone, and you can switch it back on until the Supabase project is actually deleted.

## 1. Design

- **Tokens.** The API accepts WorkOS AuthKit access tokens (RS256) as well as Supabase tokens. It checks them against `https://api.workos.com/sso/jwks/<WORKOS_CLIENT_ID>`. The `iss` claim must be `https://api.workos.com/user_management/<client_id>` (from WorkOS's discovery document) or `https://api.workos.com` (from WorkOS's docs). You can override the accepted issuers with `WORKOS_ISSUERS` when a custom auth domain is used. `aud` is checked against the client id only when the token has one. Code: `apps/api/app/domains/auth/supabase_auth.py` (`verify_workos_token`, `verify_access_token`).
- **Account mapping.** AuthKit access tokens carry no email. The first time a WorkOS subject is seen, the API reads the user from the WorkOS API (`GET /user_management/users/{id}`). If `email_verified` is true and the email matches an existing `users` row, the API links the subject through `user_identities` with provider `workos`. `users.auth_subject` (the Supabase id) is never touched. An unverified email never links (409). An unknown verified email creates a new member, the same as today. If WorkOS cannot be reached, sign-in fails closed with a 503. Later requests use the stored link and make no WorkOS call. Code: `auth/service.py`, built on the existing dual-issuer support.
- **Sign-in screens are unchanged.** The web app and the staff console keep their own screens and copy. The browser calls the API (`/api/v1/auth/workos/*`), and the API holds `WORKOS_API_KEY`:
  - email code: `magic-auth/send`, `magic-auth/verify`
  - Google/GitHub/Microsoft: `authorize`, which returns the WorkOS authorize URL (with state and PKCE S256), then `callback` on `/auth/callback`
  - session refresh: `refresh`, which rotates the refresh token
  - sign-out: `logout`, which revokes this WorkOS session, or every session with "sign out everywhere"
  - `config`
  - The redirect URI must be `<a CORS origin>/auth/callback`.
  - Code sending and verifying use the "sensitive" rate-limit tier.
- **Staff console.** It sends codes only to addresses that already have an account (`existing_only`), the same as Supabase's `shouldCreateUser: false`.
- **Account deletion.** It writes a provider-separated tombstone for the Supabase subject and for each linked identity, deletes the app data, and then erases the WorkOS user (`DELETE /user_management/users/{id}`) and the Supabase user (while one exists). The existing `retry_pending_erasures` job retries failed erasures. Migration `069_deleted_identity_provider` adds `deleted_identities.provider`.
- **The migration tool** is `apps/api/app/ops/migrate_users_to_workos.py`.

## 2. Configuration switches

| Where | Setting | Values | Meaning |
|---|---|---|---|
| API | `WORKOS_CLIENT_ID` | client id (public) | Empty: WorkOS is off and its tokens are rejected |
| API | `WORKOS_API_KEY` | `sk_test_…` / `sk_live_…` (Infisical) | Needed for sign-in proxying, the email lookup and erasure; production refuses a client id without it |
| API | `AUTH_PRIMARY_PROVIDER` | `supabase` / `workos` | Reported by `/auth/workos/config`; production refuses `workos` without the client id and key |
| API | `AUTH_ACCEPT_SUPABASE` | `true` / `false` | `false`: every Supabase token is refused, and `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are no longer required or warned about |
| API | `WORKOS_ISSUERS` | comma list | Only if a custom auth domain is used |
| API | `WORKOS_AUTHORIZE_URL` | URL | Only for a custom auth domain (the browser goes there for OAuth) |
| Web build | `NEXT_PUBLIC_AUTH_PROVIDER` | `supabase` / `workos` | `workos`: every sign-in call goes to WorkOS through the API, the Supabase client is inert (no session, refresh or URL parsing), and the Supabase build variables stop being required |
| Admin build | `NEXT_PUBLIC_AUTH_PROVIDER` | `supabase` / `workos` | Same for the staff console |

The Web and Admin settings are build-time (Docker build arg / static export), so changing them needs a rebuild and redeploy.

## 3. What the owner must do in the WorkOS dashboard

1. **Production environment.** Activate the production environment (it needs billing details; AuthKit is free up to 1M MAU). Copy the production **Client ID** and **API key** into Infisical `prod:/render-api` as `WORKOS_CLIENT_ID` and `WORKOS_API_KEY`. The key in `prod:/automation` is a **staging** key (`sk_test_`).
2. **Authentication methods, in staging and in production.** Turn on **Magic Auth** (email code). It is **currently disabled in staging**: a test call returned `authentication_method_not_allowed`. Turn on Google OAuth, GitHub OAuth and Microsoft OAuth. Passwords can stay off.
3. **OAuth credentials.** For each provider, choose "use your own credentials" and paste the client ID and secret. The existing client IDs are in Infisical prod as `GOOGLE_OAUTH_CLIENT_ID`, `GITHUB_OAUTH_CLIENT_ID` and `MICROSOFT_OAUTH_CLIENT_ID`, with their secrets alongside. Each provider then shows a WorkOS redirect URI. Add that URI in the provider's console:
   - **Google** (Cloud console → Credentials → OAuth client): add it as an extra authorized redirect URI. Keep the Supabase one until the rollback window ends.
   - **Microsoft** (Entra → App registration → Authentication): add it as an extra redirect URI.
   - **GitHub**: an OAuth app has only **one** callback URL. Either create a second GitHub OAuth app for WorkOS (recommended, because rollback keeps working) or switch the callback at cut-over.
4. **Redirects** (Dashboard → Redirects), production:
   - Redirect URIs: `https://remoteaiplatform.com/auth/callback` (the default) and `https://www.remoteaiplatform.com/auth/callback` (only if www serves the app).
   - Sign-out redirect / app homepage: `https://remoteaiplatform.com/#login`.
   - Staging: `https://dev.remoteaiplatform.com/auth/callback` and `http://localhost:3000/auth/callback`.
   - The staff console (`https://admin.remoteaiplatform.com`) signs in with email codes only, so it needs no redirect URI.
   - The API (`https://api.remoteaiplatform.com`) calls WorkOS server-to-server, so it needs no redirect URI either. Its `CORS_ORIGINS` must contain the web origins, as it already does.
5. **Email (recommended).** By default Magic Auth emails come from WorkOS with WorkOS branding. Set the branding and sender domain so the code email reads "Remote AI Platform". The product copy rules apply ("professional", no unverified claims).
6. **Custom auth domain (optional).** If one is set (for example `auth.remoteaiplatform.com`), set `WORKOS_ISSUERS=https://auth.remoteaiplatform.com` and `WORKOS_AUTHORIZE_URL=https://auth.remoteaiplatform.com` on the API.
7. **Sessions.** Keep the default short access-token lifetime and a session length of about 30 days. Refresh tokens rotate, and the apps store each new one.

## 4. Migration tool: what was run

```
cd apps/api
# dry run (default) against production Supabase: read-only GET /auth/v1/admin/users
SUPABASE_URL=… SUPABASE_SERVICE_ROLE_KEY=… python -m app.ops.migrate_users_to_workos --state <file>
# add --check-workos for read-only WorkOS lookups (needs WORKOS_API_KEY)
# real import at cut-over (a live key needs the explicit flag)
WORKOS_API_KEY=sk_live_… SUPABASE_URL=… SUPABASE_SERVICE_ROLE_KEY=… \
  python -m app.ops.migrate_users_to_workos --apply --allow-production --state <file>
```

**Production dry run, 15 Nov 2026** (counts only):
- 1,691 Supabase users, all of them with an email: 1,689 confirmed and 2 unconfirmed
- 0 deleted, 0 without an email
- Providers: email 1,664, google 25, github 1, azure 1
- The tool would import 1,691

**WorkOS staging run** (synthetic `@example.com` accounts only, from `--source-file`):
- Run 1, `--limit 2`: 2 created, 3 left for later; 1 deleted and 1 no-email record skipped
- Run 2: 2 created, 1 already in WorkOS, 2 skipped from the state file
- Run 3: nothing to do. The run is idempotent and resumable.
- WorkOS accepted `email_verified` and `external_id` (the Supabase id).
- User deletion was checked live through the API's own client: the delete succeeded, and a second delete returned 404, which counts as done.

What gets imported: email, `email_verified` (from `email_confirmed_at`) and `external_id` (the Supabase id). There are no passwords. OAuth links are not imported: WorkOS links a Google/GitHub/Microsoft sign-in to the imported user when the provider returns the same verified email. Deleted users and users without an email are skipped. Unconfirmed users are imported with `email_verified=false` and cannot link to an app account until they verify. The tool refuses `--apply` with a non-`sk_test_` key unless `--allow-production` is given. The report and the state file hold counts and ids only, never emails.

## 5. Cut-over steps (coordinator)

1. Merge `feat/workos-auth` and deploy the API with `WORKOS_CLIENT_ID` and `WORKOS_API_KEY` set, `AUTH_PRIMARY_PROVIDER=supabase` and `AUTH_ACCEPT_SUPABASE=true`. Run migration 069. Nothing changes for users. Check `GET /api/v1/auth/workos/config`: `primary=supabase`, three OAuth providers.
2. Staging smoke test with a staging web build (`NEXT_PUBLIC_AUTH_PROVIDER=workos`), once Magic Auth is on:
   - email code
   - all three OAuth buttons
   - refresh (wait for the access token to expire)
   - sign-out, and sign-out everywhere
   - account deletion; confirm the user is gone in WorkOS
   - **Decode one real access token**: check `iss` is in `WORKOS_ISSUERS` (or the defaults) and note whether `aud` is present.
3. **Import:** run `--check-workos` dry, then `--apply --allow-production`. Re-run until `failed` is 0. Archive the JSON report.
4. Set `AUTH_PRIMARY_PROVIDER=workos`. Rebuild and deploy web and admin with `NEXT_PUBLIC_AUTH_PROVIDER=workos`. New sign-ins go to WorkOS. Existing Supabase sessions keep working.
5. Watch for 24 to 72 hours:
   - API logs for `WorkOS user lookup failed` and 409 sign-in refusals
   - `user_identities` rows with provider `workos`, which should grow toward the active user count (33 app users today)
   - support mail
6. Turn Supabase off: set `AUTH_ACCEPT_SUPABASE=false`. Remaining Supabase sessions get 401, so those users sign in once through WorkOS and are linked by verified email. Drop `NEXT_PUBLIC_SUPABASE_*` from the web and admin builds.
7. After the rollback window (2 weeks suggested):
   - remove `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` from the API
   - delete Supabase Auth
   - remove the Supabase redirect URIs from Google and Microsoft
   - retire the old GitHub OAuth app

## 6. Rollback

- **Before step 6.** Rebuild web and admin with `NEXT_PUBLIC_AUTH_PROVIDER=supabase` and set `AUTH_PRIMARY_PROVIDER=supabase`. Everyone signs in with Supabase again. `users.auth_subject` was never changed. WorkOS links in `user_identities` are simply unused. Accounts created through WorkOS during the window have no Supabase subject: their owners sign in with Supabase using the same email, and the API attaches the subject to the row because `auth_subject` is empty.
- **After step 6.** Set `AUTH_ACCEPT_SUPABASE=true` first, then follow the steps above.
- **Emergency: to make every WorkOS token invalid,** unset `WORKOS_CLIENT_ID`.
- **Do not delete** the WorkOS users or the `user_identities` rows on rollback. They are inert.
- **After step 7 there is no rollback.**

## 7. Verification evidence (branch `feat/workos-auth`)

- **API unit tests** (`tests/test_workos_auth.py`, `tests/test_migrate_users_to_workos.py`) cover:
  - signature, issuer and audience checks, including expired, wrong-key and wrong-client tokens
  - linking by verified email only, with `auth_subject` untouched
  - an unverified email refused
  - a WorkOS outage failing closed
  - tombstones blocking re-creation
  - erasure of both providers, and the erasure retry
  - every proxied endpoint and the redirect allow-list
  - rate-limit classes and production config validation
  - the migration tool: dry run, apply, resume, live-key refusal, conflicts, failures
- **Full API suite:** 848 passed, coverage 89%.
- **Web and admin vitest** cover the WorkOS client modules and the deploy-env rules.
- **Playwright** (`tests/e2e/tests/auth-workos.spec.ts`, mock `tests/e2e/mock-workos/server.py`, overlay `tests/e2e/docker-compose.e2e-workos.yml`; added to the CI e2e job):
  - code sign-up, silent refresh, sign-out with session revocation, sign-in again to the same account
  - a wrong code refused
  - Google through the AuthKit redirect
  - account deletion erasing the WorkOS user
  - `privacy.spec.ts` and `navigation.spec.ts` also pass on the WorkOS stack

## 8. Known gaps

- No real AuthKit access token has been inspected yet: Magic Auth is disabled in staging, and password sign-in returned `sso_required`. The claims follow WorkOS's documentation and the discovery document. Cut-over step 2 covers the check with a real token.
- On the web app, signing in with an email code no longer refuses an unknown address: WorkOS creates the user on first verification, as OAuth always did. The staff console still refuses unknown addresses.
- Magic Auth emails carry WorkOS branding until section 3, step 5 is done.
