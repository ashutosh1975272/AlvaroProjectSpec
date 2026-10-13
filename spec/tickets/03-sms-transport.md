# 03: SMS transport + inbound webhook receiver [BETA]

**What to build:** The first provider adapter of the channel-agnostic messaging layer (ADR 0002). (1) A Twilio send implementation behind the internal `sendMessage({channel, to, text, context})` interface, writing a `Message` entity row for every send with the full delivery lifecycle (`queued → sent → delivered | failed`). (2) An **inbound webhook function** (Base44 function HTTP endpoint) that receives Twilio status callbacks and inbound SMS replies, verifies the Twilio HMAC signature in-function (the platform provides no managed verification), and writes inbound messages as `Message` rows (`direction: incoming`). (4) Quiet-hours enforcement — no automated sends outside `MESSAGING_QUIET_HOURS_START/END` (Sydney time); a message queued just before the window closes is **held and sent at the next open window (08:00)** — never dropped (DOCX §4J no-silent-failure). (5) A `STOP` reply writes two **new fields** on the client record: `opted_out` (boolean) and `stop_received_at` (timestamp) — the `ReminderClient` entity has no opt-out field today (only admin `is_paused`; verified: zero `opted_out`/`stop_received` matches across `base44/entities/*.jsonc`). All credentials are Base44 Secrets, never in code or the browser bundle.

**Blocked by:** 02 (outbox exists to send through)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/03-sms-transport` → review → merge into `beta`

**Requires:**
- External services/accounts to arrange: **Twilio account** (trial account is sufficient for testing); a **Twilio phone number** (the business SMS sender); later a **WhatsApp Cloud API** account for the fast-follow (same adapter interface) and a **verified email sender domain** for email. **WhatsApp fast-follow note:** WhatsApp Cloud API requires Meta pre-approved message templates (approval takes days) — SMS needs no template approval, which is why SMS ships first (also aligns with DOCX §7 provider criteria).
- Secrets to create in Base44 Dashboard → Secrets: `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_FROM_NUMBER` (and later `WHATSAPP_API_KEY`, `EMAIL_FROM_ADDRESS`).
- Testing needs: Twilio test credentials (Twilio provides magic test numbers that simulate delivery without real SMS); 2–3 real test phones for end-to-end verification; a tool (curl/Postman) to replay the webhook with valid and invalid signatures; the function's public HTTP URL (shown in the Base44 dashboard) to register as the Twilio webhook.

**What you get:** The ability to send an SMS to a customer and receive their reply — the minimum viable transport for all ten DOCX §6 matrix messages — plus signature-verified inbound webhooks, quiet-hours compliance, and opt-out capture.

**How to test:**
1. Trigger a test send via the outbox with Twilio test credentials → `Message` row advances `queued → sent → delivered`.
2. Send an inbound SMS to the Twilio number → webhook fires → `Message` row created with `direction: incoming`.
3. Replay the webhook with a tampered signature → HTTP 401, nothing recorded.
4. Trigger a send outside quiet hours → send is held until the window opens.
5. Reply "STOP" → the client's opt-out flag is set and visible on their record.

**What you see:** In the Messages admin view: message rows with live delivery status; inbound replies appearing; rejected forged webhooks in logs; no automated sends between 20:00 and 08:00.

- [ ] Channel-agnostic send interface + Twilio adapter
- [ ] Every send writes a `Message` row with lifecycle status
- [ ] Inbound webhook endpoint with HMAC signature verification (forged → 401)
- [ ] Quiet hours enforced from config
- [ ] "STOP" reply sets new `opted_out` + `stop_received_at` fields on the client record (entity schema extended)
- [ ] A send held by quiet hours goes out at the next open window — never silently dropped
- [ ] Credentials only via Base44 Secrets
- [ ] Merged into `beta` after Standards + Spec review
