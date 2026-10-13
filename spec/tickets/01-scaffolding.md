# 01: Spec & repo scaffolding [BETA]

**What to build:** The planning foundation for the whole automation layer. Create the `spec/` folder with `SPEC.md` (problem, solution, 40 user stories, implementation decisions, testing decisions, out-of-scope), `TRACEABILITY.md` (every DOCX requirement traced to status + file evidence + ticket), `Spec_summary.md` (status tables, coverage map, decisions register, ticket index), root `GLOSSARY.md` (shared domain language), five ADRs in `docs/adr/` (source-of-truth, messaging architecture, outbox/sweep, payments mirror, Google Calendar policy), `AGENTS.md` with the `## Agent skills` block, `docs/agents/` (issue-tracker.md, domain.md), and `.env.example` — the single commented config contract where every value names the file that uses it.

**Blocked by:** None (can start immediately)

**Status:** ready-for-agent [BETA]

**Branch:** `feature/01-scaffolding` → review → merge into `beta`

**Requires:**
- External services: none.
- Access: the DOCX (`Cut_Deluxe_Functional_Specification_Atul.docx`) as the source of truth; read access to the whole repo.
- Testing needs: none beyond file inspection.

**What you get:** A complete, navigable specification and decision record. Any future change can be checked against TRACEABILITY.md to see exactly which DOCX requirement it serves, and `.env.example` becomes the one place every configurable value is documented.

**How to test:**
1. Open `spec/SPEC.md` and confirm every DOCX section (1–13) is represented.
2. Open `spec/TRACEABILITY.md` and spot-check 5 random requirements — each must show status + file evidence + ticket number.
3. Open `.env.example`, pick any value (e.g. `ETA_DRIFT_THRESHOLD_MINUTES`), and confirm the comment names the file that consumes it.
4. Confirm `GLOSSARY.md` defines every term used in the tickets.

**What you see:** A reviewer can answer "what does the DOCX require for payments and where is it built?" by reading two files. Changing any config value requires no code change.

- [ ] `spec/SPEC.md`, `spec/TRACEABILITY.md`, `spec/Spec_summary.md` complete
- [ ] `GLOSSARY.md` at repo root with codebase-derived vocabulary
- [ ] 5 ADRs in `docs/adr/`
- [ ] `AGENTS.md` + `docs/agents/issue-tracker.md` + `docs/agents/domain.md`
- [ ] `.env.example` documents all 30+ values, each with its consuming file named
- [ ] Merged into `beta` after Standards + Spec review
