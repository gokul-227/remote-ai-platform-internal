# Owner infrastructure guide — step by step (14 November 2026)

Who this is for: you, the owner. Every account below is created by you; I never create accounts or read secret values. You put each credential into **Infisical** (project `237deba5…`, https://app.infisical.com), and the existing Secret Sync carries it to Render and GitHub. Never paste a key into chat.

Do the steps **in this order**. Each one ends with "tell Claude", which is when I verify it.

| # | Service | Why | Card needed | Cost if you stay on the free tier |
|---|---|---|---|---|
| 1 | Infisical CLI login | So I can deploy with secrets without seeing them | No | Free |
| 2 | Upstash Redis | Production Redis is down today; the worker also needs it | No | Free (500k commands/month) |
| 3 | Oracle Cloud VM | Home for the API, worker and scheduler | **Yes** (identity check, no charge on Always Free) | Free |
| 4 | Cloudflare R2 | Private storage for Files, attachments, resumes | **Yes** (billing must be enabled) | Free up to 10 GB, 1M writes, 10M reads/month |
| 5 | Neon | Development/staging database (replaces the paused Supabase dev project) | No | Free |
| 6 | Grafana Cloud | Dashboards and alerts | No | Free |
| 7 | WorkOS (staging only) | Trial of the second sign-in provider | No | Free (staging is never billed) |
| 8 | Firebase | Browser push notifications (included in this release) | No | Free (Spark plan) |

Free-tier limits were read from each provider's pricing page on 14 Nov 2026. Check them again on the day you sign up; a free allowance does not guarantee a zero bill, so set every budget alert below.

---

## 1. Infisical CLI login (2 minutes)

1. Open Terminal on this Mac.
2. Run `infisical login`, choose **Infisical Cloud (US)**, and sign in in the browser window that opens.
3. **Tell Claude:** "Infisical logged in."

## 2. Upstash Redis (10 minutes) — fixes the current production outage

1. Go to https://console.upstash.com and sign up with GitHub or Google. No card.
2. Click **Create Database**.
   - Name: `rap-prod`
   - Type: **Regional**
   - Region: **eu-central-1 (Frankfurt)** (next to the API and database)
   - Eviction: **on** (allkeys-lru)
   - TLS: **on** (default)
3. Open the database. Under **Connect**, choose the **redis-cli / TCP** tab and copy the URL that starts with `rediss://default:…@…upstash.io:6379`.
4. In Infisical: environment **prod**, folder **/render-api** → **Add secret**: key `REDIS_URL`, value = the copied URL. Save.
5. Repeat for development: create `rap-dev` the same way; put its URL into Infisical **dev**, root folder, as `REDIS_URL`.
6. In Upstash, open **Usage** and note the monthly command count; the free plan stops at 500,000 commands rather than billing you (no card on file).
7. **Tell Claude:** "Upstash done." I then check that `https://api.remoteaiplatform.com/health/ready` shows Redis HEALTHY after Render picks up the new value.

## 3. Oracle Cloud Always Free VM (45–60 minutes)

1. Go to https://www.oracle.com/cloud/free/ → **Start for free**.
2. Country: Germany. **Home region: Germany Central (Frankfurt)**. The home region cannot be changed later, and Always Free compute only exists there.
3. Enter the card when asked. Oracle places a small temporary authorisation for identity; the account stays **Always Free** unless you click **Upgrade to Pay As You Go**. Do not upgrade.
4. After the account is active (can take up to a day), open **Billing → Budgets → Create Budget**: amount **1 EUR**, alert at **1%**, to your email.
5. Create an SSH key on this Mac: `ssh-keygen -t ed25519 -f ~/.ssh/rap_oci -C rap-oci` (press Enter for no passphrase or set one).
6. **Compute → Instances → Create instance**:
   - Name: `rap-backend`
   - Image: **Canonical Ubuntu 24.04** (aarch64)
   - Shape: **Ampere → VM.Standard.A1.Flex**, **2 OCPU, 12 GB memory** (the full Always Free allowance)
   - Networking: create new VCN, public subnet, **assign a public IPv4 address**
   - SSH keys: **Paste public key** → contents of `~/.ssh/rap_oci.pub`
   - Boot volume: 100 GB
   - If you see **"Out of capacity"**, retry later or pick another availability domain in the same region.
7. **Networking → Virtual Cloud Networks → your VCN → Security Lists → Default** → **Add Ingress Rules**:
   - TCP 80 from `0.0.0.0/0`
   - TCP 443 from `0.0.0.0/0`
   - Keep TCP 22, but change its source to your own IP address only.
   - Do **not** open 6379 (Redis) or 8000 (API).
8. Copy the instance's **public IP**.
9. In Cloudflare DNS (https://dash.cloudflare.com → remoteaiplatform.com → DNS): add **A** record `api-oci` → the public IP, **Proxied** on.
10. **Tell Claude:** "OCI VM ready, IP is in Cloudflare as api-oci." Send nothing else. I will ask you to run one command that adds my access, or you can run the setup steps from `04-infrastructure-runbooks.md` R3 yourself.

Note: Oracle reclaims Always Free instances that sit idle for 7 days (CPU, network and memory all below 20%). The worker importing jobs every few hours keeps it active. The VM is a single machine with no failover; Render stays as the fallback until the release.

## 4. Cloudflare R2 (15 minutes)

1. https://dash.cloudflare.com → your account → **R2 Object Storage**.
2. Click **Purchase R2 / Add payment method** and add the card. Then **Billing → Notifications → Create**: usage alert for R2 at **5 GB stored** and **500,000 Class A operations**.
3. Create these **private** buckets (location hint **Western Europe (WEUR)**, storage class **Standard**, public access **Disabled**). R2 bucket names are unique per account, so each environment gets its own:
   - Production: `rap-prod-resumes`, `rap-prod-assets`, `rap-prod-deliverables`, `rap-prod-files`
   - Development: `rap-dev-resumes`, `rap-dev-assets`, `rap-dev-deliverables`, `rap-dev-files`
4. **R2 → Manage API tokens → Create API token**: permission **Object Read & Write**, limited to the eight buckets above. Copy the **Access Key ID**, **Secret Access Key**, and the **S3 endpoint** `https://<account-id>.r2.cloudflarestorage.com`.
5. For each `*-files` bucket: **Settings → CORS policy**:
   ```json
   [{"AllowedOrigins":["https://remoteaiplatform.com","https://www.remoteaiplatform.com"],"AllowedMethods":["GET"],"AllowedHeaders":["*"],"MaxAgeSeconds":3600}]
   ```
   (use `https://dev.remoteaiplatform.com` for the dev bucket).
6. In Infisical **prod → /render-api** add (names must match exactly):
   - `R2_ENDPOINT` = the S3 endpoint
   - `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`
   - `R2_BUCKET_RESUMES=rap-prod-resumes`, `R2_BUCKET_ASSETS=rap-prod-assets`, `R2_BUCKET_DELIVERABLES=rap-prod-deliverables`, `R2_BUCKET_FILES=rap-prod-files`
   - Do **not** change `STORAGE_PROVIDER` yourself. I switch it during the release: first with the old storage kept as a fallback, then copy and verify every existing file with `ops/storage/migrate_objects.py`, then turn the fallback off.
7. Same in Infisical **dev** with the `rap-dev-*` names.
8. **Tell Claude:** "R2 done." See the storage decision below before the release.

## 5. Neon development database (10 minutes)

1. https://console.neon.tech → sign up with GitHub. No card.
2. **Create project**: name `remote-ai-platform-dev`, Postgres **16**, region **AWS Europe Central 1 (Frankfurt)**.
3. **Dashboard → Connect**: role `neondb_owner`, **Pooled connection** on, copy the connection string.
4. Change the beginning from `postgresql://` to `postgresql+asyncpg://` and replace `?sslmode=require` with `?ssl=require`.
5. Infisical **dev**, root: add `DATABASE_URL_NEON` with that value. (I switch the dev API over only after migrations and tests pass against it.)
6. **Tell Claude:** "Neon done."

## 6. Grafana Cloud (10 minutes)

1. https://grafana.com/auth/sign-up/create-user → sign up, no card. Stack name `rap`, region **EU**.
2. In the stack: **Connections → Add new connection → Hosted Prometheus metrics**. Create an access policy token with scope **metrics:write**. Copy the **remote-write URL**, **username/instance ID** and **token**.
3. Infisical **prod**: `GRAFANA_REMOTE_WRITE_URL`, `GRAFANA_REMOTE_WRITE_USER`, `GRAFANA_REMOTE_WRITE_TOKEN`.
4. **Tell Claude:** "Grafana done."

## 7. WorkOS AuthKit — staging only (15 minutes)

1. https://dashboard.workos.com/signup → sign up. No card.
2. Stay in the **Staging** environment (top-left switch). Do **not** create or activate Production.
3. **Authentication → AuthKit → Set up**: enable **Email + magic code**, **Google**, **GitHub**.
4. **Redirects**: add `https://dev.remoteaiplatform.com/auth/callback`.
5. **API keys**: copy **Client ID** and **API key**.
6. Infisical **dev**: `WORKOS_CLIENT_ID`, `WORKOS_API_KEY`.
7. **Tell Claude:** "WorkOS staging done." I then enable it on dev only, behind the allow-list, and test sign-in, recovery and rollback.

## 8. Firebase Cloud Messaging — browser push (20 minutes)

You chose to include push in this release.

1. https://console.firebase.google.com → **Add project** `remote-ai-platform`. Turn **off** Google Analytics. The plan stays **Spark (free)**; never click Upgrade.
2. **Project settings (gear) → General → Your apps → Web (`</>`)**: register an app called `web`. Do **not** set up Firebase Hosting. Firebase shows a `firebaseConfig` object; copy only these four values: `apiKey`, `projectId`, `messagingSenderId`, `appId`.
3. In GitHub → repository **gokul-227/remote-ai-platform → Settings → Secrets and variables → Actions → Variables**, add variable `FIREBASE_WEB_CONFIG` with this one-line JSON (your values):
   `{"apiKey":"…","projectId":"…","messagingSenderId":"…","appId":"…"}`
   These four values are public by design (they ship in the browser), so a variable is correct, not a secret. For dev use the same project and add `FIREBASE_WEB_CONFIG_DEV` with the same value.
4. **Project settings → Cloud Messaging → Web configuration → Web Push certificates → Generate key pair**. Copy the key (starts with `B…`).
5. **Project settings → Service accounts → Firebase Admin SDK → Generate new private key** (downloads a JSON file).
6. On this Mac: `base64 -i ~/Downloads/<that-file>.json | pbcopy`
7. In Infisical **prod → /render-api** add: `FCM_PROJECT_ID` = your project id, `FCM_CREDENTIALS_B64` = paste, `FCM_VAPID_KEY` = the `B…` key. Repeat in **dev**.
8. Delete the downloaded JSON file and empty the Trash.
9. **Tell Claude:** "Firebase done." I then test a real notification on dev in your browser.

---

## After everything: what I do, then the single production release

1. Build and test on dev with the real providers (Neon, R2 dev, Upstash dev, WorkOS staging).
2. Deploy the backend, worker and scheduler to the Oracle VM against the **dev** database first.
3. Take and **restore-test** a production backup into an isolated database.
4. Release to production once: migrations, API on the VM behind `api-oci`, worker and scheduler on the production database, GitHub job import switched off, then move `api.remoteaiplatform.com` to the VM.
5. Watch errors, queue age, storage and costs; keep Render as fallback for a week.
6. Restore branch protection on `prod` and `dev`.

## Still yours to do from earlier sessions

- Rotate the keys that were pasted into chat earlier (GitHub OAuth client secret, Infisical token, Postman key, Resend key).
- Recreate the Google OAuth client and set it in Supabase if you want Google sign-in back.
