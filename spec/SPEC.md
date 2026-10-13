# SPEC — Cut Deluxe automation layer

*Synthesised from `Cut_Deluxe_Functional_Specification_Atul.docx` (October 2026) — the source of truth — using the `to-spec` process. The DOCX governs; this file is the working spec for implementation.*

## Problem Statement

A mobile barber business runs on manual admin work: rearranging the calendar, replying to routine messages, calculating travel time, chasing confirmations, and reconciling payments by hand. The Base44 app already books customers, auto-places requests into the day using real driving time, optimises and finalises the route, and audits every automated decision — but the automation layer around it is missing: nothing is ever sent to customers (the only outbound sends in the codebase are staff-invite emails), payments are display-only, the Route tab cannot run a live day (no live location, no ETA, and "Send Arrival" corrupts booking status), reminders are never delivered, and there is no background engine, so nothing runs once the browser closes.

## Solution

Build the automation layer on top of the existing scheduling core, in the spec's own delivery order (§12):

1. **Finish the scheduling core** — server-side stop execution states (Arrived/Start/Complete), Mapbox drive times everywhere, chunked finalisation for large days, all thresholds in config.
2. **Automated messaging** — provider adapter (Twilio SMS first, WhatsApp fast-follow), idempotent outbox + 5-minute sweep, the ten §6 matrix messages, AI assistant for routine inbound conversations with escalation.
3. **Payments** — Square OAuth, deposit rules for higher-risk bookings, payment links, webhook-confirmed payment state, refunds, receipts.
4. **Live tracking & ETA** — explicit barber live-position capture (working-route only, 24h retention), server-side ETA recalculation, drift detection, customer ETA notifications.
5. **Calendar integrations** — Google Calendar mirror of confirmed appointments + external busy blocks, with Base44 as source of truth.
6. **Hardening** — rate limits, active-booking limits, suspicious→Needs Review, migration of all browser-side state writes to gated server functions, RLS tightening, retirement of V1 stubs.

## User Stories

1. As a new customer, I want to request a booking online without speaking to the barber, so that I can book at any time.
2. As a new customer, I want to give a real availability window rather than an exact time, so that the barber can fit me in efficiently.
3. As a new customer, I want an automatic "request received" message, so that I know my booking was submitted.
4. As a customer, I want an automatic confirmation with the agreed date, time/window, service, and address, so that I know exactly when the barber is coming.
5. As a customer with a deposit-required booking, I want an automatic payment link, so that I can pay before the appointment.
6. As a customer, I want an automatic reminder before my appointment, so that I don't forget.
7. As a customer, I want an automatic "barber is on the way" message with a live ETA, so that I know when to expect them.
8. As a customer, I want an "arriving soon" message when the barber is close, so that I can be ready.
9. As a customer, I want an automatic "running late" message with a revised ETA, so that I can adjust my day.
10. As a customer, I want an automatic completion message with my receipt, so that I have proof of payment and service.
11. As a regular client, I want an automatic rebooking reminder at my expected frequency, so that I stay on schedule.
12. As a regular client, I want a follow-up message if I haven't booked after the first reminder, so that I get a nudge.
13. As a customer, I want to reply to a message and have routine questions answered automatically from my booking context, so that I don't have to wait for the barber.
14. As a customer, I want to request a change by message and have it turned into a structured request, so that my confirmed appointment is never silently altered.
15. As a customer, I want angry, ambiguous, or sensitive conversations escalated to a human, so that I get a proper response.
16. As a customer, I want to opt out of automated messages (e.g. by replying STOP), so that I am never messaged against my wishes.
17. As a customer, I want my messages and their delivery status kept on my record, so that there is a clear history.
18. As a barber, I want the Route tab to show today's finalised route in order (start, appointment, drive, appointment…), so that I can run the day.
19. As a barber, I want to see the next stop, address, scheduled time, ETA, and expected drive duration, so that I know where to go next.
20. As a barber, I want Navigate, Call, Message, Arrived, Start, and Complete actions on each stop, so that I can execute the day from one screen.
21. As a barber, I want downstream ETAs recalculated when I complete a stop or run late, so that the day stays accurate.
22. As a barber, I want to see clearly when the route is stale or needs re-optimisation, so that I never work from an outdated plan.
23. As a barber, I want to enable live location only during the working route, so that my location is not tracked outside work.
24. As a barber, I want emergency/manual changes to be possible without corrupting the rest of the route, so that the day survives surprises.
25. As an admin, I want to review a draft/provisional day in the calendar, so that I can edit, manually place, decline, lock, or confirm requests.
26. As an admin, I want to press Finalise Route once and get an optimised, validated day with drive legs, so that the day is execution-ready.
27. As an admin, I want every automated decision visible in Route Flow Activity with a status and reason, so that automation is auditable.
28. As an admin, I want impossible requests to remain visible as Unscheduled or Needs Review with a human-readable reason, so that nothing disappears.
29. As an admin, I want deposit rules applied only to higher-risk bookings (new clients, after-hours, groups, poor cancellation history), so that regular clients are not inconvenienced.
30. As an admin, I want payment state confirmed by Square webhooks rather than a front-end success screen, so that revenue data is trustworthy.
31. As an admin, I want to issue full and partial refunds with approval recorded, so that exceptions are handled and audited.
32. As an admin, I want reports covering bookings, clients serviced, revenue, average price, revenue/hour, drive time, idle time, new clients, and route efficiency, so that I can run the business.
33. As an admin, I want rate limits and active-booking limits that never block legitimate families or groups, so that spam is stopped without hurting real customers.
34. As an admin, I want suspicious requests held as Needs Review instead of auto-scheduled, so that I can check them first.
35. As an admin, I want every manual override explicit and audited, so that accountability is preserved.
36. As an admin, I want test customers and bookings fully separated from real analytics and reminders, so that QA never pollutes real data.
37. As the business owner, I want background workflows (route flow, messages, reminders, webhooks, ETA events) to keep running with the app closed, so that automation is reliable.
38. As the business owner, I want retries to be idempotent — never creating duplicate bookings, messages, or payments — so that automation is safe.
39. As the business owner, I want Google Calendar to mirror confirmed appointments and respect external busy blocks, with Base44 as the single source of truth, so that there is no double booking.
40. As the business owner, I want all configurable times, thresholds, and limits in one commented config file, so that I can tune the system without code changes.

## Implementation Decisions

- **Architecture stays in Base44** (ADR 0003): background jobs are an idempotent outbox entity processed by a 5-minute scheduled Workflow (Base44's minimum scheduled-trigger interval). No separate queue service this cycle.
- **Messaging** (ADR 0002): channel-agnostic provider adapter; Twilio for SMS first, WhatsApp Cloud API fast-follow, email secondary. Inbound webhooks land on function HTTP endpoints with HMAC signature verification. Every send writes a Message entity row.
- **Payments** (ADR 0004): Square is authoritative; the app mirrors payment state from Square webhooks and never invents a ledger. Deposits are evaluated server-side at submission from configurable risk classes.
- **Source of truth** (ADR 0001): Base44 calendar/DayRoute is the single source of truth; Google Calendar is a mirror plus a busy-block source (ADR 0005).
- **Execution state**: per-stop Arrived/Start/Completed timestamps are stored on `DayRoute.stops[]`, replacing the current "Send Arrival → waitlist" write. The customer state machine is exactly the spec §4E list: Confirmed → Barber preparing → On the way → Arriving soon → Arrived → Completed.
- **Config centralization**: every time, threshold, and limit is read from the `.env.example` contract (Base44 Secrets for backend, `VITE_`-prefixed `.env.local` for frontend). No hardcoded values remain in engine code.
- **Concurrency**: existing optimistic guards are kept (version re-reads, 409 SCHEDULE_CHANGED, 3-attempt retries with conflict settle); all new write paths follow the same pattern.
- **Authority**: all state transitions move to gated server functions; browser-side entity writes are removed (spec §4J).
- **Test data**: `is_test` separation is preserved and extended to messages, outbox, and live position.
- **Privacy**: live position is retained 24 hours maximum and never exposed as history (spec §4E/§11).
- **AI assistant**: built on Base44's built-in `invokeLLM`; autonomous only for routine intents (status, hours, location); any change to a locked/confirmed booking becomes a structured change request; ambiguous/angry/sensitive/unusual conversations are escalated; the assistant can never promise a time the shared validator has not produced.

## Testing Decisions

- **What makes a good test:** tests assert external behaviour (what the customer/admin sees, what is sent, what state results) — never implementation details. Tests of the scheduling engines assert validator verdicts and resulting schedules, not internal search steps.
- **Highest seam first:** the shared validator (`base44/shared/schedulingValidator.ts` — `evaluatePlacement`/`validatePlacement`) is the seam for all scheduling tests; existing prior art is the read-only `routeFlowDiagnostic` and `suggestPlacementSlots` functions, which run the real engine path without writes.
- **Idempotency tests:** replay the same key twice (submission token, message key, webhook event id) and assert exactly one effect.
- **Webhook tests:** valid signature → processed; invalid/forged signature → 401; replayed event → no duplicate effect.
- **Engine tests:** candidate placement scenarios (fits directly / needs Stage 2.1 / impossible) assert the resulting schedule, protected-appointment immobility, and waste-minute metrics.
- **Messaging tests:** quiet-hours suppression, opt-out suppression, test-data exclusion, delivery-status lifecycle, retry/backoff, dead-letter after max attempts.
- **ETA tests:** position updates recalculate downstream ETAs; drift ≥ threshold triggers the next-customer notification path.
- **No test may** send real messages: all provider calls go through a test-mode flag or test credentials.

## Out of Scope

Spec §11 (deliberately NOT required for the first functional release):

- Multi-barber route optimisation across several vans at once.
- Complex favourite/revenue/value weighting inside Route Flow.
- Automatic displacement of already-scheduled provisional clients purely for commercial gain.
- Full franchise/territory management.
- Advanced predictive demand modelling.
- Permanent real-time location history.
- A custom payment ledger while Square remains authoritative.
- AI making unrestricted scheduling decisions outside the shared validator.

## Further Notes

- The delivery order follows spec §12: scheduling core → messaging → payments → live tracking/ETA → calendar integrations → hardening.
- Ticket numbering (`spec/tickets/01…18`) follows that order; each ticket declares its blocking edges.
- Every ticket is worked on `feature/NN-slug`, reviewed on Standards + Spec axes against this spec and `TRACEABILITY.md`, merged into `beta`, and `beta` → `main` only on senior approval.
- Global acceptance criteria (spec §8) apply to every ticket: availability is a hard constraint; locked/confirmed protected; no unvalidated times; service+buffer once, travel once; no silent drops; idempotent retries; provider-confirmed payments; finalised routes never silently change; every automation has a visible status/reason.
- The doc's §10 MVP acceptance criteria are the beta exit criteria.
