# 05: Stop execution states + completion receipt [BETA]

**What to build:** Server-side execution of a day's stops (DOCX §4D, §6 row 9, §4J). **(0 — Schema migration, required first:)** `DayRoute.stops[]` currently holds only `booking_request_id, order, start_label, start_decimal, end_label, drive_minutes` (`base44/entities/DayRoute.jsonc:118-146`) — extend each stop with the execution fields the DOCX §4E machine needs: `state` (exact spec list: Confirmed → Barber preparing → On the way → Arriving soon → Arrived → Completed), `customer_state` (the customer-facing state for this stop), `arrived_at`, `started_at`, `completed_at`, `live_eta`, `drift_minutes`. Existing finalised days migrate cleanly (new fields default to the planned state). Tickets 11 and 12 read and write these fields. **(1)** A server function `advanceRouteStop` that records per-stop execution — **Arrived / Start service / Complete** with timestamps — **replacing the current "Send Arrival" write that corrupts booking status to `waitlist`** (`AdminRoute.jsx:196-199`). (2) The missing Route tab actions: **Call** (`tel:`), **Message** (opens a send through the messaging pipeline), **Start** — alongside the existing Navigate and Complete. (3) Completion becomes a **server-side transition** (it is currently a browser-side entity write, which §4J forbids): on Complete, the booking's outcome is recorded, revenue is captured server-side, and the completion/receipt message is enqueued (matrix row 9: thank-you + receipt with payment + service completion). All writes pass the authority gate and a version check (`expected_updated_date` → 409 `SCHEDULE_CHANGED` on conflict). The completion transition is also the hook ticket 18 uses to persist client spend/visit history (CRM materialization) — the completion function records the outcome that ticket 18 consumes.

**Blocked by:** 03 (receipt is sent through the transport), 04 (template engine renders the approved receipt copy)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/05-stop-states` → review → merge into `beta`

**Requires:**
- External services: Twilio (ticket 03) for the receipt message; Google Maps links already used for Navigate (no new service).
- Prerequisites: ticket 03 complete; a finalised day with drive legs to test against.
- Testing needs: a finalised TEST day (test barber + TEST customers) so completion can be exercised end-to-end without touching real bookings; a real test phone to receive the receipt.

**What you get:** The barber can run a stop from the Route tab — arrive, start, complete — with every transition recorded server-side, and the customer automatically receives a completion message with their receipt. This also removes a live data-corruption bug (Send Arrival → `waitlist`).

**How to test:**
1. Open a finalised test day in the Route tab; press **Arrived** → stop state persisted, booking status unchanged (never `waitlist`).
2. Press **Start** → service start recorded.
3. Press **Complete** → completion recorded server-side; customer receives the receipt message; revenue visible on the booking.
4. Reload the page / switch devices → states survived (they are server-side, not client state).
5. Attempt to complete a stop with a stale version → 409 handled with a human-readable conflict message (no silent failure).
6. Verify Call/Message actions open the phone dialer and the send flow.

**What you see:** Route tab shows real per-stop progress that survives reload; the customer's phone receives the receipt; the booking's status history is clean (no `waitlist` corruption); every transition has an audit row naming the actor.

- [ ] DayRoute.stops[] schema extended with state/customer_state/arrived_at/started_at/completed_at/live_eta/drift_minutes (evidence: DayRoute.jsonc:118-146 lacks them today)
- [ ] Existing finalised days migrate cleanly (new fields default to planned state)
- [ ] Arrived/Start/Complete persist server-side on `DayRoute.stops[]`
- [ ] The `waitlist` corruption is removed (no booking-status misuse)
- [ ] Call/Message/Start actions present on the Route tab
- [ ] Completion is a gated server function with version check
- [ ] Completion enqueues the receipt message (matrix row 9)
- [ ] Conflicts return human-readable errors (no silent failure)
- [ ] Merged into `beta` after Standards + Spec review
