# Skills used to build this spec — and how to continue with them

This file records the agent skills used to produce everything in this folder,
so any Claude/agent receiving this folder can reuse the same methodology and
continue the workflow without re-deriving it.

## Skills used

| Skill | Source | What it produced here | Where the output lives |
|---|---|---|---|
| **setup-matt-pocock-skills** | `mattpocock/skills` | Environment setup: `AGENTS.md`, `docs/agents/issue-tracker.md`, `docs/agents/domain.md`, `GLOSSARY.md` at the repo root | repo root (not pushed with this folder — regenerate via the skill if needed) |
| **grill** | `mattpocock/skills` | Structured interrogation of the DOCX: every place the spec was silent was surfaced and decided (21 registered decisions — quiet hours, drift threshold, deposit amounts, cadence days, retention, rate limits, etc.) | decisions register in `Spec_summary.md`; ADRs in `docs/adr/0001–0005` |
| **to-spec** | `mattpocock/skills` | The working spec: problem, solution, 40 user stories, implementation/testing decisions, out-of-scope | `SPEC.md` |
| **to-tickets** | `mattpocock/skills` | The 18 tracer-bullet tickets in DOCX §12 delivery order, each with: what to build, blocked-by, `ready-for-agent [BETA]` status, branch, Requires (external services/secrets/testing needs), what you get, how to test (numbered), what you see (observable evidence), acceptance criteria | `tickets/01-scaffolding.md` … `18-crm.md` |
| **rea (evidence discipline)** | `morluto/rea` | Every claim classified as **observation / inference / unknown**, cited `file:line`, no unverified guessing; removals from spec require evidence | throughout `TRACEABILITY.md` and every ticket's evidence citations |

**Note on how the mattpocock skills were applied:** the `npx skills@latest add`
installer did not complete in this environment, so the SKILL.md files were
fetched directly from `raw.githubusercontent.com/mattpocock/skills/main/...` and
applied manually. The outputs are identical to the skill-driven workflow.

## Source-of-truth hierarchy (strict)

1. **The DOCX** (`Cut_Deluxe_Functional_Specification_Atul.docx`, mirrored as
   `appendix-docx-extract.txt` in this folder) — the single source of truth.
   Where the DOCX speaks, it wins over everything.
2. **The existing codebase** — secondary truth for what is already built
   (`base44/functions/`, `base44/entities/*.jsonc`, `base44/shared/*.ts`,
   `base44/workflows/`, `src/`). Tickets cite `file:line` evidence from it.
3. **Registered decisions** (decisions register + ADRs) — only where the DOCX
   is silent. Never invent a feature the DOCX does not mention.

## How to continue the workflow

- Work happens on the Base44 platform. Branch model: `main` protected
  (senior-approved merges only); all work on `beta`; each ticket is developed
  on `feature/NN-slug` and merged into `beta` after a two-axis review:
  **Standards** (repo coding standards) and **Spec** (faithful to the DOCX
  requirement).
- Every ticket is labelled `ready-for-agent [BETA]` until merged.
- DOCX §8's ten unbreakable automation rules are global acceptance criteria for
  every scheduling/messaging/payment change.
- Config contract: no hardcoded times/thresholds/limits — everything lives in
  the `.env.example` contract (at the repo root), each entry naming the file
  that consumes it. Backend values are Base44 Secrets (`process.env` /
  `secrets.get()`); frontend values use `VITE_`-prefixed `.env.local`.
- Test data (`is_test`: manual flag OR customer last name exactly "TEST") is
  excluded from analytics, messaging, reminders and reports everywhere — see the
  exclusion-scope matrix in `TRACEABILITY.md` §4J.
- When auditing or extending this folder, apply the rea discipline: cite
  `file:line` for every claim, label inferences, and list anything you cannot
  verify as unknown rather than guessing.

## Standing rules for every Base44 agent prompt

Include these rules in every prompt given to the Base44-building agent:

1. Do not modify anything in `base44/shared/` — `schedulingValidator.ts`,
   `reshuffleEngine.ts`, `routeFlowOptimiser.ts`, `routeFlowEngine.ts`,
   `dayRoute.ts`, `appointmentTiming.ts`. The scheduling core works; new
   work sits beside it, never inside it.
2. Do not change any existing entity's existing fields. Add new fields and
   new entities only.
3. Until the real provider lands, no message is actually sent — the sender
   is a test double that logs.
4. Every new backend function must be admin-gated using the existing
   `requireAdminActor` from `base44/shared/schedulingAuth.ts`, except where
   the ticket says otherwise. (The platform's workflow runner invokes
   functions with a service identity that resolves to admin —
   `schedulingAuth.ts:15-21` — so Workflows can call gated functions,
   exactly as `routeFlowAutoPlaceV2` is called today.)
5. Test data is never sent to. Anything flagged `is_test` is marked
   `suppressed`, never delivered.
6. Verify every acceptance check against the live app with test records,
   then delete the test records.

**Platform constraints discovered on Day 1 (Base44):**
- Entities have **no database-level unique constraint** — idempotency must
  be enforced application-side (oldest-row-wins cleanup at enqueue time;
  same-key collapse at sweep claim time, per ticket 02).
- Entity write permissions can require the **service role**, which no
  signed-in user holds and which bypasses row-level security — so only
  server code (backend functions, workflows) can write; admins and users
  read only.
- `enqueueMessage` requires every message to link to a `booking_request_id`
  or `client_id` — an unverified link is refused (422) rather than assumed
  production data, because `is_test` can only be determined through the
  linked record.
