# 09 — Sign in with LinkedIn (owner runbook)

The code is done on `feat/growth`. LinkedIn sits next to Google, Microsoft and GitHub on the sign-in
screen, and the screen shows it only when the provider is enabled:

- WorkOS (`NEXT_PUBLIC_AUTH_PROVIDER=workos`): the API's `/auth/workos/config` lists `linkedin`,
  and `/auth/workos/authorize` sends `provider=LinkedInOAuth`. On a member's **first** sign-in,
  their name (`first_name` and `last_name`) and photo (`profile_picture_url`, https only) come from
  the WorkOS user and fill the new account. Existing accounts are never overwritten. Linking still
  needs a verified email, as it does for every provider.
- Supabase (during the transition): the button uses the provider id `linkedin_oidc` and appears only
  if the Supabase project has it enabled.

## 1. Create the LinkedIn app

1. Go to https://developer.linkedin.com → My apps → **Create app**.
2. App name "Remote AI Platform". The LinkedIn Page must be the company page (create one if needed,
   and a page admin must approve the association). Add the privacy policy URL
   `https://remoteaiplatform.com/privacy` and the logo, then accept the terms.
3. In the app's **Products** tab, request **"Sign In with LinkedIn using OpenID Connect"**. It is
   approved instantly and gives the scopes `openid`, `profile` and `email`.
4. In the **Auth** tab, copy the **Client ID** and **Primary Client Secret**.
5. In **Authorized redirect URLs for your app**, add the redirect URI that WorkOS shows in step 2.
   It looks like `https://api.workos.com/sso/oauth/linkedin/callback` or your custom auth domain.
   Copy it exactly from WorkOS.

## 2. Configure WorkOS

1. Go to the WorkOS dashboard → the environment (Staging, then Production) → **Authentication** →
   **OAuth providers** → **LinkedIn** → Enable.
2. Choose "Your app's credentials", paste the Client ID and Client Secret, and save. Copy the redirect
   URI it shows into the LinkedIn app (step 1.5).
3. Repeat for the production environment, with the same or a separate LinkedIn app.

No API env change is needed: the provider list is in code. Add nothing to Render.

## 3. Supabase (only while Supabase sign-in is still live)

Supabase dashboard → Authentication → Providers → **LinkedIn (OIDC)** → paste the same client ID and
secret, and add the Supabase callback (`https://<project>.supabase.co/auth/v1/callback`) to the
LinkedIn app's redirect URLs. If you skip this, the button stays hidden on Supabase.

## 4. Check

- Sign in with LinkedIn on dev with a new account. The name and avatar should come prefilled.
- Sign in with an existing account whose email matches the verified LinkedIn email. It should link
  to that account and not create a second one.
