# 02: Message outbox + background sweep [BETA]

**What to build:** The reliable delivery backbone every automated message will use (ADR 0002, ADR 0003). A durable outbox entity (work item: idempotency key, channel, recipient, payload, attempt count, next-attempt-at, dead-letter reason) plus a server-side enqueue helper that automations call instead of sending directly, plus a **scheduled Workflow that sweeps every 5 minutes** (Base44's minimum scheduled-trigger interval) — delivered as a new `base44/workflows/MessageSweep.jsonc` (scheduled trigger, same pattern as the in-repo `RouteFlowAutoPlace.jsonc`) **and activated in the Base44 dashboard** — which claims due items, attempts delivery through the provider adapter, applies backoff between attempts, and after `MESSAGE_MAX_SEND_ATTEMPTS` dead-letters the item to a visible admin **Needs Review** queue with a human-readable reason. Processing is idempotent: the same key processed twice has exactly-once effect. **Idempotency key format (DOCX §8):** `channel|target_id|event_type` (e.g. `sms|db-abc123|booking_confirmed`) — one key per business event, so replays across retries and sweeps are always safe and auditable. **Sweep health (DOCX §4J no-silent-failure):** the sweep records `last_successful_at` and a consecutive-failure count in an admin-visible health record — Base44 auto-deactivates workflows after 5 consecutive failures, so a dead sweep must itself be visible, or the entire automation layer fails silently. Test-data items are never sent.

**Blocked by:** 01 (config contract exists)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/02-outbox-sweep` → review → merge into `beta`

**Requires:**
- External services: Base44 **Workflows** (scheduled trigger) must be enabled on the app — this app already uses Workflows (`base44/workflows/`), so scheduled triggers are the available mechanism; no third-party service needed for this ticket.
- Secrets: none yet (the sweep calls the provider adapter, which ticket 03 implements; until then a test double logs sends).
- Testing needs: a way to observe sweep runs (Workflow run history in the Base44 dashboard); test outbox items marked `is_test`.

**What you get:** Any automation (reminders, confirmations, ETA alerts) can enqueue a message and trust it will be delivered even if the app/browser is closed, retried safely, and never lost silently — satisfying DOCX §7 ("must survive the app/browser being closed") and §8 ("background retries must be idempotent").

**How to test:**
1. Enqueue a message with key `K` and a test-double adapter that succeeds → sweep runs → item delivered exactly once.
2. Enqueue the same key `K` again (duplicate) → sweep runs → still exactly one send.
3. Enqueue an item whose adapter fails → confirm retry with backoff up to `MESSAGE_MAX_SEND_ATTEMPTS`.
4. Let attempts exhaust → item appears in the admin Needs Review queue with its reason.
5. Enqueue an `is_test` item → never sent.

**What you see:** In the Base44 dashboard: the scheduled Workflow firing every 5 minutes; per-item attempt counts advancing; exhausted items visible in Needs Review; zero duplicate sends for replayed keys.

- [ ] Outbox entity + enqueue helper exist
- [ ] `base44/workflows/MessageSweep.jsonc` (scheduled trigger) created **and activated in the Base44 dashboard**; sweeps at `BACKGROUND_SWEEP_INTERVAL_MINUTES`
- [ ] Retry with backoff, then dead-letter to visible Needs Review with reason
- [ ] Idempotency key format (`channel|target_id|event_type`) documented and enforced (same key twice → one effect)
- [ ] Sweep health (last successful run + consecutive-failure count) visible in admin
- [ ] `is_test` items never sent
- [ ] Config values read from secrets (`MESSAGE_MAX_SEND_ATTEMPTS`, `BACKGROUND_SWEEP_INTERVAL_MINUTES`)
- [ ] Merged into `beta` after Standards + Spec review
