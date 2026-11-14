# Slack — communication and collaboration (research package, 14 Nov 2026)

Evidence labels: **[Observed-web]** signed-in Slack web client tab snapshot; **[Observed-native]** REA `observe_native_ui` on the running macOS Slack app (evidence `ev_b0de381c…`, window "general (Channel) - Gokul - Slack", 12 accessibility nodes, Electron shell exposes little structure); **[Documented]** Slack Help; **[Inferred]**.

## Product overview
Workspace-scoped messaging with channels, DMs, threads, huddles, canvases, lists and files.

## Feature inventory
- Top-level tabs: Home, DMs, Activity, Files, Agents & tools, Admin **[Observed-web]**.
- Sidebar tree: Threads, Huddles, Directories, Starred, Channels, Direct Messages, Apps; "Channel or user name" quick search; New message **[Observed-web]**.
- Channel header: star, channel details, invite teammates, start huddle, **Edit notifications (current: All new posts)**, search in channel, more actions; tabs Messages / Add canvas / List / Folder **[Observed-web]**.
- Composer: bold/italic/underline/strike, paragraph styles, link, lists, blockquote, code, code block, divider, attach, emoji, mention, video/audio clip, run shortcut, Send now, **Schedule for later** **[Observed-web]**.
- Threads: reply in thread, "also send to channel", follow/unfollow, Threads view sorted by unread, per-thread notification toggle **[Documented]**.
- Notifications: triggers (DMs, thread replies you follow, mentions, channel-wide alerts, keywords); "Everything" vs "Mentions and DMs" with per-channel exceptions; keywords; notification schedule; mobile delivery timing; sound/preview; email bundles every 15 min or hourly when inactive **[Documented]**.

## Screens and navigation
Three-pane: nav rail → sidebar tree → conversation with optional thread pane.

## Detailed workflows
1. Post → reply in thread → optionally broadcast to channel → followers notified.
2. Set channel-level notification override; mute.
3. Search in channel; file tab per channel.
4. Schedule a message for later.

## Roles and permissions **[Documented/Inferred]**
Workspace owner/admin, member, guest (single/multi-channel). Channels public/private; private channel membership is the access boundary.

## Data model hypotheses
Workspace, Channel(type), Membership(role, notification_level, muted), Message(parent_id for thread, scheduled_at), Reaction, File, Mention, UnreadCursor per member per conversation, NotificationPreference(global + per-channel + keywords + schedule).

## Observed interaction patterns
- Unread ordering in Threads view.
- Composer exposes rich formatting but defaults to a single-line text box.
- Notification level is visible in the channel header ("Current: All new posts").

## Official documentation
- Threads: https://slack.com/help/articles/115000769927
- Notification guide: https://slack.com/help/articles/360025446073
- Configure notifications: https://slack.com/help/articles/201355156

## Observation record
Web tab "general (Channel)" and native window, 14 Nov 2026. Workspace ids in URLs are not retained beyond this note; no message bodies were present.

## Implementation opportunities
- Per-channel notification level already exists (mute/levels); add **keywords**, **notification schedule**, and **email bundling** (15-min digest) in the unified notification system.
- Scheduled messages (`scheduled_at`) and Files tab per channel/project.
- Threads view (my followed threads, unread first).
- Message attachments bound to the Files application with channel-membership authorization.

## Acceptance criteria
- Membership enforced on every read/write; removed member loses history and search.
- Thread reply notifications only to participants/followers; "send to channel" broadcasts.
- Digest email contains at most one bundle per 15-minute window per user; preferences honoured.
