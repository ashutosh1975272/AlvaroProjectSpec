# 15: Server-side migration of direct writes + RLS tightening [BETA]

**What to build:** Close the security gap the spec forbids (DOCX §4J). Today ~10 places write booking/payment/client state **directly from the browser SDK**, bypassing the server-side authority gate: `Schedule.jsx` (completion, declined, no_show, confirmed/waitlist, other status flips), `AdminRoute.jsx` (completion, the corrupting Send-Arrival write — already replaced in ticket 05), `Requests.jsx` (status edit), `AdminPayments.jsx` (mark paid/unpaid — already replaced in ticket 09), `AppointmentDetail.jsx` (service/duration/price edits), and `Clients.jsx` (booking merge via `updateMany`). **(0 — Migration order, highest-impact first:)** (1) `AdminRoute.jsx:196-199` Send-Arrival corruption → ticket 05's `advanceRouteStop`; (2) `Schedule.jsx:553,581,597,489,499,396` completion/decline/no-show/confirm/waitlist/status flips → server functions; (3) `AdminPayments.jsx:27,32` mark paid/unpaid → already replaced by ticket 09; (4) `Requests.jsx:72` status edit; (5) `AppointmentDetail.jsx:60` service/duration/price edits; (6) `Clients.jsx:260` booking merge. **(1)** Each transition moves to a gated server function using the existing authority model (`requireAdminActor` / `resolveSchedulingActor` with admin-any / staff-own-bookings tiers) and an `expected_updated_date` version check returning 409 `SCHEDULE_CHANGED` on conflict. **(2)** **RLS tightening**: revenue/spend data becomes admin-only at the entity level (today any signed-in `user` role can read most entities). **(3)** The silent `catch` in completion handling (`Schedule.jsx:571,648`) is removed — failures surface a human-readable reason (§4J "no silent failure"). **(4) Audit consistency:** `placement_checked_at` is currently written only by the Route Flow engine (`placementReason.ts:85`) and nulled by `updateBookingRequest/entry.ts:223` — manual placements must also stamp it, so the audit field is consistent across every placement path.

**Blocked by:** 05 (stop-state server function exists as the pattern)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/15-server-authority` → review → merge into `beta`

**Requires:**
- External services: none new.
- Prerequisites: ticket 05 (establishes the server-transition pattern and the stop states); tickets 09 where it touches payments.
- Testing needs: a staff-tier account (must be able to edit only their own bookings, never override); an admin account; attempts to perform each old browser-SDK write path directly (must be refused by RLS); a revenue-report read as a non-admin (must be denied); a conflict scenario (two admins editing the same booking — second gets 409 with a readable message).

**What you get:** Every scheduling, payment, and client state transition happens server-side through the authority gate with version checks — exactly what DOCX §4J requires — and sensitive financial data is readable by admins only.

**How to test:**
1. Perform each transition (complete, decline, no-show, confirm, edit services, mark paid, merge clients) → each now flows through a server function with an audit row naming the actor.
2. Attempt the old direct browser-SDK write → refused (RLS/validation); nothing changes.
3. As staff, edit another barber's booking → refused; edit own booking → allowed.
4. As staff, attempt an availability override → refused (automation/staff can never override; admin-only).
5. Two admins edit the same booking simultaneously → second receives 409 `SCHEDULE_CHANGED` with a human-readable message.
6. Read revenue/spend data as a non-admin → denied.
7. Force a completion failure → the error is visible (no silent catch).

**What you see:** An audit trail for every transition; staff correctly constrained; conflicts handled visibly; revenue locked to admins; zero direct browser writes to scheduling/payment state.

- [ ] Migration order followed (Send-Arrival corruption first, then Schedule.jsx, then Requests/AppointmentDetail/Clients)
- [ ] All ~10 browser-side write paths migrated to gated server functions
- [ ] Authority tiers enforced (admin-any / staff-own-bookings; overrides admin-only)
- [ ] Version checks on every transition (409 with human-readable message)
- [ ] Revenue/spend data admin-only (RLS tightened)
- [ ] Silent catch removed; failures surface reasons
- [ ] `placement_checked_at` stamped by manual placements too (audit consistency)
- [ ] Existing behaviour preserved (no regression to the working calendar)
- [ ] Merged into `beta` after Standards + Spec review
