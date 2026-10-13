# TRACEABILITY — spec DOCX → code → tickets

Every requirement from `Cut_Deluxe_Functional_Specification_Atul.docx` traced to its implementation status with file evidence, then to the ticket that completes it.

**Legend:** ✅ Complete · ⚠️ Partial · ❌ Missing · ➡️ Ticket(s) that complete it.
Evidence is `file:line` from the current codebase. OBS = observed in code.

## §2 — What already works (per doc)

| Doc claim | Status | Evidence |
|---|---|---|
| Public booking (5-step flow) | ✅ | `src/pages/Book.jsx` (RegionStep→ServicesStep→AvailabilityStep→DetailsStep→ReviewStep); `submitPublicBooking/entry.ts` |
| Availability window (not exact time) | ✅ | `BookingRequest.availability_window`; `publicBookingValidators.ts` |
| Coverage validation + service areas | ✅ | `shared/bookingLocation.ts` (`resolveBookingLocation`), `shared/serviceCoverage.ts` |
| Shared booking window (min notice / max horizon) | ✅ | `BookingRules` entity; `shared/bookingWindow.ts` |
| Calendar with provisional/locked/confirmed + manual drag | ✅ | `src/pages/admin/Schedule.jsx`; `applyManualPlacement/entry.ts` |
| Route Flow auto-place | ✅ | `routeFlowAutoPlaceV2/entry.ts` → `shared/routeFlowEngine.ts` |
| Route Flow reshuffle (provisional only) | ✅ | `routeFlowApplyReshuffleV2/entry.ts`; `shared/provisionalPlacement.ts` (`isAutomationAllowed`: only `provisional` is movable) |
| Stage 2.1 fit | ✅ | `shared/routeFlowFit.ts` (`fitUnscheduledRequest`) |
| Timing engine (duration + buffer + travel; no hidden padding) | ✅ | `shared/appointmentTiming.ts` (`occupancyMinutes`) |
| Barber start location for first leg | ✅ | `TeamMember.start_location_*`; `dayRoute.ts` (`START_KEY`) |
| Activity log surfaced | ✅ | `ActivityLog` entity; `shared/routeFlowActivity.ts`; `src/lib/routeFlowActivity.js` |
| Security (protected routes, server-side functions, role restrictions, audited overrides) | ⚠️ | All 36 functions gated (`schedulingAuth.ts`); **but ~10 browser-side direct writes bypass the gate** (see §4J) |
| Test data isolation + cleanup | ✅ | `shared/testData.ts`; `deleteTestData/entry.ts` |

## §4A — Automated messaging & AI assistant

| Requirement | Status | Evidence | Ticket |
|---|---|---|---|
| WhatsApp/SMS primary; email secondary | ❌ | Only outbound sends in repo: staff-invite emails (`inviteStaffMember/entry.ts:56,85`). No provider code, no credentials. | 03 |
| Auto: request received | ❌ | Trigger exists (`submitPublicBooking/entry.ts:268`); no send. | 04 |
| Auto: booking confirmed | ❌ | Trigger exists (`applyManualPlacement/entry.ts:291-295`, `finaliseRoute/entry.ts:46-54`); no send. | 04 |
| Auto: payment/deposit request | ❌ | No deposit subsystem at all. | 08, 09 |
| Auto: reminder | ⚠️ | "Prepare" only (`ClientWorkflow.jsx:169-187,248-253`); clipboard copy (`Messages.jsx:37-39`). | 06 |
| Auto: ETA/on-the-way | ❌ | "Send Arrival" button writes `status:'waitlist'` (`AdminRoute.jsx:196-199`) — corrupts booking status. | 05, 11, 12 |
| Auto: running late | ❌ | No delay/drift detection anywhere. | 11, 12 |
| Auto: reschedule/cancellation | ⚠️ | Admin-side transitions exist (`declineBookingRequest/entry.ts`); no customer notification. | 04 (pattern), 07 |
| Auto: completion/receipt | ❌ | Completion is a browser-side write (`AdminRoute.jsx:190-194`); no receipt. | 05 |
| Auto: rebooking follow-up | ⚠️ | Cadence maths exists (`reminderUtils.js:108-130,287-307`); no send; **real leads never become ReminderClients** (`submitPublicBooking/entry.ts:274-303` creates test clients only). | 06, 18 |
| AI assistant for routine inbound | ❌ | No inbound channel; ChatPanel is a static placeholder (`ChatPanel.jsx:38-49`). Only LLM use: drive-time estimation (`driveTimeService.js:28`). | 07 |
| Assistant understands booking context | ⚠️ | All context fields exist on `BookingRequest.jsonc`; sanitized lookup `checkBookingStatus/entry.ts:47-56`. No assistant consumes them. | 07 |
| New-lead minimum data collection | ⚠️ | Collected on Book.jsx 5-step flow; no conversational collection. | 07 |
| Existing-client recognition | ⚠️ | Name+phone match (`checkBookingStatus/entry.ts:24-37`); 10-min dedup. No inbound path. | 07 |
| Structured change requests (never silently alter protected) | ⚠️ | Admin-side structured edits exist; no customer-initiated flow. | 07 |
| Escalate ambiguous/angry/sensitive/unusual | ❌ | No mechanism, field, or UI. | 07 |
| Never promise unvalidated times | ⚠️ | Validator governs all placement; no AI exists yet, so no guardrail layer. | 07 |
| Message history with automated-or-manual flag | ⚠️ | `Message.jsonc` schema complete (direction/type/delivery_status/manual_or_automated); **zero writers** — only referenced by delete flows. | 02, 03 |

## §4B — Calendar & scheduling

| Requirement | Status | Evidence | Ticket |
|---|---|---|---|
| Base44 calendar remains planning surface | ✅ | `Schedule.jsx` | — |
| Requests become provisional placements inside availability | ✅ | `routeFlowEngine.ts` | — |
| Shared validator governs public booking, automation, manual movement | ✅ | `schedulingValidator.ts` used by all three paths | — |
| Provisional movable; locked/confirmed protected | ✅ | `provisionalPlacement.ts` (`isAutomationAllowed`) | — |
| No overlap / invalid travel / invalid hours / availability violations | ✅ | `schedulingValidator.ts` violation codes | — |
| Admin can edit, manually place, decline unresolved requests | ✅ | `updateBookingRequest`, `applyManualPlacement`, `declineBookingRequest` | — |
| Calendar exposes route status: Draft, Finalising, Finalised, Needs Update | ✅ | `DayRoute.jsonc:36-46`; `dayRouteStatus.js`; `DayRouteBar.jsx` | — |
| Google Calendar sync with source-of-truth policy | ❌ | Zero grep matches repo-wide. | 16 |

## §4C — Route Flow / optimisation engine

| Requirement | Status | Evidence | Ticket |
|---|---|---|---|
| Stage 1 place new request | ✅ | `routeFlowEngine.ts` | — |
| Stage 2 optimise provisional (total drive + idle) | ✅ | `routeFlowOptimiser.ts` (`optimiseDay`) | — |
| Stage 2.1 full reshuffle rather than giving up | ✅ | `routeFlowFit.ts` | — |
| Barber start location first leg; real road travel | ✅ | `mapboxTravel.ts`; `dayRoute.ts` START_KEY. **Caveat:** draft days use client-side LLM estimates (`driveTimeService.js:5-14`, "TO SWAP TO GOOGLE MAPS LATER") | 13 |
| 5-minute scheduling increment | ✅ | `appointmentTiming.ts` (STEP_MINUTES=5) → move to config | 13 |
| Service occupancy = duration + buffer, counted once; travel separate, once | ✅ | `appointmentTiming.ts` (`occupancyMinutes`) | — |
| Finalise Route optimises whole day + creates drive legs | ✅ | `dayRoute.ts` (`finaliseDayRoute`) | — |
| Finalised route that changes → Needs Update; never silently altered | ✅ | `dayRoute.ts` (`markRouteNeedsUpdate`, fingerprint) | — |
| Every automation action auditable in Route Flow Activity | ✅ | `routeFlowActivity.ts`; ActivityLog | — |

**Trigger mechanism (verified):** auto-place runs as an entity-trigger Workflow on `BookingRequest` create (`base44/workflows/RouteFlowAutoPlace.jsonc:30-39` → invokes `routeFlowAutoPlaceV2`); the reshuffle Workflow triggers on create/update/delete of scheduling-relevant fields. **"New request after finalisation" (DOCX §4C question):** the create trigger fires; if the new request is placed, the day's membership changes, so `routeFingerprint` (id + assigned_time + schedule_status + duration + location — `dayRoute.ts:177-191`) no longer matches, and the finalised DayRoute reconciles to **needs_update** on next read (`getDayRouteState`) — a finalised route is never silently altered (§4C).

| Requirement | Status | Evidence | Ticket |
|---|---|---|---|
| Today's finalised route in order (start, appt, drive, appt…) | ⚠️ | Finalised days render via `routeTimeline.js` + `DayRoute.stops[]`; non-finalised days show a draft estimate | 05 |
| Next stop, address, scheduled time, ETA, expected drive duration | ⚠️ | Next stop/address/time/drive-duration exist (`AdminRoute.jsx:409-456`); **ETA missing** — only planned leave-by (`AdminRoute.jsx:42-45`) | 11 |
| Actions: Navigate, Call, Message, Arrived, Start, Complete | ⚠️ | Navigate ✅ (`AdminRoute.jsx:424-431`); Complete ✅ (`:190-194`); Arrived ⚠️ corrupts to `waitlist` (`:196-199`); Call ❌; Message ❌; Start ❌ | 05 |
| Step completion updates progress + recalculates downstream ETAs | ⚠️ | Progress is ephemeral client state (`AdminRoute.jsx:147-149`); downstream ETA recalc missing | 11 |
| Clearly show stale / needs re-optimisation | ✅ | `RouteStatusBar.jsx`; `getDayRouteState` | — |
| Emergency/manual change without corrupting the route | ⚠️ | Manual changes go through `applyManualPlacement` 409 guards; no in-Route-tab emergency change (skip/reorder/redirect) | 05, 15 |

## §4E — Live tracking & ETA

| Requirement | Status | Evidence | Ticket |
|---|---|---|---|
| Barber enables live location only during working route (explicit, privacy-aware) | ❌ | No geolocation in execution paths (only `ServiceCoverage.jsx:139-140` for map picking); no toggle, no entity field | 10 |
| Current location + route data → arrival estimate | ❌ | No location capture, no ETA engine | 11 |
| Customer receives useful ETA/status without continuous location exposure | ❌ | Message entity ready; no writer | 12 |
| Customer states: Confirmed, Barber preparing, On the way, Arriving soon, Arrived, Completed | ❌ | No journey states on any entity — `DayRoute.stops[]` holds only `booking_request_id/order/start_label/start_decimal/end_label/drive_minutes` (`DayRoute.jsonc:118-146`) and `RouteConnection` has no live-ETA fields (`RouteConnection.jsonc:31-64`); both schema extensions land in tickets 05 and 11 | 05, 11 |
| Material ETA drift auto-notifies next affected customer | ❌ | No ETA, no drift detection | 11, 12 |
| Store only needed location data with retention rules | ❌ | No location stored at all; no retention policy | 10 |

## §4F — Payments

| Requirement | Status | Evidence | Ticket |
|---|---|---|---|
| Payment state attached to booking | ⚠️ | `BookingRequest.jsonc:276-292` — binary Unpaid/Paid, "display only — Square owns the truth"; no amounts | 09 |
| Configurable deposits for higher-risk bookings (not forced on everyone) | ❌ | No deposit fields/rules anywhere; risk inputs exist server-side at submission (`submitPublicBooking/entry.ts:200-253`) | 08 |
| Full payment, deposit+balance, refund, partial refund states | ❌ | Enum is Unpaid/Paid only; zero `refund` code matches repo-wide | 09 |
| Automatic payment link/request when rule requires | ❌ | No payment-link creation, no Square API calls | 09 |
| Payment marked received via webhook/server confirmation | ❌ (inverted) | No webhook function exists; only writer is the client-side button (`AdminPayments.jsx:26-34`) — the exact anti-pattern §8 forbids | 09, 15 |
| Post-completion balance collection / admin-approved offline payment | ⚠️ | Manual admin mark exists (Cash/Transfer/Card); no post-completion trigger | 05, 09 |
| Receipt sent | ❌ | Zero receipt code; no messaging layer to send through | 03, 05 |
| Revenue/reporting updated | ⚠️ | `AdminPayments.jsx:38-40` computes totals client-side; no persisted revenue entity | 17 |
| Reuse Square rather than invent a ledger | ⚠️ | Square IDs exist as data shapes (`square_item_id`, `square_payment_id`); no OAuth, no credentials, no API calls | 09 |

## §4G — Client CRM & regular-client reminders

| Requirement | Status | Evidence | Ticket |
|---|---|---|---|
| Single client profile (contact, addresses, visit history, notes, favourite, service history, spend, booking/cancellation history) | ⚠️ | `ReminderClient.jsonc` holds contact/notes/favourite; visit history + spend are **derived client-side by name-matching** (`clientStatus.js:62-66,143-161`), not persisted | 18 |
| Preferred frequency (weekly, fortnightly, 3-weekly, monthly, custom) | ✅ | `ReminderClient.frequency` + `custom_frequency_days`; `reminderUtils.js:49` | — |
| Automatically identify clients due + send rebooking reminders | ⚠️ | Due-date maths exists (`reminderUtils.js:108-130`); auto-stamped on completion (`Schedule.jsx:548-573`); no server-side scheduler, no send | 06 |
| Cadence: early-week outreach + later-week follow-up to non-responders | ⚠️ | `followUpDueDate = sentDate + 3 days` (`reminderUtils.js:287-292`); admin presses "Weekly Reminders" manually | 06 |
| Never message booked / opted-out / unresolved-conversation clients | ⚠️ | Booked: enforced at completion only, not at booking creation; **no opt-out field exists** (only admin `is_paused`); unresolved-conversation gate missing (ChatPanel stub) | 06, 18 |
| Clear Clients/Reminders view (due, messaged, replied, booked, follow-up needed) | ⚠️ | 4-stage kanban exists (`ClientWorkflow.jsx:111-131`); "replied"/"follow-up needed" are implicit | 06, 18 |

## §4H — Booking protection & reliability

| Requirement | Status | Evidence | Ticket |
|---|---|---|---|
| Protection against duplicates/spam/malicious repeated bookings | ⚠️ | 10-min dedup + submission_token (`submitPublicBooking/entry.ts:26-28,54-62,88,118-123`); nothing beyond | 14 |
| Rate-limit by sensible identifiers without blocking families/groups | ❌ | No rate/throttle code anywhere. **Phone-key mismatch:** `normPhone`→`"412345678"` (`submitPublicBooking/entry.ts:30-36`) vs `normalizeAuPhone`→`"0412345678"` (`checkBookingStatus/entry.ts:71-78`) — endpoints cannot recognise the same customer | 14 |
| Active-booking limits by customer/phone | ❌ | Only business-wide `max_bookings_per_day` (`groupPricing.js:42-46`) | 14 |
| Suspicious requests → Needs Review | ⚠️ | `placement_status: needs_review` exists but is written only for placement failures (`routeFlowEngine.ts:79,83,137`), never for fraud/spam | 14 |
| Cancellation/no-show history informs deposits | ❌ | `no_show` status exists; nothing counts or evaluates history | 14, 08 |
| Background jobs idempotent (no duplicate bookings/messages/payments on retry) | ⚠️ | Route Flow idempotent (`routeFlowEngine.ts:106-117`); submission dedup; decline idempotent. **No queue/worker exists** — message/payment retry-safety unbuilt | 02 |

## §4I — Admin & operations

| Requirement | Status | Evidence | Ticket |
|---|---|---|---|
| Booking Requests: Edit, Add to Calendar, Decline with reasons/states | ✅ | `Requests.jsx`; `updateBookingRequest`; `declineBookingRequest` | — |
| Route Flow Activity: placements, reshuffles, fits, unscheduled, finalisation, retries, overrides | ✅ | `routeFlowActivity.js`; ActivityLog + PlacementValidationLog | — |
| Team Member profile (hours, special hours, start location, permissions) | ✅ | `TeamMember.jsonc`; `TeamMembers.jsx` | — |
| Booking Rules (min notice, horizon, group discounts, capacity, deposit rules) | ⚠️ | min notice/horizon/group discounts exist; **deposit rules missing** | 08 |
| Service Coverage: editable areas and tiers | ✅ | `ServiceAreas.jsx`; `ServiceCoverage.jsx` | — |
| Reports: bookings, clients serviced, revenue, avg price, revenue/hour, drive time, idle time, new clients, route efficiency | ⚠️ | Bookings/outcomes/new-clients exist (`Reports.jsx`); revenue & avg price missing from Reports (only Dashboard/AdminRoute per-day); drive/idle only per-day; efficiency per-leg only, no aggregate | 17 |

## §4J — Security, data, failure handling

| Requirement | Status | Evidence | Ticket |
|---|---|---|---|
| Authoritative transitions server-side | ⚠️ | All 36 functions gated; **~10 browser-side direct writes bypass the gate**: `Schedule.jsx:553,581,597,489,499,396`, `AdminRoute.jsx:192,198`, `Requests.jsx:72`, `AdminPayments.jsx:27,32`, `AppointmentDetail.jsx:60`, `Clients.jsx:260` | 15 |
| Role-based permissions | ⚠️ | `schedulingAuth.ts` three tiers; entity RLS is coarse (admin OR user on most entities) | 15 |
| Public endpoints expose minimum necessary data | ✅ | `checkBookingStatus` (name+phone, generic no-match, sanitized fields); `publicBookingData`/`publicHomepageData` | — |
| Every manual override explicit and audited | ✅ | Override path with recorded reason (`applyManualPlacement/entry.ts`; `placementAudit.ts`) | — |
| No silent failure (→ Processing/Unscheduled/Needs Review with human-readable reason) | ⚠️ | True for placement (`routeFlowEngine.ts:79-83,137,259-293`); **silent catch in completion** (`Schedule.jsx:571,648`) | 15 |
| Version/concurrency checks before reshuffles | ⚠️ | `applyManualPlacement` (`expected_updated_date`), routeFlowEngine, dayRoute all have guards; most other write paths have none | 15 |
| Production/test-data separation | ✅ | `is_test` flags; `testData.ts`; excluded from Reports/Dashboard/AdminRoute/ClientWorkflow; admin-only purge | — |

**Test-data exclusion scope (decision, per DOCX §4J "so QA does not pollute real analytics or reminders"):**

| Feature | is_test included? | Basis |
|---|---|---|
| Public booking flow (QA) | Yes | QA must exercise the real flow |
| Route Flow auto-place / reshuffle | Yes (flagged) | Automation must run on test bookings; events visible in Route Flow Activity marked `is_test` |
| Messaging / outbox sends | **No** | Ticket 02 acceptance |
| Reminder campaigns | **No** | Ticket 06 acceptance |
| ETA notifications | **No** | Ticket 12 suppression |
| Reports / analytics / revenue | **No** | Existing `testData.ts` query-level filters |
| Square charges | **No** | Sandbox test customers only (ticket 09) |
| Google Calendar sync | **No** | Ticket 16 acceptance |
| AI assistant context | **No** (decision — inference from §4J intent, not explicit DOCX text) | Assistant must not present test bookings as real client history |
| DayRoute drive/idle metrics | Excluded from business metrics (decision) | Test days must not skew efficiency/revenue-per-hour reports |

## §5, §6, §8, §10, §12 — process requirements

| Requirement | Status | Ticket |
|---|---|---|
| §5 Finalise Route workflow (requests → provisional layout → review → finalise → drive legs → execution-ready → needs-update → re-optimise) | ✅ (workflow exists) | 05 (execution-ready completion) |
| §6 messaging matrix — 10 rows | 0/10 deliver; 2 prepare-only | 04, 06, 07, 08, 09, 12, 05 |
| §8 ten unbreakable automation rules | Mostly exist for scheduling; missing for messaging/payments (no engine) | global acceptance criteria on all tickets |
| §10 MVP acceptance criteria | Scheduling core criteria met; messaging/payment/ETA/background criteria unmet | all tickets (exit criteria) |
| §12 delivery phases 1–6 | Phase 1 ~85% (defects in 13/05/15); Phases 2–6 unbuilt | ticket order follows phases |

## Ticket index

| Ticket | Completes |
|---|---|
| 01 Scaffolding | spec/, GLOSSARY, ADRs, AGENTS.md, `.env.example` contract |
| 02 Outbox + sweep | §4H idempotent background jobs; §7 background job layer |
| 03 SMS transport + webhook | §4A channel; §7 WhatsApp/SMS provider |
| 04 Request-received + confirmed | §6 rows 1, 3 |
| 05 Stop execution states + receipt | §4D actions & states; §6 row 9; §4J server-side completion |
| 06 Reminder campaign engine | §4G auto-identification + cadence; §6 rows 5, 10; real-lead capture |
| 07 AI assistant | §4A assistant; §6 row 8 |
| 08 Deposit rules engine | §4F deposits; §4H no-show history |
| 09 Square links + webhook + refunds | §4F links/webhook/refunds/receipts; §8 provider-confirmed payments |
| 10 Barber live-position capture | §4E explicit live location + retention |
| 11 Server-side ETA engine | §4E arrival estimate + customer states + drift |
| 12 Customer ETA notifications | §4E customer updates; §6 rows 6, 7 |
| 13 Config centralization + hardening | §4C thresholds configurable; LLM→Mapbox swap; >24-stop chunking; V1 retirement |
| 14 Booking protection | §4H rate limits, active limits, suspicious→Needs Review, phone-key fix |
| 15 Server-side migration + RLS | §4J authority, RBAC, no silent failure, version checks |
| 16 Google Calendar integration | §4B sync with source-of-truth policy (Phase 5) |
| 17 Reports completion | §4I full report metrics |
| 18 CRM materialization | §4G persisted history, opt-out, message history view |
