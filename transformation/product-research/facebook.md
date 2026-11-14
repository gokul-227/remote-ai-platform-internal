# Facebook — social network and groups (research package, 14 Nov 2026)

Evidence labels: **[Observed]** signed-in Facebook tabs (home feed, Groups hub) accessibility snapshot, read-only; **[Documented]** Meta newsroom/help (help-centre pages returned localized shells without body text to anonymous fetch; group role details come from third-party summaries and are marked **[Secondary]**); **[Inferred]**.

## Product overview
Consumer social network: feed, stories, reels, groups, marketplace, messaging, notifications, saved items.

## Feature inventory
- Top nav: Home, Reels, Marketplace, Groups, Gaming; account controls: menu, Messenger, Notifications (unread count), profile **[Observed]**.
- Left rail shortcuts: profile, Friends, Saved, Creator dashboard, Support, Memories, Groups, Reels, "See more"; pinned shortcuts to groups/pages **[Observed]**.
- Composer: "What's on your mind", Live video, Photo/video, Feeling/activity **[Observed]**.
- Post: author, time, audience icon ("Shared with Public"), actions menu, Hide post, body, media, Like/React, Comment (count), Share (count), reaction tallies by type, comments with Like/React/Downvote/Reply, "Hide or report this" **[Observed]**.
- Groups hub: Your feed, Discover, Your groups, Create new group, "Groups you've joined" with last-active time, group settings; group post shows "Shared with Public group", Privacy: Public, Follow author, inline comment composer with sticker/emoji/photo/GIF **[Observed]**.
- Group governance: admins vs moderators; post approval and membership approval; public/private, visible/hidden **[Secondary/Documented newsroom]**.
- Report content via per-post/per-comment menu **[Observed]**.

## Detailed workflows
1. Compose post with audience → feed → reactions/comments/shares → notifications.
2. Join group (open or approval) → post (optionally held for approval) → moderation (remove, ban) → report.
3. Save post for later; hide post; unfollow author.

## Roles and permissions
Member, friend, follower; group admin, moderator, member, pending; page admin **[Inferred/Secondary]**.

## Data model hypotheses
Post(audience: public/friends/custom), Reaction(type, unique per user/post), Comment(parent, reactions), Share, Group(privacy, visibility, approval settings), Membership(role, status: pending/active/banned), Report(target, reason, status), SavedItem, Notification.

## Observed interaction patterns
- Audience is always visible on the post.
- Reactions are typed and tallied per type.
- Group list ordered by recent activity with "Last active" timestamps.
- Comments support downvote and nested replies.

## Official documentation
- Audience controls: https://about.fb.com/news/2021/03/more-control-and-context-in-news-feed/
- Group admin flexibility: https://about.fb.com/news/2025/11/facebook-update-group-admins-flexibility-protecting-member-privacy/

## Observation record
Tabs "(2) Facebook" and "(2) Groups | Facebook", 14 Nov 2026. Friends' names, stories and group names seen in the snapshot are not retained.

## Implementation opportunities (professional variant)
- Typed reactions with per-type tallies (today: single like + reactions table; verify types).
- Group "last active" ordering and "Your feed / Discover / Your groups" hub.
- Post audience chip on every post; save-for-later list.
- Moderation queue: post approval for groups, pending members, bans, report triage (admin console).

## Acceptance criteria
- Audience enforced server-side: outsider cannot fetch CONNECTIONS/PRIVATE posts via API.
- One reaction per user per post (type changeable); concurrent double-like test on PostgreSQL.
- Pending/banned members cannot read or post in private groups; reports create moderation records visible only to staff.
