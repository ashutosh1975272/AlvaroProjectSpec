# 16: Google Calendar integration [BETA]

**What to build:** The optional external calendar layer (DOCX §4B, §7, §12 Phase 5; ADR 0001, ADR 0005). (1) **Mirror out**: confirmed appointments are written to the business's Google Calendar (via Base44's native Google Calendar connector) after the confirmation transition; cancellations/deletes remove the Google event. (2) **Busy blocks in**: external Google Calendar busy time is read into the day's validation context and respected by the shared validator as unavailable time (it can block a slot but never creates or moves a Base44 appointment). (3) **Loop prevention**: events our integration creates carry a sync marker (extended property) and the Google event id is stored in a new `google_event_id` field on the booking record (verified: zero `google` matches across `base44/entities/*.jsonc` — the field does not exist yet); read-back ignores events carrying our sync marker or matching a stored `google_event_id`. (4) **Conflict policy**: Base44 wins; discrepancies surface as admin review items. (5) **Scope**: only confirmed, non-test appointments sync; provisional/locked/test data never appears in Google Calendar.

**Blocked by:** 04 (confirmation transition is the sync trigger), 09 (webhook/outbox patterns are reused)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/16-google-calendar` → review → merge into `beta`

**Requires:**
- External services/accounts to arrange: **Google Calendar connector connection** in the Base44 dashboard (OAuth via Base44's native connector — no app-side OAuth code); a **test Google account/calendar** to verify mirroring.
- Prerequisites: tickets 04 and 09 complete (Phase 5 per DOCX §12 — only after source-of-truth rules are agreed, which ADR 0001/0005 record).
- Testing needs: a test Google calendar; a confirmed booking; an external busy block added directly in Google; a cancellation to verify event removal; a sync-loop check (our own writes must be ignored on read-back).

**What you get:** Customers and staff who live in Google Calendar see confirmed appointments automatically, and external commitments (busy blocks) are respected by the scheduler — with Base44 remaining the single source of truth and no double-booking, exactly as DOCX §4B requires.

**How to test:**
1. Confirm a booking in Base44 → the event appears in the test Google Calendar with the customer's details.
2. Add an external busy block in Google → the shared validator refuses a new booking inside it (with the travel/overlap reason).
3. Cancel the Base44 booking → the Google event is removed.
4. Trigger a sync pass twice → no duplicate events (idempotent), and no loop (our own writes ignored on read-back).
5. Create a conflict (edit the same appointment's time in Google directly) → Base44 wins on the next sync; the discrepancy appears as an admin review item.
6. Confirm a TEST booking → nothing syncs.

**What you see:** Confirmed appointments mirrored to Google; external busy time respected by the validator; cancellations reflected; admin review items for discrepancies; test data never synced.

- [ ] Loop prevention: sync marker (extended property) + new `google_event_id` booking field; own writes ignored on read-back
- [ ] Confirmed appointments mirrored out to Google Calendar
- [ ] External busy blocks imported and respected by the shared validator
- [ ] Sync is idempotent and loop-guarded (own writes ignored on read-back)
- [ ] Base44 wins conflicts; discrepancies surface as admin review items
- [ ] Cancellations remove Google events
- [ ] Only confirmed, non-test appointments sync
- [ ] Merged into `beta` after Standards + Spec review
