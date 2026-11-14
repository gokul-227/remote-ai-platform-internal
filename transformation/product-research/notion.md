# Notion — knowledge and workspace management (research package, 14 Nov 2026)

Evidence labels: **[Documented]** notion.com/help (sharing-and-permissions; what-is-a-database); **[Inferred]**.

## Documented model
- Pages contain blocks; a database is a page whose rows are pages; properties drive search/filter/sort; views: table, list, board, calendar, gallery, timeline, chart; linked data sources **[Documented]**.
- Sharing: invite specific people or groups; guests (external, need account); teamspaces; general access = only invited / everyone in workspace (optionally hidden from search) / anyone on the web with link (expiry possible); Notion Sites are indexable, public links are not **[Documented]**.
- Permission levels: Full access, Can edit, Can edit content (DB), Can create (DB), Can comment, Can view; subpages inherit parent permissions and may be overridden; broadest access wins **[Documented]**.
- Comments allowed only for comment/edit/full access; web visitors must log in to comment **[Documented]**.

## Relevance
The plan asks for a lightweight knowledge base integrated with projects, organisations and profiles, explicitly **not** a full Notion replacement before the core platform is complete.

## Implementation opportunities (bounded, Phase 12 extension)
- Reuse the existing `documents` domain (TipTap JSON) as "pages": add `parent_id` (nesting), `space` (project | organisation | personal), `slug`, page tree navigation and breadcrumb; inherit shares from parent with per-page override; "broadest access wins".
- Project "Docs" tab (as in Jira) listing the page tree; organisation knowledge base; templates (project brief, meeting notes, deliverable review).
- Search pages through unified search with ACL.
- Defer: databases/views, public sites.

## Acceptance criteria
- Child page inherits parent share; removing share on parent removes access to the child unless the child has its own share.
- Viewer cannot comment; commenter cannot edit; search never returns pages the user cannot open.
