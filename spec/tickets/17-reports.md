# 17: Reports completion [BETA]

**What to build:** The missing report metrics (DOCX §4I). Today `Reports.jsx` shows bookings/outcomes and new clients, but revenue, average price, revenue/hour, drive time, idle time, and route efficiency exist only scattered (Dashboard, per-day Route tab) or not at all. (1) Compute the full §4I metric set **server-side**: bookings, clients serviced (unique clients, not just completed bookings), revenue, average price, revenue/hour, drive time, idle time, new clients, and route efficiency (aggregated from `DayRoute.drive_minutes`/`idle_minutes`/`waste_minutes` and `RouteConnection.efficiency`). (2) All metrics exclude `is_test` data **at the query level** (as `Reports.jsx` already does). (3) Revenue/spend metrics are **admin-only** (entity RLS tightened in ticket 15). (4) Date-range reporting preserved and extended to the new metrics.

**Blocked by:** 15 (RLS tightening lands first so revenue is admin-only from day one)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/17-reports` → review → merge into `beta`

**Requires:**
- External services: none new.
- Prerequisites: ticket 15 (RLS); some completed finalised days to report on.
- Testing needs: a date range with completed finalised days (including drive/idle data) plus test data in the same range (must be excluded); an admin account and a non-admin account.

**What you get:** The complete operations report the spec asks for — revenue, average price, revenue/hour, drive time, idle time, and route efficiency alongside bookings and clients — computed server-side, test-clean, and visible only to admins.

**How to test:**
1. Open Reports as admin for a range with completed days → all §4I metrics present and correct (spot-check revenue against the completed bookings' prices).
2. Confirm "clients serviced" counts unique clients, not completed bookings.
3. Confirm route efficiency aggregates the drive-leg efficiency bands.
4. Seed TEST bookings in the same range → none of the metrics include them.
5. Open Reports as a non-admin → revenue/spend metrics are denied.
6. Change the date range → metrics recompute server-side.

**What you see:** A complete, correct operations report; test data invisible; financial metrics restricted to admins.

- [ ] All §4I metrics present: bookings, clients serviced (unique), revenue, average price, revenue/hour, drive time, idle time, new clients, route efficiency
- [ ] Computed server-side (not client-side reduction)
- [ ] `is_test` excluded at the query level
- [ ] Revenue/spend admin-only
- [ ] Date-range reporting preserved
- [ ] Merged into `beta` after Standards + Spec review
