# Brave — safe browsing concepts for the project demo viewer (research package, 14 Nov 2026)

Evidence labels: **[Documented]** brave.com/privacy-features; **[Observed-native]** REA native observation of the running Brave Browser window (5 Nov evidence, 54 nodes; re-observed 14 Nov, see phase record); **[Inferred]**.

## Documented concepts
- Shields: blocks trackers, cross-site cookies, fingerprinting; per-site panel shows what was blocked **[Documented]**.
- Storage partitioning per site (ephemeral third-party storage) **[Documented]**.
- Temporary hardware permissions (camera/microphone) with "until I close this site" / "24 hours" options **[Documented]**.
- Private windows with Tor **[Documented]**.
- Native UI **[Observed-native]**: tab strip, address bar with Shields icon, bookmarks bar, window controls.

## Relevance to Remote AI Platform (not a browser engine)
The product needs a **secure project-demo viewer**: users add HTTP(S) demo URLs to projects/milestones, open them in a sandboxed frame, review and leave feedback, approve or request changes.

## Current state (Phase 0)
`demoUrl.ts` validation (http/https only, private-network rejection), sandboxed iframe with CSP `frame-src https:`, "open in new tab" fallback when the destination forbids framing, retired server-side proxy (410). Review notes exist on deliverables.

## Implementation opportunities (Phase 13)
- Demo registry per project: name, URL, environment (preview/staging/production), linked milestone, version; "last verified reachable" check performed by a worker using HEAD with a 5-second timeout and no body retrieval (no fetching of arbitrary HTML).
- Permission prompts are the browser's; the viewer must request none and must set `sandbox="allow-scripts allow-same-origin allow-forms allow-popups"` without `allow-top-navigation`.
- Review panel beside the frame: comments with timestamps, approve / request changes bound to the milestone state machine.
- Shields-style status strip: shows that the demo runs in isolation, the exact origin, and a warning when the destination blocks framing.

## Acceptance criteria
- Non-http(s), localhost, RFC1918 and link-local URLs rejected (422) server-side and client-side.
- The frame can never read parent cookies/tokens (different origin + sandbox); CSP forbids `frame-src http:`.
- Approve/request changes transitions the milestone; a reviewer outside the contract gets 404.
