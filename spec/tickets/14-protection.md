# 14: Booking protection & reliability [BETA]

**What to build:** Anti-abuse and reliability guards for public booking (DOCX §4H). (1) **Rate limiting**: max `BOOKING_RATE_LIMIT_PER_HOUR` requests per normalized phone per hour and per normalized email per hour, counted separately. **Family-aware algorithm (concrete, per DOCX §4H "without blocking legitimate families/groups"):** a submission is refused only when the same normalized phone AND the same normalized email both exceed the hourly limit, or when a single identifier exceeds 3× the limit — so a family sharing one phone but using different emails is never blocked, while a spammer hammering one phone+email pair is. Same phone + same address + different customer names counts as a household (allowed up to the per-customer active limit). This is a per-identifier limit with a household exemption path, not a blanket block. (2) **Active-booking limits**: max `ACTIVE_BOOKING_LIMIT_PER_CUSTOMER` concurrent unconfirmed (unscheduled/provisional) bookings per customer. (3) **Suspicious → Needs Review**: repeated/threat-pattern submissions become `placement_status: needs_review` with a human-readable reason instead of entering the schedule. (4) **Cancellation/no-show history**: a server-side counter of no-shows/cancellations per customer (feeding the deposit rules of ticket 08). (5) **Canonical phone normalization**: this ticket **consumes** the shared normalizer delivered by ticket 08 (`base44/shared/phoneNormalization.ts`) — it does not re-implement it.

**Blocked by:** 02 (sweep exists for any async counter work)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/14-protection` → review → merge into `beta`

**Requires:**
- External services: none new.
- Prerequisites: ticket 02 (sweep) and ticket 08 (delivers the shared `phoneNormalization.ts` module this ticket consumes; the no-show counter this ticket builds feeds ticket 08's deposit rules — sequence 08 before 14 so the canonical key exists first).
- Testing needs: burst test (10+ rapid submissions from one phone/email); a family-of-4 scenario (same phone + same address, 4 different customers — must NOT be blocked); a customer with 3 concurrent unconfirmed bookings (4th must be refused with a clear message); a suspicious burst pattern (must land in Needs Review with reason); customers with recorded no-shows (counter correctness).

**What you get:** Spam and repeated submissions are throttled, families are never blocked, suspicious requests are held for admin review with a reason, and no-show history exists to inform deposit decisions — all without breaking legitimate booking flows.

**How to test:**
1. Submit 10+ bookings in an hour from one phone+email pair → excess requests refused with a clear rate-limit message.
1b. Submit bookings from one shared household phone but different emails → never blocked (family-aware: refusal requires both identifiers over the limit).
2. Submit 4 bookings from one household (same phone/address, different customers) → all accepted (family-aware).
3. Hold 3 unconfirmed bookings for one customer → a 4th is refused with the active-limit message.
4. Run a suspicious burst pattern → requests land as `needs_review` with a human-readable reason, visible in the admin queue.
5. Record 2 no-shows for a customer → the counter shows 2 and the deposit engine (ticket 08) requires a deposit.
6. Look up the same phone in public status check and admin view → same customer (unified key).

**What you see:** Rate-limit and active-limit messages that a human understands; a Needs Review queue with reasons; a per-customer no-show counter; consistent phone identity across the app.

- [ ] Rate limit per normalized phone/email per hour, family-aware
- [ ] Active-booking limit per customer
- [ ] Suspicious patterns → `needs_review` with human-readable reason
- [ ] No-show/cancellation history counter (feeds deposits)
- [ ] Canonical phone normalization shared across endpoints
- [ ] Legitimate family/group flows never blocked
- [ ] Merged into `beta` after Standards + Spec review
