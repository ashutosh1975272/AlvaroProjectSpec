# 06: Reminder campaign engine [BETA]

**What to build:** Automatic regular-client rebooking reminders (DOCX §4G, §6 rows 5 and 10). (1) A **server-side due-list computation** that reuses the existing client frequency model (`ReminderClient.frequency` + `custom_frequency_days`; 7/14/21/30-day mapping) and `last_visit_date`/`next_reminder_date` to determine who is due — replacing today's client-side-only maths. (2) A scheduled campaign that, on `REMINDER_OUTREACH_DAY_OF_WEEK` (Tuesday), enqueues the first outreach to due clients and, on `REMINDER_FOLLOWUP_DAY_OF_WEEK` (Thursday), enqueues the follow-up to non-responders — all sends through the outbox. (3) **Suppression rules** evaluated at send time: never message a client who already has a booking this week, has opted out (STOP from ticket 03), is paused, has an unresolved conversation needing human attention, or is `is_test`. **"Unresolved conversation" defined (DOCX §4G requires the suppression but not the mechanism):** a client is suppressed when their latest conversation thread contains an escalation that is not yet resolved. Implementation: `ReminderClient` gains a `conversation_status` field (`none | open | escalated | resolved`), set when the AI assistant escalates (ticket 07) and cleared by an admin; the campaign suppresses `open` and `escalated`. (4) **Real-lead capture**: modify `submitPublicBooking/entry.ts:274-303` — today the ReminderClient creation block runs **only when `isTest === true`**, so real website leads never enter the rebooking pipeline. The real (non-test) path must create or link a `ReminderClient` record with the customer's details. The existing manual boards (ClientWorkflow/Messages) are kept as an admin fallback.

**Blocked by:** 03 (transport exists)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/06-reminders` → review → merge into `beta`

**Requires:**
- External services: Twilio (ticket 03) — no new service.
- Prerequisites: ticket 03 complete (opt-out suppression depends on it); client records with frequencies set.
- Testing needs: TEST clients with `is_test` (must never be messaged); real test clients with controlled `last_visit_date` values to force "due this week"; a client with a booking already this week (must be suppressed); a STOP-ed client (must be suppressed); a client with an active unresolved conversation (must be suppressed).

**What you get:** Regular clients automatically receive a rebooking nudge at their expected frequency, with a follow-up to non-responders — and never a message to someone who booked, opted out, or needs a human. Real website customers now become client records, so the CRM actually grows.

**How to test:**
1. Set a real test client due this week (via `last_visit_date` + frequency) → Tuesday campaign sends the reminder SMS; `Message` row marked automated.
2. Do not respond → Thursday campaign sends the follow-up.
3. Client books an appointment → no reminder is sent to them that week.
4. STOP-ed / paused / unresolved-conversation / `is_test` clients → never messaged.
5. Submit a real (non-TEST) public booking → a `ReminderClient` record is created/linked with their details.
6. Re-run the campaign immediately → no duplicate messages (idempotent).
7. DST check: run the campaign across a Sydney daylight-saving transition — Tuesday outreach and Thursday follow-up still fire on the correct local days (the codebase's scheduling is Sydney-TZ-safe, e.g. the DST helpers in `base44/shared/bookingWindow.ts:74,107-109`).

**What you see:** In the Clients/Reminders view: due clients get messages on Tuesday, non-responders on Thursday; suppression visibly applies; new website bookings appear as client records; re-runs produce zero duplicates.

- [ ] Server-side due-list computation reuses frequency model
- [ ] Tuesday outreach + Thursday follow-up from config
- [ ] Suppression: booked-this-week / opted-out / paused / unresolved-conversation (conversation_status open|escalated) / test
- [ ] `conversation_status` field added to ReminderClient (schema change)
- [ ] Real (non-test) public bookings create or link a `ReminderClient`
- [ ] Campaign re-run is idempotent
- [ ] Manual boards retained as admin fallback
- [ ] Merged into `beta` after Standards + Spec review
