# File management applications — Google Drive, Dropbox, OneDrive, Windows File Explorer, Apple Files (research package, 14 Nov 2026)

Evidence labels: **[Documented]** official help centres (Google Drive answers 2494822, 2375102; Dropbox version-history overview; OneDrive restore article); **[Inferred]** from documented behaviour; desktop file managers are **[Common knowledge]** of the platform conventions and were not observed through REA.

## Feature inventory (documented)
- **Roles**: Viewer / Commenter / Editor; editors may by default re-share and change permissions (toggle); download/copy/print can be disabled per item **[Drive]**.
- **Link scopes**: Restricted (only added people), Anyone with the link, Your organisation; expiry dates on access (work accounts, viewer role for folders) **[Drive]**.
- **Inheritance**: a file cannot be less accessible than its parent folder; tighten by moving **[Drive]**; Notion-style inheritance confirmed separately.
- **Ownership**: transfer ownership; deleting an owned file trashes it and collaborators keep access until permanent deletion; non-owners only "Remove" from their view **[Drive]**.
- **Trash**: 30-day retention, then automatic permanent delete; trash counts toward storage; empty trash is irreversible **[Drive]**.
- **Version history**: all file types; 30 days on OneDrive (admins can disable); Dropbox 30/180/365 days by plan; restoring makes the old version current and keeps the previous current as a version **[OneDrive/Dropbox]**.
- **Activity**: folder activity (edits, renames, uploads, deletions, moves) and per-file version history **[Dropbox]**.
- **Limits**: 600 direct shares per file, 100 simultaneous editors **[Drive]**.
- Standard navigation **[Common]**: My Files, Shared with me, Recent, Starred, Trash; breadcrumbs; list/grid; sort by name/modified/size/type; type filters (documents, images, videos, presentations, spreadsheets); storage usage meter; drag-and-drop upload; rename/move/copy/delete/restore; preview pane.

## Detailed workflows
1. Upload (single/multi, folder) → metadata row → preview → share (people or link, role) → recipient sees it under Shared with me.
2. Move/copy/rename with permission inheritance recalculated; moving into a more restrictive folder tightens access.
3. Delete → Trash (30 days) → Restore or permanent delete; shared recipients keep access during the trash window.
4. Edit → new version → version history → restore a version → new current version.
5. Star/recent for quick access; search by name, type, owner, date.

## Roles and permissions
Owner, editor (optionally can share), commenter, viewer; organisation admin can restrict external links.

## Data model hypotheses
Node(id, parent_id, kind: folder|file, name, owner_id, org_id, project_id, size, mime, starred_by, trashed_at, deleted_at), Version(node_id, number, object_key, size, created_by), Share(node_id, principal: user|org|project|link, role, expires_at, can_reshare), LinkToken(scope, expiry, download allowed), Activity(node_id, actor, action), StorageQuota(owner, used, limit).

## Official documentation
- Drive sharing: https://support.google.com/drive/answer/2494822
- Drive trash: https://support.google.com/drive/answer/2375102
- Dropbox version history: https://help.dropbox.com/delete-restore/version-history-overview
- OneDrive restore: https://support.microsoft.com/en-us/office/restore-a-previous-version-of-a-file-stored-in-onedrive-159cad6d-d76e-4981-88ef-de6e96c93893

## Implementation plan for Remote AI Platform Files (Phase 12)
- New domain `files`: `file_nodes`, `file_versions`, `file_shares`, `file_activity`; S3-compatible object storage (MinIO locally, Supabase Storage today, Cloudflare R2 when the owner enables it) with private buckets and signed upload/download only.
- Navigation per the plan: My Files, Shared with Me, Recent, Starred, Project Files, Organisation Files, Documents, Images, Videos, Presentations, Spreadsheets, Trash.
- Operations: create folder, upload (signed PUT + finalize), download (signed GET, 5-minute expiry), move, rename, copy (server-side object copy), delete→trash, restore, permanent delete (30-day sweep by worker), star, search (name + type + owner, trigram), sort, preview (image/PDF/text/CSV/Office-derived PDF where available), share (user/org/project/link with role and expiry), version history and restore.
- Quotas: per-user and per-organisation storage caps and max upload size, validated MIME/extension allow-list, no executable content.

## Acceptance criteria
- Non-shared user gets 404 on node, version and download; link token outside expiry 410.
- Restore from trash returns the node to its parent (or root if parent trashed) with shares intact; permanent delete removes versions and objects.
- Version restore creates a new version; the previous current version remains listed.
- Moving a file into a project folder grants project members access according to the folder's share.
