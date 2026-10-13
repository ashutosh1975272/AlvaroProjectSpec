# 11: Server-side ETA engine [BETA]

**What to build:** Live arrival estimation (DOCX §4E). **(0 — Schema additions, required first:)** `RouteConnection` currently holds only `from/to_appointment_id, day, date, start_time, end_time, duration_minutes, distance_km, efficiency, status` (`base44/entities/RouteConnection.jsonc:31-64`) — add `live_eta`, `drift_minutes`, `recalculated_at` for the live legs the ETA engine writes. (1) An **ETA engine** that, given the barber's latest live position (ticket 10) and the day's remaining `RouteConnection` drive legs, computes the estimated arrival at the next stop via the existing server-side Mapbox matrix (`mapboxTravel.ts`), and **recalculates downstream ETAs** whenever a position update arrives or a stop completes (ticket 05). **Throttling (necessary for the feature to work at all):** ETA recomputes are throttled to at most one per barber per 30 seconds — position updates batch into the next recompute — so a moving barber cannot exhaust the Mapbox Matrix API. (2) **Drift detection**: live ETA minus scheduled time is compared against `ETA_DRIFT_THRESHOLD_MINUTES`; drift at or beyond the threshold flags the route and feeds the notification path (ticket 12). **Drift state machine (prevents message spam while keeping §4E's auto-notify):** notification fires **once per material drift event** — state moves Normal → DriftNotified when drift ≥ threshold, back to Normal when drift recovers below threshold, and a new drift event re-notifies. (3) The **customer state machine**, implemented exactly as the spec lists — **Confirmed → Barber preparing → On the way → Arriving soon → Arrived → Completed** — persisted per stop on `DayRoute.stops[]` (fields added by ticket 05) so the customer-facing status is always current. (4) The Route tab shows, per stop: scheduled time, **live ETA**, and expected drive duration (today only the planned leave-by time exists — `AdminRoute.jsx:42-45`).

**Blocked by:** 05 (stop states exist), 10 (live position exists)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/11-eta-engine` → review → merge into `beta`

**Requires:**
- External services: Mapbox (existing `CUTDELUXEMAP` secret — already used by the scheduling engine; no new service).
- Prerequisites: tickets 05 and 10 complete.
- Testing needs: a finalised test day with known drive legs; scripted position updates (including "barber moved", "barber idle", "barber running late"); a stop-completion mid-route to verify downstream recalculation; a simulated ≥10-minute drift case.

**What you get:** The Route tab and the customer-facing status reflect reality as the day unfolds — the next stop's ETA is live, downstream stops shift when the day shifts, and material delays are detected automatically instead of discovered by the customer.

**How to test:**
1. Start a finalised test day with tracking on → next-stop ETA appears and updates as position moves.
2. Complete a stop mid-route → all downstream ETAs recalculate and persist.
3. Simulate the barber running 10+ minutes behind → drift threshold flags the route (visible status + reason).
4. Verify the customer state machine transitions in the exact spec order (Confirmed → … → Completed) as the day progresses.
5. Reload the Route tab → ETAs and states survived (server-side, not client state).
6. Without tracking on → the tab falls back to planned times and says so clearly (no fake ETAs).

**What you see:** Live ETAs per stop that shift with reality; downstream updates on stop completion; a visible "drifting" status with the minute delta; customer states matching the spec's exact list.

- [ ] RouteConnection schema extended with live_eta/drift_minutes/recalculated_at (evidence: RouteConnection.jsonc:31-64 lacks them today)
- [ ] ETA computed server-side from live position + Mapbox drive legs
- [ ] ETA recompute throttled to ≤1 per 30s per barber (position updates batch into next recompute)
- [ ] Downstream ETAs recalculate on position update and stop completion
- [ ] Drift ≥ `ETA_DRIFT_THRESHOLD_MINUTES` flags the route with reason
- [ ] Drift notifies once per drift event; recovery resets; re-drift re-notifies
- [ ] Customer states implemented exactly: Confirmed → Barber preparing → On the way → Arriving soon → Arrived → Completed
- [ ] Route tab shows live ETA + expected drive duration per stop
- [ ] No fake ETAs when tracking is off (clear planned-time fallback)
- [ ] Merged into `beta` after Standards + Spec review
