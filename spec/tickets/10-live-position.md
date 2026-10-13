# 10: Barber live-position capture [BETA]

**What to build:** Explicit, privacy-aware live location for the working route (DOCX §4E, §11). (1) A **"Start live tracking" / "Stop" toggle** in the Route tab — the only way position is ever captured. (2) While enabled **and** the day is the barber's working route (a finalised day in progress), the browser's Geolocation API watch posts position updates to a server function, which stores the **current** position on the day route (a single latest-position record, plus the minimal trail needed for the ETA engine — not a full history). (3) **Retention**: positions are pruned to `LIVE_POSITION_RETENTION_HOURS` (24h) by the sweep; there is no location-history view anywhere in the app (spec §11 explicitly forbids permanent real-time location history). (4) Tracking auto-stops when the working day ends or the barber toggles off; off-days never capture anything.

**Blocked by:** 01 (config contract exists)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/10-live-position` → review → merge into `beta`

**Requires:**
- External services: none (browser Geolocation API + existing Base44 functions).
- Prerequisites: HTTPS — browser geolocation requires a secure context (the Base44 preview/production domain qualifies; local `localhost` also qualifies).
- Testing needs: a phone (or browser device emulation) on the Route tab of a finalised day; a second device/desk-check to verify off-days capture nothing; a way to fast-forward or backdate positions to verify the 24h prune.

**What you get:** The barber can share live location during the working route — and only then — giving the ETA engine (ticket 11) the current position it needs, with hard privacy limits the spec demands.

**How to test:**
1. Open a finalised working day on a phone; press **Start live tracking** → grant location permission → position updates arrive at the server function and are stored.
2. Press **Stop** → updates cease immediately.
3. Open a non-working day / off-day / non-finalised day → toggle is unavailable or captures nothing.
4. Leave tracking off entirely → zero position records exist.
5. Backdate a position beyond 24h → sweep prunes it; no history view exposes it.
6. Reload the app mid-day → tracking state survives (server-side), and the toggle reflects reality.

**What you see:** A clear toggle that shows tracking state; position updates flowing only during a working route; no location history anywhere; old positions purged automatically.

- [ ] Explicit start/stop toggle; nothing captured without it
- [ ] Capture gated to the barber's working route/day
- [ ] Position stored server-side (latest position + minimal ETA trail only)
- [ ] 24h retention prune from config (`LIVE_POSITION_RETENTION_HOURS`)
- [ ] No location-history view exists anywhere
- [ ] Auto-stop at end of working day
- [ ] Merged into `beta` after Standards + Spec review
