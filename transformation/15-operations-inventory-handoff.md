# Phase 15 operations inventory and backup evidence — 14 November 2026

Scope: dedicated read-only API router and console component. No schema changes,
provider calls, backup execution, commits, pushes or deployments. Concurrent
meeting changes in the product checkout were left untouched.

## Coordinator wiring

In `apps/api/app/main.py`, import:

```python
from app.domains.admin.inventory_router import router as admin_inventory_router
```

Next to the other admin router registrations inside the app factory, add:

```python
app.include_router(admin_inventory_router, prefix=prefix)
```

In `apps/admin/src/console/Console.tsx`, import `OperationsInventory` from
`./inventory`, add `["inventory", "Operations inventory", "layers"]` to the
operations navigation group, and add `inventory: <OperationsInventory />` to
the screen map. Alternatively embed the component in the operations screen.
Shared wiring files were deliberately left for the integration owner.

## Evidence semantics

`GET /api/v1/admin/operations/inventory` checks the existing ADMIN dependency.
Both member roles and anonymous requests are denied. It resolves registered API
paths through the running app's OpenAPI schema, including lazy included routers,
groups paths by API domain, and lists only existing GET operational views.
This is a process route catalogue, not a provider catalogue or health probe.
Routes excluded from OpenAPI are outside this inventory. API domain names and
path counts are metadata; no credentials, user content or storage URLs are read.

The backup section always reports `evidence_unavailable`, with null last-backup
and last-verified-restore timestamps. Existing scripts do not publish their run
results to the API. This does not prove that backups failed or never ran.
No provider backup, encrypted archive or restore has been verified by this work.
No writable evidence endpoint or upload of operator-supplied success claims exists.

## Backup procedure and acceptance evidence

Use product `ops/backup/README.md`, `backup.sh` and `restore.sh`. The backup covers
the database public schema and object bytes/metadata; it excludes authentication
users/sessions, grants/roles, bucket policies and provider configuration.

Retain an off-provider encrypted archive, its checksum and manifest timestamp.
Use the restore dry run against a separate empty database and buckets, review the
printed target identity, then follow the existing explicit target confirmation.
Retain output verifying row counts, object bytes and metadata. Review dangling
file-reference warnings separately: the script can print its final verified
message even when those warnings exist. Its reference scan currently covers
profile resume URLs; broader Files and attachment references need independent
review before claiming complete file-reference integrity. Reapply erasures newer
than the archive before any production restoration. Keep passphrases separate.

Success evidence must identify the archive, environment, execution time and restore
target without credentials. Wiring this page does not satisfy the Phase 16 restore
or Phase 17 verified-backup gates. A future evidence reader needs a durable trusted
run source, freshness policy and environment binding before showing success.

## Focused validation

Backend: `DEBUG=false .venv/bin/pytest tests/test_operations_inventory.py -q`
from `apps/api`. Tests mount an isolated app so the coordinator can wire main.py
independently; they verify real authentication/role enforcement, runtime route
discovery, null success timestamps, absence of credential field names and 405 for
mutating methods. Run Ruff and mypy on the dedicated router too.

Frontend: `npm test -- --run src/console/inventory.test.tsx` and `npm run lint`
from `apps/admin`. Tests cover unavailable evidence, navigation and fetch failure.
After shared wiring, verify the actual console navigation and generated OpenAPI;
these have not been verified in a browser by this work.
