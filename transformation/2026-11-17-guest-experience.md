# Guest experience — 17 November 2026

Candidate bf223fb3156b3252401672737c761e5aaf648924 on feat/guest-light-signup/dev.
Integrated current7aaab40production baseline, preserving billing/Slack/latestSEO changes.
Production stays7aaab40 until candidate checks/deployment finish.

User resolved SEO tradeoff: preserve Google Jobs by keeping full descriptions public;
signup required to apply. No crawler-only content or login wall around descriptions.
Official rule: https://developers.google.com/search/docs/appearance/structured-data/job-posting

Changes: white/light guest pages even with dark OS/saved member preference; guest rendering
never overwrites saved appearance; simpler home hero/search/signup/sixjobpreviews and folded
role/country/FAQ directories. Crawlable links/canonical/sitemaps/descriptions/validJobPosting
markup retained. External/application links require a member session; anonymous job list,
detail, company-list and unified search redact external_url after copying shared cache data.
Auth-dependent responses private/no-store with Vary Authorization/Cookie. Application actions
route guests to registration and remember intended job, including imported job apply buttons.

Evidence:258frontend tests,48focusedbackendtests (guest/member/cacheorder/visibility/search/
billing),7browser tests including dark OS+saved dark preference, signup/noexternalpopup,
mobile widths, noJSHTML/accessibility. TypeScript/format/lint/Ruff/mypy passed. FullCI44074893619
and build-only44074893853 running at this checkpoint. No migrations or provider settings changed.

Operational note: an older692deployment retry was mistakenly run before refreshing production
state. Its migration guard rejected unknown080revision and restored newer7aaab40. Exact7aaAPI
andhealthy DB/Redis/worker/scheduler verified; current7aaCaddyfile restored (other shipped
compose/deploy/postgresfiles unchanged). No database revision downgrade or user data deletion.
Do not retry olddeploymentjobs; use currentcandidate workflows and verify prodtip first.

Follow-up:5562b3d preserves intended-job return after onboarding. FullCI at556 had all
backend/frontend/migration checks pass; its browser failures were two old cross-engine
navigation checks expecting the removed guest"View and apply"label. Updated the check to
verify public200/canonical/404and"Sign up to apply"->registration plus rememberedjob.
Targeted test passes Chromium/WebKit/Firefox(3tests). Product code unchanged by that testfix.
Finalcandidate da360a367607cd435279889677f70f0a70c2e7ae committed/pusheddevandguestbranch;
CI44079246882/build44079247042running. Production stays7aaab40until exactCIgreenanddeploy.
