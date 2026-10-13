# 13: Config centralization + Route Flow hardening [BETA]

**What to build:** The configuration contract and the scheduling-core hardening (DOCX §4C, §9 Priority 6). (1) **Config centralization**: migrate every hardcoded time/threshold/limit in the engine code to read from the `.env.example` contract — `SCHEDULING_STEP_MINUTES`, `ROUTE_FLOW_DRIVE_SAVED_MIN`, `ROUTE_FLOW_IDLE_SAVED_MIN`, `ROUTE_FLOW_MIN_COMBINED_SAVED_MINUTES`, `ROUTE_FLOW_SEARCH_ROUNDS`, `ROUTE_FLOW_MAX_SLOTS_PER_ITEM`, `MAPBOX_MAX_MATRIX_POINTS`, `SCHEDULE_MAX_ATTEMPTS`, `SCHEDULE_MAX_WRITE_ATTEMPTS`, `SCHEDULE_CONFLICT_SETTLE_MS`, `ROUTE_FINALISING_GRACE_MS`, `BOOKING_DEDUP_WINDOW_MS`, `EFFICIENCY_GREEN_MAX_MIN`/`_YELLOW_` — with the values read from Base44 Secrets. (The DOCX requires the optimiser itself — §4C, §9 Priority 6 — but never an admin UI for its thresholds; verified: zero `Settings`/`threshold` matches in `spec/appendix-docx-extract.txt`. Config-in-Secrets satisfies the requirement, so **no Settings UI is built**.) (2) **Mapbox-matrix chunking**: days with more than `MAPBOX_MAX_MATRIX_POINTS − 1` stops currently fail finalisation (`dayRoute.ts` returns `too_many_stops`); implement chunked matrices so large days finalise correctly. (3) **Replace the client-side LLM drive-time placeholder** (`driveTimeService.js`, header comment: "TO SWAP TO GOOGLE MAPS LATER") with server-side Mapbox matrices so draft-day estimates use real road data. (4) **Cleanup**: delete the four retired V1 endpoint stubs (`routeFlowAutoPlace`, `routeFlowApplyReshuffle`, `routeFlowEvaluateReshuffle`, `validateBookingPlacement` — all returning 410) and remove the temporary `schedulingAuthSelfTest` function (its own comment says to remove it after Stage 0 verification).

**Blocked by:** 01 (config contract exists)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/13-hardening` → review → merge into `beta`

**Requires:**
- External services: Mapbox (existing `CUTDELUXEMAP` secret) — no new service.
- Prerequisites: ticket 01 (`.env.example` contract); Base44 Secrets for the backend values.
- Testing needs: a test day with 25–30 stops (to exercise chunking); a draft day (to verify Mapbox drive times); a before/after performance comparison on finalisation; a check that changing each threshold in config changes engine behaviour without a code deploy.

**What you get:** Every knob in one place (change a value, no code change), big days that finalise, real road times everywhere, and a clean codebase with dead stubs removed.

**How to test:**
1. Change `ROUTE_FLOW_MIN_COMBINED_SAVED_MINUTES` in Secrets → next auto-optimise run uses the new threshold (verify via a routeFlowDiagnostic-style dry run).
2. Finalise a 30-stop test day → completes via chunked matrices (previously `too_many_stops`).
3. Open a draft day → drive times come from Mapbox (verify against a known address pair), not the LLM placeholder.
4. Call a retired V1 endpoint → it no longer exists (removed).
5. Confirm `schedulingAuthSelfTest` is gone and no code references it.
6. Change each of the five optimiser thresholds in Base44 Secrets → the next optimisation run uses the new values with no code deploy.

**What you see:** Config-driven thresholds (change a Secret, behaviour changes); a 30-stop day finalising; Mapbox-sourced drive times on draft days; zero references to V1 stubs or the self-test.

- [ ] All engine constants read from config (no hardcoded values remain)
- [ ] Five optimiser thresholds read from config (no hardcoded values; DOCX §4C requires the optimiser, not a settings UI)
- [ ] Chunked matrices: >24-stop days finalise
- [ ] Draft-day drive times from Mapbox (LLM placeholder removed)
- [ ] Four V1 410-stubs deleted
- [ ] `schedulingAuthSelfTest` removed
- [ ] Merged into `beta` after Standards + Spec review
