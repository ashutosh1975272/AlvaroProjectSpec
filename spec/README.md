# Cut Deluxe — Spec folder

Automation-layer specification for **Cut Deluxe** (Base44 mobile-barber booking
app). Source of truth: `Cut_Deluxe_Functional_Specification_Atul.docx`
(October 2026), mirrored textually in `appendix-docx-extract.txt`.

## Folder map

| File | What it is |
|---|---|
| `SPEC.md` | Working spec: problem, solution, 40 user stories, implementation/testing decisions, out-of-scope |
| `TRACEABILITY.md` | Evidence ledger — every DOCX requirement → status → `file:line` code evidence → ticket that completes it |
| `Spec_summary.md` | Status tables, coverage map, 21-decision register (DOCX-silent items), ticket index |
| `SKILLS.md` | The agent skills used to build this folder (mattpocock: setup/grill/to-spec/to-tickets; morluto/rea evidence discipline) and how to continue the workflow |
| `appendix-docx-extract.txt` | Full text extraction of the source-of-truth DOCX (173 paragraphs, 4 tables) |
| `tickets/01-scaffolding.md` … `18-crm.md` | The 18 Base44-ready tickets in DOCX §12 delivery order |

## Read this first

1. `SPEC.md` — what the automation layer is.
2. `TRACEABILITY.md` — what exists today vs what each ticket builds, with
   `file:line` evidence from the codebase.
3. `tickets/` — in dependency order: 01 (scaffolding) → 02 (outbox+sweep) →
   03 (SMS transport) → 04 (booking messages) → 05 (stop states) → … → 18
   (CRM). Unblocked frontier: 01, 10, 13.

Every ticket carries: what to build, blocked-by, `ready-for-agent [BETA]`
status, branch, Requires (external services/accounts/secrets + testing needs),
what you get, how to test (numbered steps), what you see (observable
evidence), and acceptance criteria.

## Evidence discipline

Every claim in this folder is an **observation** (verified, `file:line` cited),
an **inference** (labelled with its assumption), or an **unknown**. Nothing is
asserted without proof; features not in the DOCX are not built.
