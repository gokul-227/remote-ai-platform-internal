# Configuration retirement register — 15 November 2026

This register authorizes no deletion and contains no credential values.

| Settings/service | Current dependency | Retirement condition |
|---|---|---|
| SUPABASE_URL and auth admin/JWKS settings | Existing account authentication and erasure | WorkOS production activation, verified identity import, login/recovery/revocation continuity and rollback evidence |
| MINIO_* names | Logical bucket mapping and optional legacy fallback | Do not delete by name: R2/OCI adapter maps logical MINIO bucket names to actual Oracle buckets |
| R2_* names | Current Oracle S3-compatible primary storage | Keep; names describe adapter settings rather than actual provider identity |
| SENTRY_* names | May configure compatible GlitchTip endpoint | Inspect destination before retiring; rename alone does not prove unused |
| /render-api Infisical path | VM render-env.sh still reads it | Migrate keys and update renderer before deleting path |
| Render/hosted Redis/Cloudflare Workers fallback | Time-bound old deployment recovery | End fallback window, verify VM backups/restore, DNS, auth and rollback first |
| JOB_SYNC_GITHUB_ENABLED | Transitional GitHub import execution | Keep disabled while VM scheduler owns imports; preserve rollback procedure |
| JOOBLE_API_KEY | Live source integration | Keep; verify source run outcome without exhausting daily request budget |
| ADZUNA credentials/license gate | Provider permission remains unconfirmed | Do not enable redistribution until owner has written permission |

Exposed credentials require owner/provider rotation; values must not be copied into reports.

## Backup configuration correction

The fresh pre-release backup failed because /opt/rap/backup.env still selected the old Supabase
S3 endpoint. Runtime primary storage was already Oracle through STORAGE_PROVIDER=r2 and R2_*.
Backup source settings were aligned with that verified primary adapter without printing credentials.
The corrected backup succeeded at 22:18 UTC on15Nov:98 MB,99 tables,5 objects, offsite upload succeeded,
and API evidence report returned201. The previous backup.env was retained root-only for recovery.
A successful backup does not substitute for the required restore drill after material schema changes.
