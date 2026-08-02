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

## Triage comes first

A client drop is not a clean Slovene corpus: it arrives as nested folder
trees mixing Slovene with English, Croatian/Serbian and more. `tr-inventory`
classifies every file by source language before anything is translated, and
`tr-run` then works only on the matching ones. Pushing a Croatian file
through an sl→en prompt wastes the inference *and* caches a wrong answer in
`work/tm.sqlite` that is reused silently from then on.

Its `manifest.tsv` and `by-lang/*.txt` list file paths, and a filename in a
criminal matter carries party names, dates and case numbers — so both are
case material. Only `summary.txt`, which holds counts and no paths, is safe
to send to the client.

## Open items (manual §14)

- Measure the 7,000-row spreadsheet: `tr-xlsx <file> --survey`
- Trial the container on a throwaway 1 GB image before real data —
  `case-init` has never run, so nothing is encrypted yet
- Decide what to do about the unencrypted disk. The finding is settled: the
  root filesystem is plain ext4, so the container is the only protection
- Decide which pages earn the vision cross-check, and test it on a real scan
- Establish how often the model completes a statutory provision from memory
- Seed the translation memory from the translator's prior work
- Verify German quality against base Gemma before extending to that pair
- Re-measure language detection on legal text; the quoted figures come from
  UDHR, which is thin and general-register (`tools/calibrate_lang.py`)
- Settle the professional questions with the certifying translator

The last item gates everything else.

## Two things a reader should know early

**Throughput is about five times slower than the planning documents assumed.**
Measured: 0.81 output tokens per second sustained, 48 seconds per segment.
Most of that is not generation — every call re-reads the system prompt,
about 28 seconds of every 48, whether the segment is a sentence or two
words. Manual §3.4.

**One error class defeats every automated check.** The model finishes famous
statutory provisions from memory, adding words the source does not contain.
It reads perfectly, carries no number or glossary term, and no prompt
prevented it. Segments citing a statute are flagged so a human reads them
against the source. Manual §8.1.

## Where things live

The kit is a git repository — `offline-translation-kit`, private on GitHub —
and these documents are `offline-translation-docs`. Clone the kit; do not
unpack it from an archive. A copy without `.git` cannot have the hook that
keeps client documents out of the repository.
