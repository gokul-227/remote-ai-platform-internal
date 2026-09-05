# IDOR/BOLA Audit — contracts, groups, quality

Resolves Phase 0 report P1-3. 2026-09-05.

**Bottom line: no exploitable IDOR/BOLA confirmed in any of the three domains.** Traced every mutating and sensitive-read endpoint from router through to the DB query in each.

## quality/
No attack surface for IDOR to apply to: `evaluate_submission`, `review_code`, `batch_evaluate` all take content directly in the POST body, statelessly. There's no persisted "Submission" model and no ID-based lookup anywhere in this domain — nothing owned to leak or tamper with cross-user.

## contracts/
Every mutating/sensitive-read endpoint checks party membership before touching the row (`current_user.id in (contract.client_id, contract.worker_id)`, 404 not 403 so existence isn't leaked). `create_contract` correctly verifies the calling company owns the target project before linking a contract to it — the exact company_id-spoofing class of bug found elsewhere earlier, handled correctly here.

Two low-severity **business-logic** (not IDOR) gaps noted for a future product decision:
- `add_milestone` lets either party (including the worker) add milestones with arbitrary amounts — doesn't self-execute since APPROVED/PAID still needs the client, so not exploitable for fund theft, but a scope gap.
- Milestone status has no guard against moving backward (e.g. PAID → PENDING). Checked whether "PAID" triggers real escrow release — it doesn't; contracts aren't wired to `payments/` at all currently, so this is cosmetic bookkeeping, not a payments bypass.

## groups/
Private-group gating applied consistently across every read/write path. Role/ownership checks (`update_group`, `update_member_role`, `delete_group`, `delete_group_post`) are all correctly scoped per-group, not globally. No kick/ban endpoint exists at all, so that specific escalation scenario doesn't apply (nothing to exploit yet).

One functional gap (fails closed, not a vuln): private-group `join_group` creates a `pending` membership, but no endpoint transitions pending → active. Private groups may be effectively unjoinable today — but pending members are correctly excluded everywhere, so this is a missing feature, not an access-control hole.

## Recommendation
No P0/P1 action needed from this audit. The two contracts/ business-logic gaps are worth a product decision whenever contract/milestone UX gets real design attention, and the groups/ join-approval flow is a missing feature to build when private groups become a priority, not a security fix.
