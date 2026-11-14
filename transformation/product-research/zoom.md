# Zoom — video meetings (research package, 14 Nov 2026)

Evidence labels: **[Observed-web]** signed-in Zoom web portal (Home, Schedule a Meeting) snapshot; **[Observed-native]** REA native UI observation of the running macOS Zoom app (see phase record for evidence id); **[Documented]** Zoom support (official KB pages render client-side and returned empty bodies to anonymous fetch; waiting-room details below are **[Secondary]** from university help pages); **[Inferred]**.

## Product overview
Meetings, scheduling, recordings, AI summaries, whiteboards, docs, notes, clips, scheduler/booking pages, tasks.

## Feature inventory
- Portal nav: Home, My Products, AI, Meetings, Recordings, Summaries, Hub, Whiteboards, Notes, Clips, Canvas, Paper, Sheets, Slides, Tasks, Scheduler, My Account, Admin, Support **[Observed-web]**.
- Home: Host / Join / Schedule, Personal Meeting ID with Copy, booking links (view/create page, meeting poll), upcoming meetings (empty state "No Upcoming Meetings"), Test Audio and Video **[Observed-web]**.
- Schedule form: Topic, description, date, start time, duration (Basic plan capped at 40 min — plan limit surfaced inline), time zone, Recurring, Invitees (calendar connection required to deliver invites), Meeting ID (generate vs personal), Template, Whiteboard/Docs attachments, Security: Passcode (required), Waiting Room, Encryption (enhanced / end-to-end), AI auto-start toggles, workflow, notes permissions (org only / all), meeting chat before/after, Video host/participant on/off, interpretation, Save/Cancel **[Observed-web]**.
- Waiting room: host/co-host admit one or all; send participant back to waiting room; option for admitted users to admit others when host absent; account-level lock **[Secondary]**.
- Participant management: mute, rename, remove, lock meeting, screen-share permissions **[Documented previously: KB0058641]**.

## Detailed workflows
1. Schedule → invite (needs calendar) → link + passcode → participants join → waiting room admit → meeting → recording/summary.
2. Booking page: invitee picks a time.
3. Join by ID/passcode or link.

## Roles and permissions
Host, co-host, alternative host, participant, waiting-room participant; account admin **[Documented/Inferred]**.

## Data model hypotheses
Meeting(topic, start, duration, tz, recurring rule, meeting_id, passcode, waiting_room, encryption, settings), Invitation(status), Participant(role, admitted_at, muted), Recording, Summary, BookingPage, Poll.

## Observed interaction patterns
- Plan limits explained inline where they bite (40-minute cap).
- Security section groups passcode / waiting room / encryption.
- Dependencies surfaced before failure ("Participants won't receive this invite until your calendar is connected").

## Official documentation
- Screen-sharing/participant controls: https://support.zoom.com/hc/en/article?id=zm_kb&sysparm_article=KB0058641

## Observation record
Tabs "Home - Zoom" and "Schedule a Meeting - Zoom", 14 Nov 2026. The owner's personal meeting ID and generated passcode visible in the snapshot are not retained.

## Implementation opportunities
- Meeting settings: passcode, waiting room (host admits), recurring rule, time zone, host/participant video defaults; expose "Test audio and video" pre-join page.
- External meeting link (Jitsi/other) remains the supported calling path; native WebRTC stays labelled experimental until TURN is available.
- Booking links (availability slots) are a later feature.

## Acceptance criteria
- Only invited/admitted participants can join; join outside window or after cancellation fails; waiting-room admit is host-only.
- Meeting invite emails go through the notification system with preference checks.
- UI never claims recording/transcription until implemented.
