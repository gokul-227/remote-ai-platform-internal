# REA reference-product research — 5 November 2026

Current instruction: use REA on the named reference products, and normal source tools on Remote AI Platform. Research now uses the owner's already-open Chrome session. Five signed-in web interfaces and six native application interfaces have been verified; see the latest evidence section below. The earlier isolated profile is historical and unused. No credentials are requested in chat.

## Actual REA calls

The REA6.8.0 MCP server was invoked with `inspect_web_page` for every product below through a separate Chromium profile. Each call used an exact selected target and origin allowlist, accessibility text enabled, and script sources, console text, JSON body shapes, WebSocket shapes, storage keys and storage fingerprints disabled. No competitor source-code recovery/decompilation was performed. Raw capture JSON stays locally under `/Users/gokulr/.local/share/rea-tool/public-observations`; the compact evidence-id index is in `evidence-2026-11-05/rea-reference-products.json`.

A successful tool response proves capture success, not feature completeness. Upwork returned a Cloudflare block page and is explicitly blocked. Facebook/LinkedIn login pages and other marketing pages do not establish signed-in application behavior. No access challenge was bypassed. Public docs cited in the product audit provide behavior specifications separately from these observations.

| Product | Public observation / boundary | Independent implementation mapping |
|---|---|---|
| Facebook | Public account entry and cookie controls; feed not accessible without login | Existing professional feed, audience rules, groups, reactions, reporting; permission regressions tested |
| LinkedIn | Public career, hiring, professional-community entry points | Profiles, public discovery, jobs, saved jobs and hiring; crawlable homepage and account conversion added |
| Upwork | Cloudflare access block; no authenticated workflow observation | Existing proposals/contracts/milestones from source and public help; no payment-protection claims |
| Jira | Public work-management, task and team feature presentation | Existing board/backlog/tasks/dependencies; removed-member task-access defect fixed |
| Uber | Public consumer service and request entry points; dispatch internals unknown | Existing explainable professional/work matching, availability/invitation lifecycle; no transport/payment product or proprietary-algorithm claims |
| Zoom | Public collaboration and meeting feature presentation; no live meeting | Existing scheduling, participant permissions, external links and bounded WebRTC; real restrictive-network calling unverified |
| Slack | Public collaboration/agent feature navigation; workspace threads not observed | Existing channels, threads, reactions, unread/mute; cross-channel retry disclosure fixed, composite history cursors and history UI added |
| GitHub | Public developer/platform navigation | Existing scoped OAuth, repository/commit delivery references; real private-repository OAuth unverified |
| Word | Public document-authoring feature navigation | Existing native document editing, comments/versioning, DOCX paths; PostgreSQL save/restore concurrency fixed |
| Excel | Public spreadsheet/data feature navigation | Existing sheets/formulas/XLSX; patched dependencies verified by XLSX formula/conditional-format round trip |
| PowerPoint | Public presentation-authoring feature navigation | Existing native slide editor/PPTX generation; patched dependencies verified with a real PPTX export |
| Brave | Public browser/privacy feature presentation | Bounded HTTPS demo viewer; retired server proxy that bypassed framing rules; direct sandbox and new-tab fallback verified |

## Research continuation

Use the owner's existing signed-in Chrome window and select each specific authorized target by exact origin. Observe the relevant create/read/edit/history/error/permission workflow using synthetic data and explicit permitted actions. Do not publish, send messages, invite people, incur charges or change real external data without explicit instruction. Product captures may contain third-party personal information; minimise/avoid such content and retain only redacted behavior evidence.

The remaining authored feature breadth, acceptance criteria and live-provider gaps are recorded in `2026-11-05-product-audit.md`. Full feature parity with every named product is not established by this research pass. Captures are provisional local research, not copied product assets or implementation code.

## Existing Chrome session — updated evidence

At the owner’s explicit request, switched to the already-open Chrome session. REA connected through the existing browser’s local DevTools endpoint. Unrelated tab inventory was discarded; only reference-product selectors remain locally. No Remote AI Platform REA calls are part of this pass.

REA successfully observed signed-in LinkedIn jobs/feed interfaces, Facebook navigation, and Upwork’s authenticated navigation (including proposals/messages/contracts). The earlier Upwork Cloudflare block applies only to the isolated public pass and is superseded for this existing-session observation. Jira’s current welcome target returned an inspection error; its deeper workflow remains unverified. Slack, Zoom, GitHub and Office application targets were not yet found in this session.

Only allowlisted product navigation labels, origin, coarse route category, evidence id and structural counts were retained in the redacted summary. Private post/message content, names, account identifiers, cookies, tokens, credentials and storage were not retained. Script sources and source-map fetches remain disabled. These are genuine REA MCP observations, not claims of proprietary implementation recovery or complete feature parity.

## Latest access verification and coverage boundary

The coordinating agent subsequently selected the correct Jira workspace and GitHub signed-in target. Jira navigation (Spaces/Dashboards/Teams/Plans/Create) was observed with evidence `ev_ab158c312ae189c5dbaec5f40399be48627ba1ac5035581e85b9574bf3ad76ac`; GitHub dashboard/issues/pull requests with evidence `ev_5f8f76c501c4145521d04f0acc11386b73f504207031b3aeb3342c7274c346a7`. The earlier Jira welcome-tab error is superseded for access to the selected workspace. The five currently verified signed-in websites are Facebook, LinkedIn, Upwork, Jira and GitHub. Uber remains public observation only.

Device Control and Data Access and screen capture are enabled on macOS27.0.1. All six native reference interfaces were then observed successfully through REA: Slack, Zoom, Brave, Word, Excel and PowerPoint. The redacted [native access evidence](evidence-2026-11-05/rea-native-access.json) records ids and structural counts only. No private document/message content is retained here. These observations establish native access, not complete feature exploration, format fidelity or all account-tier capabilities.

The [capability and acceptance register](2026-11-05-reference-capability-matrix.md) separates existing source, current additions, partial/missing features, provider-dependent proof and explicit scope exclusions for every core reference plus the enterprise research references named in section8.12. Native permission is no longer a blocker; deeper synthetic workflow research, integrated tests and live-provider/release proof remain outstanding.
