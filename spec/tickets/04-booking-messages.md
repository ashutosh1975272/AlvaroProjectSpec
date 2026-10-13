# 04: Request-received + booking-confirmed automation [BETA]

**What to build:** The first two automated customer messages (DOCX §6 matrix rows 1 and 3). (1) **Request received** — when `submitPublicBooking` creates a booking request, enqueue an automatic message to the customer ("we received your request", with name, service, requested day/window). (2) **Booking confirmed** — when an admin confirms a request (the `confirm_request` path of `applyManualPlacement`, and the day finalisation path of `finaliseRoute`), enqueue the confirmation message with the agreed date, time/window, service, address, and payment status. A small template engine renders approved copy with the booking's context. Every send goes through the outbox (ticket 02) and the transport (ticket 03), creates a `Message` row marked `manual_or_automated: automated`, and is excluded entirely for `is_test` bookings. No provisional-holding message is sent (matrix row 2 says "usually no" — decided: never in MVP).

**Blocked by:** 03 (transport exists)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/04-booking-messages` → review → merge into `beta`

**Requires:**
- External services: Twilio (from ticket 03) — no new service.
- Prerequisites: ticket 03 complete; **approved message copy** for the two templates (business owner signs off the wording — the current `BookingRules.jsx` templates are copy-only placeholders).
- Testing needs: TEST customers (last name exactly "TEST") to verify exclusion; 1–2 real test phones to receive the messages; a confirmed booking in the admin calendar.

**What you get:** A new customer who books online automatically hears back without the barber doing anything, and a customer whose booking is confirmed automatically receives the agreed time — the two trust-critical moments of the DOCX §3 journey, steps 1 and 5.

**How to test:**
1. Submit a public booking as a TEST customer → no message is sent (test exclusion).
2. Submit a real test booking (real phone, test scenario) → customer receives the "request received" SMS; a `Message` row exists with `manual_or_automated: automated`.
3. As admin, confirm the request in the calendar → customer receives the confirmation SMS with date, time/window, service, address.
4. Finalise the day → any confirmed-but-not-yet-messaged bookings receive their confirmation.
5. Check Route Flow Activity / Message history: every automated send has an audit row.

**What you see:** Real phone receives both SMS at the right moments; the Messages admin view shows the automated rows; zero messages for TEST customers; no holding message after provisional placement.

- [ ] "Request received" fires from `submitPublicBooking`
- [ ] "Booking confirmed" fires from the confirm path and finalisation
- [ ] Templates render name/service/date/time-window/address/payment status
- [ ] `is_test` bookings never messaged
- [ ] Every send has a `Message` row + audit entry
- [ ] No provisional-holding message sent (matrix row 2 = no)
- [ ] Merged into `beta` after Standards + Spec review
