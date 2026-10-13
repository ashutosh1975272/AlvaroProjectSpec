# 08: Deposit rules engine [BETA]

**What to build:** Configurable, risk-based deposits (DOCX §4F). **(0 — Schema migration, required first:)** `BookingRequest` has no deposit fields today (verified: zero `deposit` matches across `base44/entities/*.jsonc`) — add `deposit_amount`, `deposit_status`, `deposit_expiry`, `deposit_risk_class` so the evaluation's decision persists on the booking. (1) A server-side **deposit evaluation** that runs inside `submitPublicBooking` after price/duration recalculation and before record creation: it applies the configured risk classes — **new client** (no prior completed booking), **after-hours** booking, **group** of `DEPOSIT_GROUP_MIN_PEOPLE` or more, and **poor cancellation history** (≥ `DEPOSIT_NO_SHOW_THRESHOLD` no-shows/cancellations) — and writes the resulting deposit (amount or percentage, status, expiry) onto the booking. Deposits are **never forced on everyone** (spec: "rather than forcing deposits for everyone"). **After-hours defined from existing code (not invented):** a booking is after-hours when `after_hours_mode` and `after_hours_slot` are set — the exact test `updateBookingRequest/entry.ts:91` already uses — so the deposit rule reuses the platform's own after-hours definition. (2) The **phone-key unification fix**: `submitPublicBooking`'s `normPhone` (→ `"412345678"`, `entry.ts:30-36`) and `checkBookingStatus`'s `normalizeAuPhone` (→ `"0412345678"`, `entry.ts:71-78`) currently cannot recognise the same customer — this ticket **delivers** the shared canonical normalizer (new `base44/shared/phoneNormalization.ts`) used by every endpoint, so client history (and deposit/no-show history) is findable by phone. Ticket 14 consumes this module. (3) Deposit state is exposed to the booking flow and the admin views.

**Blocked by:** 04 (deposit messages attach to the booking flow)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/08-deposits` → review → merge into `beta`

**Requires:**
- External services: none new (Square link creation is ticket 09; this ticket only decides *whether* a deposit is required and the amount).
- Prerequisites: ticket 04 complete; **business sign-off on deposit amounts** (defaults in `.env.example`: `DEPOSIT_NEW_CLIENT_AMOUNT=20`, `DEPOSIT_AFTER_HOURS_AMOUNT=30`, `DEPOSIT_GROUP_PERCENT=20`, `DEPOSIT_GROUP_MIN_PEOPLE=4`, `DEPOSIT_NO_SHOW_THRESHOLD=2`).
- Testing needs: customer personas — first-time client, returning client, after-hours booking, group of 4+, a client with 2 recorded no-shows, and a regular client with clean history (must require no deposit); test phones for the normalization check.

**What you get:** Higher-risk bookings automatically require a deposit at submission (with the amount visible before the customer confirms), while regular clients book deposit-free — and the system can finally recognise the same customer across endpoints by phone.

**How to test:**
1. Submit a first-time-client booking → deposit required (amount from config), shown in the review step.
2. Submit an after-hours booking → deposit required.
3. Submit a group booking with 4+ people → percentage deposit applied.
4. Submit a booking for a client with 2 no-shows → deposit required.
5. Submit a regular client with clean history → **no** deposit.
6. Look up the same customer by phone in both the public status check and the admin view → same client recognised (phone keys unified).

**What you see:** Deposit decisions on the booking record with the triggering risk class visible to admin; consistent phone matching across endpoints; no deposit on low-risk bookings.

- [ ] Four risk classes evaluated server-side at submission (after-hours = `after_hours_mode` + `after_hours_slot`, per updateBookingRequest/entry.ts:91)
- [ ] Deposits never forced on everyone
- [ ] Amounts/percentages/thresholds read from config
- [ ] Deposit schema fields added (deposit_amount/deposit_status/deposit_expiry/deposit_risk_class — BookingRequest has none today)
- [ ] Deposit state (amount, status, expiry) stored on the booking
- [ ] Shared `base44/shared/phoneNormalization.ts` delivered and used by both public endpoints (fixes normPhone vs normalizeAuPhone mismatch, submitPublicBooking/entry.ts:30-36 vs checkBookingStatus/entry.ts:71-78)
- [ ] Merged into `beta` after Standards + Spec review
