# 09: Square payment links, webhook ingestion, refunds [BETA]

**What to build:** Real payments on top of Square (DOCX §4F, §8; ADR 0004). **(0 — Schema migration, required first:)** `BookingRequest.payment_status` is binary `Unpaid|Paid` today (`base44/entities/BookingRequest.jsonc:276-292`, marked "display only — Square owns truth") — expand to the DOCX §4F state set: `unpaid, deposit_required, deposit_paid, paid, refunded, partially_refunded, offline_approved`, and add `square_payment_id`. **(1)** **Square connector**: OAuth connection (Base44's native Square connector) with the access token stored as a Secret. **(2)** **Payment links**: when a booking's deposit rules (ticket 08) require payment, a Square payment link/invoice is created (amount + expiry from `DEPOSIT_EXPIRY_HOURS`) and sent automatically through the messaging pipeline (matrix row 4: amount, expiry, booking ID). **Deposit expiry:** when `DEPOSIT_EXPIRY_HOURS` passes without payment, the link is marked expired and the booking surfaces in the admin Needs Review queue with a human-readable reason (DOCX §4J no-silent-failure) — the customer can then be re-sent a fresh link. **(3)** **Webhook ingestion**: a function HTTP endpoint registered as the Square webhook, verifying the Square notification signature in-function, mapping the Square payment id to the booking, and updating the mirrored `payment_status` **only from provider events** — the browser-side "Mark Paid" button (`AdminPayments.jsx:26-34`) is removed, because §8 says payments are confirmed "by provider/server event, not client-side assumption". **(4)** **Refunds**: admin-only server functions for full and partial refunds via Square, with approver + reason audited. **Refund math:** a refund applies to the most recent payment first (balance before deposit) unless the admin explicitly chooses the scope; every refund records amount, scope (full/partial), approver and reason. **(5)** **Receipts**: on payment confirmation, the receipt message is enqueued (matrix row 9). **(6)** Post-completion balance collection: after a deposit+balance booking completes, the balance link is sent or an admin marks an approved offline method (Cash/Transfer/Card) with audit.

**Blocked by:** 08 (deposit rules decide when a link is needed)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/09-square-payments` → review → merge into `beta`

**Requires:**
- External services/accounts to arrange: **Square developer account + Square app** (OAuth client id/secret) — **confirm with the business whether Square is already connected**; if not, this onboarding step is a prerequisite inside this ticket. **Square webhook registration** (the function's public HTTP URL in the Square dashboard). **Square sandbox** environment for all testing (never live credentials during development).
- Secrets: `SQUARE_ACCESS_TOKEN`, `SQUARE_WEBHOOK_SECRET` (and the OAuth client id/secret via the Base44 connector flow).
- Testing needs: Square sandbox buyer/test cards; a deposit-required booking (from ticket 08 test scenarios); a webhook replay tool for valid/forged/duplicated signature tests; test customers with `is_test` excluded from real Square charges.

**What you get:** Deposit-required customers automatically receive a payment link; payments are confirmed only by Square events; the app mirrors Square's truth (no custom ledger — spec §11); refunds and receipts are handled and audited. This closes the last anti-pattern in the codebase (browser-side mark-paid).

**How to test:**
1. Create a deposit-required booking → payment-link SMS sent automatically with amount, expiry, booking ID.
2. Pay via the Square sandbox link → Square webhook fires → booking mirrored to Paid, receipt message sent, `square_payment_id` stored.
3. Replay the webhook with a forged signature → 401, no state change.
4. Replay the same Square event id twice → exactly-once effect (idempotent).
5. Attempt "Mark Paid" from the browser → the old path is gone; only admin-approved offline entry (audited) or a provider event can set Paid.
6. Issue a partial refund as admin → Square refund executed, amount/state recorded, approver + reason audited.

**What you see:** In the Payments admin view: payment state that traces to Square event ids; links sent automatically; forged/duplicated webhooks rejected; refund audit trail; no browser-side payment writes.

- [ ] Square OAuth connection established (or onboarding step completed first)
- [ ] Payment-status schema expanded (7-state set) + square_payment_id added (evidence: BookingRequest.jsonc:276-292 is binary today)
- [ ] Payment links created + sent automatically when a deposit rule requires
- [ ] Expired deposit link (DEPOSIT_EXPIRY_HOURS) surfaces in Needs Review with reason — never silently dropped
- [ ] Webhook endpoint verifies Square signature; updates mirror only from provider events
- [ ] Webhook processing idempotent on Square event id; forged → 401
- [ ] Browser-side "Mark Paid" removed; offline payments are admin-approved + audited
- [ ] Full/partial refunds via admin-only functions with audit (most recent payment refunded first unless admin chooses scope)
- [ ] Receipt sent on payment confirmation
- [ ] Merged into `beta` after Standards + Spec review
