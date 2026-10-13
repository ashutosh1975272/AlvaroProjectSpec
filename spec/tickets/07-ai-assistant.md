# 07: AI assistant [BETA]

**What to build:** The automated inbound conversation handler (DOCX §4A, §6 row 8). (1) An **intent layer** fed by the inbound webhook (ticket 03): routine intents (booking status, business hours, location/coverage question, "are you open") are answered **autonomously** from live server state — booking context comes from the same sanitized lookup `checkBookingStatus` uses (customer name + phone match; existing booking, services, address, availability window, calendar status, payment state). (2) **Change requests**: any request to change a booking — especially a locked/confirmed one — is turned into a **structured change request** attached to the booking (never a silent alteration; protected appointments are never touched by automation). (3) **Escalation**: ambiguous, angry, sensitive, or unusual conversations, and anything payment-dispute-related, are escalated to the barber (SMS + an "Escalated" queue in Messages) with a 2-business-hour response expectation. (4) **The hard guardrail**: the assistant can never state or promise an appointment time that the shared scheduling validator has not produced — every time-like statement is sourced from validated booking/calendar state. (5) Every assistant action is logged as a `Message` row marked automated, with the intent and outcome recorded. Model: Base44's built-in `invokeLLM` (already used in the repo at `driveTimeService.js:28`), with a bring-your-own-provider fallback via Secrets.

**Blocked by:** 03 (inbound channel), 06 (suppression context)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/07-ai-assistant` → review → merge into `beta`

**Requires:**
- External services: Base44 built-in `invokeLLM` integration (available on the platform); Twilio (ticket 03) for inbound + escalation SMS. No separate AI vendor required.
- Prerequisites: tickets 03 and 06 complete; a signed-off **intent list** (which intents are autonomous vs draft-only) and the **escalation contact** (barber's phone / admin queue).
- Testing needs: test conversations for each intent class (routine, change-request, angry, ambiguous, payment dispute); a booking in each state (provisional, locked, confirmed) to verify the assistant never alters protected state; a name+phone pair that matches no customer (must get the generic privacy-safe response).

**What you get:** Customers can text routine questions and get instant, correct answers from live booking context — while the barber is driving or cutting — and anything the assistant cannot safely handle becomes a visible human task. This is DOCX §4A's core promise.

**How to test:**
1. Text "when is my appointment?" with a real customer's phone → assistant answers with the confirmed time/window from live state.
2. Text a change request against a **confirmed** booking → a structured change request is created and attached; the booking itself is untouched; admin sees it in the queue.
3. Text something angry/ambiguous → escalation delivered to the barber within SLA; thread marked as needing human attention (and suppressed from automation per ticket 06).
4. Ask about a booking with a time the validator never produced → the assistant refuses to state a time (guardrail).
5. Check the client's message history: every assistant reply is recorded as automated with intent + outcome.

**What you see:** Routine questions answered in seconds; change requests appear as structured admin items (never silent changes); escalations reach the barber; the message history shows automated/manual labels; zero unvalidated time promises in any test path.

- [ ] Routine intents answered autonomously from live booking context
- [ ] Existing-client recognition (no re-asking known information)
- [ ] Change requests become structured requests; protected appointments never auto-altered
- [ ] Ambiguous/angry/sensitive/unusual escalated to barber within SLA
- [ ] Never-unvalidated-times guardrail enforced on every path
- [ ] All assistant actions logged automated with intent + outcome
- [ ] Merged into `beta` after Standards + Spec review
