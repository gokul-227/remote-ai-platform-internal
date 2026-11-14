# Jira — project and work management (research package, 14 Nov 2026)

Evidence labels: **[Observed]** signed-in Jira Cloud tabs (owner's sites, board SCRUM) snapshot; **[Documented]** Atlassian support; **[Inferred]**.

## Product overview
Work tracking with spaces (projects), work items (epic/story/task/subtask), boards, backlogs, sprints, dashboards, filters, automation and plans.

## Feature inventory
- Global: search, Create, Notifications, Help, Settings, account menu **[Observed]**.
- Sidebar: For you (Recommended / Assigned to me / Starred / Worked on / Viewed), Recent, Starred, Apps, Plans, Spaces (recent list, create space), Filters, Dashboards, Assets, Teams, Goals, Projects, Customize sidebar **[Observed]**.
- Space navigation: Summary, Backlog, Board, Docs, Development, "Add to navigation" **[Observed]**.
- Board: search board, filter by assignee avatars, Filter, Group, sprint selector ("SCRUM Sprint 0"), Sprint insights, View settings; columns To Do / In Progress / In Review / Done each with edit, collapse, more actions, Create work item; "Add status"; empty state "No work items — Create a work item to get started" **[Observed]**.
- Work item types observed: Epic, Story, Task, Subtask with avatars **[Observed]**.
- Automation, Share, Link contributing teams, full screen **[Observed]**.
- Workflows: statuses + one-way transitions; workflow schemes bind work types to workflows; rules (conditions/validators/post-functions); team-managed vs company-managed **[Documented]**.
- Permissions: global permissions, space permission schemes (categories: space access/config, work item permissions, time tracking, voters/watchers/comments/attachments), work-item security schemes, guests, Permission helper **[Documented]**.

## Detailed workflows
1. Create work item → assign → move across board columns (status transition) → Done.
2. Backlog grooming → sprint planning → active sprint → insights.
3. Filters/dashboards for cross-project views; "For you" aggregates assigned/starred/recent.

## Roles and permissions **[Documented]**
Site admin, space admin, member, viewer, guest; per-permission grants to users/groups/project roles; transition restrictions per permission.

## Data model hypotheses
Space, WorkItem(type, status, assignee, reporter, parent, sprint, labels, priority, due), Status(category: todo/in-progress/done), Transition, Board(columns→statuses), Sprint, Filter(JQL), Dashboard, Comment, Attachment, Watcher, Link(dependency), Automation rule.

## Observed interaction patterns
- Column = status; "Add status" from the board.
- Per-column create action and explicit empty states.
- Personal "For you" hub instead of a project-first landing.

## Official documentation
- Workflows overview: https://support.atlassian.com/jira-service-management-cloud/docs/overview-of-jira-cloud-workflows/
- Manage permissions: https://support.atlassian.com/jira-cloud-administration/docs/manage-permissions/
- Dependencies: https://support.atlassian.com/jira-software-cloud/docs/create-or-remove-dependencies-on-your-timeline/

## Observation record
Tabs "For you - Worked on - Jira" and "Board - SCRUM board - Jira", 14 Nov 2026. The owner's own issue titles were visible; none are retained here.

## Implementation opportunities
- Status categories (todo/in-progress/done) with per-project configurable statuses mapped to board columns (today: fixed statuses).
- "For you" hub: assigned to me / starred / worked on / viewed across projects.
- Saved filters and per-project roles (admin/member/viewer) with transition restrictions.
- Project "Docs" tab (knowledge pages, Phase 12/Notion) and "Development" tab (GitHub deliverables).

## Acceptance criteria
- Status transition validated server-side against the project's workflow; invalid 409; viewer role cannot transition (403).
- Board columns reflect configured statuses; moving a card persists and survives refresh.
- Removed member loses access to items, comments, attachments, filters.
