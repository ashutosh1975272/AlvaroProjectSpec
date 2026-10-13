# Spec_summary.md — Cut Deluxe end-to-end summary

**Project:** Cut Deluxe — mobile-barber operating system on Base44
**Source of truth:** `Cut_Deluxe_Functional_Specification_Atul.docx` (October 2026)
**Workflow:** all work on `beta`; each ticket on `feature/NN-slug` → review (Standards + Spec axes) → merge into `beta`; `beta` → `main` only on senior approval.
**Config:** every time/threshold/limit lives in `.env.example` (documented contract) — backend values in Base44 Secrets, frontend values in `.env.local` with `VITE_` prefix. No hardcoded values in engine code.

## 1. Status at a glance

| # | Area (doc section) | Status | ~Done | Completing ticket(s) |
|---|---|---|---|---|
| 1 | Public booking flow (§3.1) | ✅ Complete | 95% | — |
| 2 | Scheduling engine: auto-place / reshuffle / Stage 2.1 fit (§4C) | ✅ Complete | 95% | 13 (thresholds to config, chunking) |
| 3 | Calendar + Finalise Route + drive legs (§5) | ✅ Complete | 95% | — |
| 4 | Route Flow Activity audit (§4I) | ✅ Complete | 90% | — |
| 5 | Route tab / day execution (§4D) | ⚠️ Partial | 40% | 05 (states + Call/Message/Start), 11 (ETA) |
| 6 | Security / server-side authority (§4J) | ⚠️ Partial | 70% | 15 (browser-write migration, RLS) |
| 7 | Reports (§4I) | ⚠️ Partial | 55% | 17 |
| 8 | CRM & reminders (§4G) | ⚠️ Partial | 35% | 06 (campaign), 18 (history/opt-out) |
| 9 | Booking protection (§4H) | ⚠️ Partial | 25% | 14 |
| 10 | Payments (§4F) | ❌ Missing | 10% | 08 (deposits), 09 (Square/links/webhook/refunds) |
| 11 | Messaging automation (§4A, §6) | ❌ Missing | 0% | 02 (outbox), 03 (transport), 04, 06, 12 |
| 12 | AI assistant (§4A) | ❌ Missing | 0% | 07 |
| 13 | Live tracking & ETA (§4E) | ❌ Missing | 0% | 10, 11, 12 |
| 14 | Google Calendar sync (§4B) | ❌ Missing | 0% | 16 |

## 2. What the doc mandates → where it lands (coverage map)

| Doc element | Ticket(s) |
|---|---|
| §6 matrix row 1 (request received) | 04 |
| §6 matrix row 2 (provisional holding message — "usually no") | Not sent in MVP (per doc) |
| §6 matrix row 3 (confirmed) | 04 |
| §6 matrix row 4 (deposit required) | 08 + 09 |
| §6 matrix row 5 (reminder) | 06 |
| §6 matrix row 6 (on the way) | 12 |
| §6 matrix row 7 (running late) | 12 |
| §6 matrix row 8 (customer change request) | 07 |
| §6 matrix row 9 (completed / receipt) | 05 |
| §6 matrix row 10 (due for rebooking) | 06 |
| §4D route actions (Navigate/Call/Message/Arrived/Start/Complete) | 05 (+12 for messages) |
| §4E customer states (exact list: Confirmed → Barber preparing → On the way → Arriving soon → Arrived → Completed) | 11 (implemented verbatim) |
| §10 MVP acceptance criteria | mapped into every ticket's acceptance criteria; they are the beta exit criteria |
| §12 delivery phases 1→6 | ticket numbering order |
| §8 ten unbreakable rules | global acceptance criteria on all 18 tickets |
| §11 NOT-required list | out-of-scope in `spec/SPEC.md` |

## 3. Decisions register (doc is silent — chosen values, none contradict the doc)

| Decision | Chosen value | Ticket |
|---|---|---|
| Messaging provider | Twilio — SMS first, WhatsApp fast-follow, email secondary | 03 |
| Consent / opt-out | Consent tick at booking; "STOP" reply opts out of all automation | 06, 18 |
| Quiet hours | No automated sends outside 08:00–20:00 Sydney | 03 |
| AI provider / autonomy | Base44 built-in `invokeLLM`; autonomous for routine intents only; escalates angry/ambiguous/sensitive/unusual; never promises unvalidated times | 07 |
| Escalation | Escalated thread → SMS to barber + "Escalated" queue; 2-business-hour SLA | 07 |
| Deposit rules | $20 new client · $30 after-hours · 20% groups ≥4 · deposit after 2 no-shows · link expires 48h | 08 |
| Rate / active limits | 10 requests/hour per phone/email · 3 concurrent unconfirmed bookings · family-aware | 14 |
| Reminder cadence | Tuesday outreach, Thursday follow-up, per client frequency (7/14/21/30 days) | 06 |
| Live tracking | Phone browser (PWA), explicit toggle, working-route only, 24h retention | 10 |
| ETA drift | ≥10 min = "material" → notify next customer; "arriving soon" at 15 min out | 11, 12 |
| Execution state | Per-stop Arrived/Start/Completed on `DayRoute.stops[]` | 05 |
| Architecture | Stay in Base44: idempotent outbox + 5-minute Workflow sweep (Base44's minimum scheduled-trigger interval) | 02 |
| Google Calendar | Phase 5; Base44 = source of truth; mirror confirmed out + busy blocks in | 16 |
| Optimiser thresholds | All five read from config (Base44 Secrets) — DOCX never requires a settings UI (verified: zero `Settings`/`threshold` matches in the DOCX extract) | 13 |
| Hardcoded constants | All moved to `.env.example` contract (each entry names the file that uses it) | 01, 13 |

## 4. Ticket index (dependency order — blockers first)

| # | Ticket | Blocked by | Branch |
|---|---|---|---|
| 01 | Spec & repo scaffolding | — | `feature/01-scaffolding` |
| 02 | Message outbox + background sweep | 01 | `feature/02-outbox-sweep` |
| 03 | SMS transport + inbound webhook receiver | 02 | `feature/03-sms-transport` |
| 04 | Request-received + booking-confirmed automation | 03 | `feature/04-booking-messages` |
| 05 | Stop execution states + completion receipt | 03 | `feature/05-stop-states` |
| 06 | Reminder campaign engine | 03 | `feature/06-reminders` |
| 07 | AI assistant | 03, 06 | `feature/07-ai-assistant` |
| 08 | Deposit rules engine | 04 | `feature/08-deposits` |
| 09 | Square payment links + webhook + refunds | 08 | `feature/09-square-payments` |
| 10 | Barber live-position capture | 01 | `feature/10-live-position` |
| 11 | Server-side ETA engine | 05, 10 | `feature/11-eta-engine` |
| 12 | Customer ETA notifications | 03, 11 | `feature/12-eta-messages` |
| 13 | Config centralization + Route Flow hardening | 01 | `feature/13-hardening` |
| 14 | Booking protection & reliability | 02 | `feature/14-protection` |
| 15 | Server-side migration of direct writes + RLS tightening | 05 | `feature/15-server-authority` |
| 16 | Google Calendar integration | 04, 09 | `feature/16-google-calendar` |
| 17 | Reports completion | 15 | `feature/17-reports` |
| 18 | CRM materialization | 06 | `feature/18-crm` |

## 5. What completed means (per ticket)

- **01** → the spec folder, glossary, ADRs, AGENTS.md and the `.env.example` contract exist; every DOCX requirement traced in TRACEABILITY.md.
- **02** → a message can be enqueued and will be delivered exactly once even if the app is closed; failures dead-letter to a visible Needs Review queue.
- **03** → the app can send an SMS and receive a reply; inbound webhooks are signature-verified.
- **04** → a public booking triggers a "request received" SMS; an admin confirmation triggers the confirmation SMS.
- **05** → the barber runs a stop end-to-end (Arrived → Start → Complete) server-side; the customer gets a receipt; no status corruption.
- **06** → due clients get the Tuesday reminder and non-responders the Thursday follow-up; booked/opted-out clients are never messaged; real website leads become client records.
- **07** → routine inbound questions are answered automatically from live booking context; anything ambiguous/angry/sensitive is escalated; change requests become structured, never silent.
- **08** → higher-risk bookings require a deposit at submission; regular clients are untouched.
- **09** → deposit payment links are sent automatically; Square webhooks confirm payment (never a front-end screen); refunds are admin-only and audited.
- **10** → the barber can go live during the working route; position is retained 24h and never exposed as history.
- **11** → live position recalculates downstream ETAs; drift ≥10 min flags the route; customer states match the spec list exactly.
- **12** → customers get on-the-way, arriving-soon and running-late messages automatically.
- **13** → every threshold is configurable in one place; large days finalise; drive times always come from Mapbox; V1 stubs removed.
- **14** → bursts are throttled, families are never blocked, suspicious requests become Needs Review with a reason.
- **15** → no booking/payment/client state changes from the browser; every transition is gated, version-checked and audited; revenue data is admin-only.
- **16** → confirmed appointments mirror to Google Calendar; external busy blocks are respected; Base44 remains the source of truth.
- **17** → Reports shows all §I metrics, computed server-side, test-excluded, admin-only.
- **18** → client profiles show persisted spend/visit history; STOP opts a client out; the full message thread is visible per client/booking.

## 6. Verification approach (how each ticket is tested)

- **Scheduling seam:** all scheduling behaviour is verified through the shared validator (`base44/shared/schedulingValidator.ts`) — the highest seam. Prior art: the read-only `routeFlowDiagnostic` and `suggestPlacementSlots` functions run the real engine path without writes.
- **Idempotency:** replay the same key twice (submission token, message key, webhook event id) → exactly one effect.
- **Webhooks:** valid signature → processed; forged signature → 401; replayed event → no duplicate.
- **Messaging:** all provider calls run in test mode with test credentials — no test ever sends a real message; quiet-hours, opt-out and `is_test` suppression are asserted.
- **ETA:** position updates recalculate downstream ETAs; drift ≥ threshold triggers the next-customer notification path.
- **Review gate:** every `feature/NN-slug` branch is reviewed on the **Standards** axis (repo coding standards) and the **Spec** axis (does it faithfully implement the originating DOCX requirement?) before merging into `beta`.
