# 18: CRM materialization [BETA]

**What to build:** The last CRM gaps (DOCX §4G). **(0 — Schema additions, required first:)** `ReminderClient` currently has contact/notes/favourite/frequency/`last_visit_date` (`ReminderClient.jsonc:89`) but no spend/visit counters and no opt-out or conversation fields — add `total_spent`, `visit_count`, `avg_spend`, `opted_out`/`stop_received_at` (ticket 03), and `conversation_status` (ticket 06) so CRM state persists server-side. (1) **Persisted visit/spend history**: today `clientStatus.js` derives visits, last visit, and lifetime spend client-side by name-matching completed bookings (`clientStatus.js:62-66,143-161` — `completedBookings.some(b => b.client_name.toLowerCase() === client.client_name.toLowerCase())`), which is fragile (name collisions, typos). At completion (server-side, ticket 05's transition), the client record is updated with `total_spent`, `visit_count`, `last_visit_date`, and `avg_spend` as persisted fields, and the name-matching derivation is removed. (2) **Self-service opt-out**: the "STOP" reply captured by ticket 03 (`opted_out` + `stop_received_at`) blocks **all automation** (reminders, ETAs, marketing); an admin can still send manually, and that manual send is audited. (3) **Message history view**: the client drawer and the booking detail show the full message thread (all `Message` rows for the client/booking) with automated-vs-manual labels and delivery status — the DOCX §4A requirement to "keep a message history on the client/booking and log whether the message was automated or manual".

**Blocked by:** 05 (server-side completion transition is the CRM update hook), 06 (suppression context exists)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/18-crm` → review → merge into `beta`

**Requires:**
- External services: none new (uses the messaging pipeline from ticket 03).
- Prerequisites: tickets 05 (server-side completion hook), 03 (STOP capture), 06 (suppression evaluation).
- Testing needs: clients with completed bookings (to verify persisted spend/visits); a STOP-ed client (automation must not touch them; admin manual send must work and be audited); a client with a mixed thread (automated + manual messages) for the history view.

**What you get:** Client profiles that actually reflect the relationship (spend, visits, last visit) without fragile derivation, a working self-service opt-out, and a full auditable message history per client and per booking.

**How to test:**
1. Complete a booking → the client record shows updated `total_spent`, `visit_count`, `last_visit_date` (persisted, not derived — verify by checking the record directly).
2. Reply "STOP" to an automated message → the client's opt-out flag is set; the next scheduled reminder skips them; an admin manual send still goes out and is recorded as manual.
3. Open a client with message history → the thread shows every message with automated/manual labels and delivery status.
4. Open a booking detail → its message thread (request received, confirmation, reminder, receipt) is shown.
5. Verify a STOP-ed client receives zero automation across reminder, ETA, and follow-up campaigns.

**What you see:** Persisted lifetime spend and visit counts on profiles; opt-outs honoured everywhere with manual-send escape hatch; complete message threads with automated/manual labels.

- [ ] Schema fields added to ReminderClient (total_spent/visit_count/avg_spend/opted_out/stop_received_at/conversation_status; last_visit_date already exists at ReminderClient.jsonc:89)
- [ ] `total_spent` / `visit_count` / `last_visit_date` / `avg_spend` persisted at completion (server-side)
- [ ] STOP opt-out stored on the client record and blocks all automation
- [ ] Admin manual send to opted-out clients still possible and audited
- [ ] Message history view on client drawer and booking detail (automated/manual labels + delivery status)
- [ ] Name-matched derivation removed (clientStatus.js:62-66,143-161 — no more client-side guessing)
- [ ] Merged into `beta` after Standards + Spec review
