# Offline Legal Translation — Document Set

Reference documents for the on-premises translation pipeline (EN↔SL, later DE).
Kept here, one level above the encrypted container, so they remain readable when
the container is closed — which is exactly when you need the instructions for
opening it.

**Location:** `~/translation-work/docs/`
**Kit location:** `~/Claude_Stuff/cli_projects/translation-tools/`
**Data location:** `~/translation-work/confidential-projects/` (encrypted container)

None of these documents contain case material. Claude Code may read them.

---

## Current — use these

| File | What it covers |
|---|---|
| `01-operating-manual` | **Start here.** Full operating manual v1.2: layout, setup, per-format handling, the Phase 1 decision experiment, daily commands, lint report, troubleshooting, environment variables, decisions register. |
| `02-claude-code-handover` | Where work runs, the confidentiality boundary for Claude Code sessions, guard configuration, the task list with acceptance criteria, and what must never enter a session. |
| `03-encrypted-case-container` | Options, setup, and honest trade-offs for the LUKS container; the `claude` guard that makes the boundary structural rather than remembered. |

Each is provided as `.docx` (primary), `.pdf`, and `.md`.

---

## Archive — superseded, retained for the record

These informed the current design and are kept for provenance. Do not follow
their instructions; paths and recommendations in them are out of date.

| File | Superseded by | Still interesting for |
|---|---|---|
| `archive/hardware-assessment` | manual §1–3 | The memory-bandwidth analysis and why MoE models win on this machine |
| `archive/workflow-architecture` | manual, all | First articulation of the OCR risk and the CAT-tool argument |
| `archive/addendum-1-confirmed-hardware` | manual §2, addendum 2 | Why the RAM upgrade was rejected; the vision-capability discovery |
| `archive/addendum-2-quality-first` | manual §13 | The quality-vs-throughput reversal and the back-translation proposal (later dropped) |

---

## Reading order for someone new to this

1. `01-operating-manual` §1 — what the system is and the one number that judges it
2. `03-encrypted-case-container` §1–2 — why the data is where it is
3. `02-claude-code-handover` §2 — the boundary, before opening any session
4. `01-operating-manual` §6 — the Phase 1 experiment that decides whether to proceed

## Open items (manual §14)

- Measure the 7,000-row spreadsheet: `tr-xlsx <file> --survey`
- Trial the container on a throwaway 1 GB image before real data
- Confirm full-disk encryption as well as the container
- Test whether Qwen vision input works, for dual-engine OCR
- Seed the translation memory from the translator's prior work
- Verify German quality against base Gemma before extending to that pair
- Settle the professional questions with the certifying translator

The last item gates everything else.
