# 12: Customer ETA notifications [BETA]

**What to build:** The customer-facing live-route messages (DOCX §4E, §6 rows 6 and 7). (1) **On the way** — when the barber starts the drive to a stop, the customer receives "the barber is on the way" with the **live ETA** (not a static time). (2) **Arriving soon** — at `ETA_ARRIVING_SOON_MINUTES` (15) before the ETA, the customer receives the arriving-soon message. (3) **Running late** — when drift reaches the threshold, the affected customer receives a revised-ETA message ("running late, now arriving around …"), within the late-notification policy (no approval needed within policy). (4) **Drift propagation** — a material delay recalculates and notifies the **next affected** customer, not just the current one. **"Next affected" defined (DOCX §4E):** the next stop whose scheduled time shifts by ≥ `ETA_DRIFT_THRESHOLD_MINUTES` as a result of the delay — not merely the next stop in sequence; if no downstream stop shifts beyond the threshold, no notification is sent. (5) All messages flow through the outbox (idempotent, retried, quiet-hours-respecting, opt-out-aware) and are recorded as automated `Message` rows; the customer's current state (ticket 11) is included so messages and status never disagree.

**Blocked by:** 03 (transport), 11 (ETA engine)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/12-eta-messages` → review → merge into `beta`

**Requires:**
- External services: Twilio (ticket 03); Mapbox via ticket 11 — no new service.
- Prerequisites: tickets 03 and 11 complete.
- Testing needs: a finalised test day with 2+ stops and two test phones (current + next customer); scripted position scenarios: on-time start, arriving-soon window, ≥10-minute delay (to verify the *next* customer is notified), and a case where the customer has opted out (must not be messaged).

**What you get:** Customers get honest, live arrival information automatically — the "where is my barber" question disappears — and delays are communicated proactively to everyone affected, exactly as DOCX §4E and the §6 matrix promise.

**How to test:**
1. Barber starts driving to stop 1 → customer 1 receives "on the way" with a live ETA.
2. ETA enters the 15-minute window → customer 1 receives "arriving soon".
3. Simulate a 10+ minute delay → customer 1 receives "running late" with the revised ETA, **and** customer 2 (next affected — the next stop whose schedule shifts by ≥ the threshold) is notified of the shifted window.
3b. Simulate a delay where the next stop's shift stays below the threshold → **no** notification to that customer (only genuinely affected customers are messaged).
4. Opted-out customer → no ETA messages (suppression respected).
5. Outside quiet hours → sends are held until the window opens (or suppressed per policy).
6. Re-trigger the same notification (idempotency check) → no duplicate message.

**What you see:** Both test phones receive the right message at the right moment with live ETAs; the next customer is proactively notified on delay; opt-outs and quiet hours hold; Message history shows the automated rows tied to the booking.

- [ ] "On the way" with live ETA sent when the drive starts
- [ ] "Arriving soon" sent at the configured lead time
- [ ] "Running late" with revised ETA sent at drift threshold
- [ ] Next affected customer notified on material drift
- [ ] Quiet hours + opt-out suppression respected
- [ ] All sends idempotent and recorded as automated
- [ ] Merged into `beta` after Standards + Spec review
