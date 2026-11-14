# Microsoft Word, Excel, PowerPoint — document, spreadsheet and presentation management (research package, 14 Nov 2026)

Evidence labels: **[Documented]** Microsoft Support (Word accept/reject tracked changes; Excel co-authoring; OneDrive version history); **[Observed-native]** REA native observation of the running macOS Word/Excel/PowerPoint apps (5 Nov evidence ids in `../../go-live/evidence-2026-11-05/rea-native-access.json`; re-observed 14 Nov, see phase record); **[Inferred]**. No Office binary was decompiled; only accessibility trees and official help were used.

## Word
- Review: Track Changes on/off; accept/reject one at a time or all; Next/Previous navigation; right-click accept/reject; No Markup is only a view, Document Inspector reveals hidden tracked changes and comments; shared-for-review documents may lock tracking **[Documented]**.
- Comments: threads, reply, resolve (modern comments) **[Documented, prior research]**.
- Version history via File > Info > Version History (cloud-stored files); co-authoring requires OneDrive/SharePoint and .docx **[Documented]**.
- Native UI **[Observed-native, 128 nodes on 5 Nov]**: ribbon tabs (Home, Insert, Draw, Design, Layout, References, Mailings, Review, View), document canvas, comments pane.

## Excel
- Co-authoring requires Microsoft 365, .xlsx/.xlsm/.xlsb, cloud storage, AutoSave; last save wins with exceptions; offline edits merge on reconnect; version history per platform; restore only when nobody else is co-authoring; "File is locked"/"Refresh recommended"/"Upload failed" states when unsupported features or versions are present **[Documented]**.
- Native UI **[Observed-native, 108 nodes on 5 Nov]**: sheet tabs, formula bar, ribbon.

## PowerPoint
- Official collaboration article not reachable anonymously (404); features known from product: comments with mentions and resolve, co-authoring, version history, presenter view, export to PDF/video **[Inferred/Common]**.
- Native UI **[Observed-native, 146 nodes on 5 Nov]**: slide thumbnails pane, slide canvas, notes pane, ribbon.

## Data model hypotheses (deliverable-oriented)
Document(kind: doc|sheet|slides, title, owner, project/contract/milestone links), Version(number, object or JSON, author, created_at, change summary), Comment(anchor, thread, resolved), Share(role), ReviewRequest(status: requested/approved/changes_requested), Export(format).

## Current Remote AI Platform state (from Phase 0)
Native TipTap document editor, bounded sheet formula engine, slide editor; DOCX/XLSX/PPTX import/export; per-document comments, versions with restore, shares; row-locked saves. **Missing**: previews of uploaded Office/PDF deliverables, tracked-changes, review/approval workflow on documents, folder placement, PDF export.

## Implementation opportunities (Phase 12)
- Upload-and-preview path for DOCX/XLSX/PPTX/PDF deliverables: PDF rendered client-side (pdf.js); DOCX → HTML preview via mammoth (already a dependency); XLSX → grid preview via exceljs (already a dependency); PPTX → slide text/outline preview (JSZip-based) with honest "download for full fidelity" label. No server-side macro/formula execution.
- Review and approval: request review → reviewer comments → approve / request changes, tied to deliverables and milestones.
- Version history UI with compare (text diff for docs), restore creating a new version.
- Comments with resolve and mentions feeding notifications.
- Export: DOCX/PDF for docs, XLSX/CSV for sheets, PPTX/PDF for slides (PDF via browser print for a bounded first step; label as such).

## Acceptance criteria
- Uploading a 10 MB .docx produces a preview or an explicit "preview unavailable, download" state; macros never execute; XLSX formulas are displayed, not evaluated server-side.
- Reviewer without share → 404; viewer cannot save; approval records who/when and is visible on the milestone.
- Restoring version N creates version N+1 identical to N; concurrent saves yield exactly one 409 (PostgreSQL test exists).
