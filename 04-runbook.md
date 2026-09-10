**Runbook — the procedure, step by step**

Everything below runs in a **plain terminal with no Claude Code session
open**. The guards enforce that: `case-open` refuses while Claude is
running, and Claude refuses while the container is mounted. That is
deliberate — case material must never enter an assistant session.

Part 1 is the order of operations. Part 2 is the reference for every command,
generated from the tools themselves so it cannot fall out of step with them.

---

## Part 1 — The procedure

### Once per machine

| | |
|---|---|
| `tr-setup` | Packages, Python environment, dictionaries, git hooks. Idempotent. |
| `tr-model` | Registers the Slovene↔English model with Ollama as `gams3:q8`. |
| `tr-model hf.co/mradermacher/EuroLLM-9B-Instruct-2512-GGUF:Q8_0 eurollm9b-2512:q8` | Only for English↔German matters: registers the model for that pair. About 10 GB. |
| `case-init 40G` | Creates the encrypted container. Once, ever. Put the passphrase in your password manager and copy `~/.case/header.bak` onto a USB stick — a corrupted LUKS header loses the data even with the right passphrase. |
| `tools/install-desktop-guard.sh` | Routes the desktop launcher through the guard. The PATH wrapper only covers the terminal. |

The header backup only changes when a passphrase is added, changed or
removed. Writing data never touches it, so one copy stays valid until you
change the passphrase.

**Then check that the boundary is real:**

    case-status

It reports whether each guard is actually installed, not merely whether the
container is closed. Both must read `installed`. This matters because the
guard is what makes the boundary structural rather than remembered — but
installing it is itself remembered, and that is the part that has failed in
practice: the terminal guard was in place, the desktop guard was not, and
the desktop application opened with case material mounted. `case-status`
used to say "Safe to start Claude Code" throughout, because it was reporting
the mount and assuming the rest.

### Per matter

**1. Open the container and create the project.**

    case-open
    tr-project --new kranj-2024

**For any pair other than Slovene→English, set it now.** `tr-project --new`
writes Slovene→English into the project's `project.conf`. For an
English→German matter, edit that file so these three lines read:

    TR_SRC=en
    TR_TGT=de
    TR_OCR_LANGS=eng

Nothing else is set. The model follows the pair — GaMS3 for
Slovene↔English, EuroLLM for English↔German — and `tr-run` refuses to start
when that model is not installed, instead of writing a failed translation
for every segment. Set the pair before step 3: triage keeps only the files
in `TR_SRC`, and counts only those toward the volume.

**2. Copy the client's drop into `source/`, preserving its folder
structure.** Filenames and the shape of the tree are reproduced in
`translated/`, so the structure you create here is the structure you
deliver. Do not flatten it.

**3. Classify every file by source language.**

    tr-inventory

A drop is not a clean corpus. It arrives mixing Slovene with English,
Croatian/Serbian and whatever else, and pushing a Croatian file through an
sl→en prompt wastes the inference *and* caches a wrong answer that is reused
silently from then on.

Read `work/inventory/by-lang/unknown.txt`. Files not in the source language
are out of scope for work and for billing.

**Correcting a language, so it stays corrected.** Edit the `lang` column in
`work/inventory/manifest.tsv`, and set that row's `method` column to
`manual`. A row marked `manual` is never re-detected — every later run
refreshes its size, word count and segments but leaves the language alone.

Without the marker your correction still survives, but as a preference
rather than a decision: the next run that re-reads the file will notice the
detector disagrees and **ask** before changing anything. The recorded
language always wins by default, including when you answer nothing, because
the detector's alternative verdict is often `unknown` — which would drop the
file out of the translation set entirely.

`--rescan` re-examines every file but does not discard your corrections; it
proposes, like any other run. `--accept-revisions` takes the detector's
verdict everywhere without asking, which is for scripts, not for a drop you
have curated.

**4. OCR everything and count the words.**

    tr-inventory --count --with-ocr

This is the volume baseline the quote rests on. It reads every file in full
rather than sampling, and has `tr-pdf` make the text layer of every PDF —
the same layer, made the same way, that translation will use — so the OCR
cost is paid once. It writes:

| File | What it is | Safe to send? |
|---|---|---|
| `work/inventory/manifest.tsv` | Every file, language, words, segments | **No** — paths carry party names |
| `work/inventory/summary.txt` | Counts only | Yes |

Two things about the numbers. Source words is the billing unit; segments
govern machine time, which is a different question. And **spreadsheets are
counted for words but not for time** — a sheet of cells has no sentences to
segment, so the "machine time" figure excludes them. On a corpus with
spreadsheets the real figure is much higher than the one printed; run
`tr-xlsx --survey` on each sheet for its unique-string count, which is what
actually governs the work there.

**Not every PDF is a scan.** A born-digital document — exported from a word
processor rather than photographed — already carries exact text, and `tr-pdf`
uses it directly rather than rasterising and re-reading it, which could only
introduce errors that were never in the document. It says so:
`born-digital: 412 words of real text, no OCR needed`.

A PDF carrying somebody else's OCR layer is *not* treated as born-digital.
Tesseract writes its invisible text in `GlyphLessFont`, so its presence means
the text is OCR of unknown quality, and the file is re-read here — where at
least the confidence of each word is recorded.

**5. Check whether the OCR is good enough to build on.**

    tr-ocrstat

Worst file first, with a verdict against measured thresholds: under 5% of
tokens unreadable proceed, 5–20% look at the marked pages first, 20% or more
stop and get better copies. Exits non-zero if anything is in the stop band,
or if any text layer cannot be measured.

**A text layer made by an earlier `tr-inventory --with-ocr` cannot be
measured.** That version wrote the text with plain `pdftotext`, so nothing
in it is marked `OCR_ILLEGIBLE` — and `tr-pdf` reused it, so neither did the
translation. `tr-ocrstat` names those files instead of reporting them at 0%,
and step 4 reads them again, keeping the old copy as `<name>.txt.unmarked`.
A project already translated from such layers was translated without
unreadable words marked; re-running step 4 and then `tr-run` there
retranslates the affected PDFs.

**6. Read what it flags.** For every file in the `look` or `STOP` band, open
its text layer against the page images:

    less work/ocr/<name>.txt
    xdg-open work/ocr/<name>.ocr.pdf

The name is the source path with `/` replaced by `__`. What you are looking
for is what is marked `OCR_ILLEGIBLE`: if a hand-filled amount or a case
number sits inside one, that value has to come from a person.

**7. For pages whose numbers carry weight, read them twice.**

    tools/ocr-check.py source/<file>.pdf --pages 2

This is the step `tr-ocrstat` cannot do for you, and the distinction matters:
**`tr-ocrstat` measures legibility, `ocr-check.py` measures correctness.** A
confidence floor catches handwriting and damage. It cannot catch `12.450,00`
read as `1248000` — that scores 86% and reads as a fact all the way to the
deliverable. Only two engines disagreeing finds it. A file at 0% unreadable
is legible, not verified.

**8. Translate.**

    tr-run

`tr-run` works from the **manifest**, not from a directory walk — so a file
added to `source/` after the last `tr-inventory` is invisible to it. Re-run
step 3 whenever you add anything. `tr-run` now names any file it finds on
disk that the manifest has never seen, rather than leaving it out silently.

Resumable at two levels: it skips files already delivered, and within a file
every segment already in the memory is reused. Interrupting it costs at most
one segment. It re-translates a file whose source is newer than its output,
or whose output was produced by a different model or a superseded prompt.

**9. Get the reviewer's worklist.**

    tr-lint

No model, seconds. Reports numbers dropped or invented, non-translatables
altered, glossary terms not used, and segments returned unchanged. This is
what the translator works from, not the raw draft.

**10. Harvest terminology.**

    tr-terms --min-count 3 --pin-all

Writes `glossary/candidates.tsv` — terms the model rendered more than one
way, and frequent terms worth pinning before they drift. The translator
picks one rendering per line; the survivors go into `glossary/project.tsv`,
or `_shared/glossary/base.tsv` if they are general legal vocabulary that
should outlive this matter.

**11. Re-run.** Pinning a term invalidates only the segments containing it,
so this is minutes, not hours:

    tr-run
    tr-lint

**12. Deliver from `translated/`, then close.**

    cd ~ && case-close

`cd` first, and not out of tidiness. A shell whose working directory is
inside the container keeps the filesystem busy exactly as an open file does,
so closing from within the project directory fails — and `-f` does not help,
because the kernel counts a working directory as a use of the mount.

Closing unmounts the container; it does not empty it. Everything stays
inside the container file, encrypted, and returns at the next `case-open`.
The corollary is retention: a finished matter occupies the container at full
size until someone opens it and deletes the project directory by hand. No
script removes client work.

### The order that matters

Two changes invalidate the translation memory very differently, and it
decides what you can afford to do when:

| Change | What re-runs |
|---|---|
| A glossary term | Only the segments containing that term |
| The prompt text | **Everything** translated for the pairs whose prompt changed |

So a prompt change belongs before a full run, not after. On a twenty-hour
corpus that is the difference between minutes and starting again.

---

## Part 2 — Command reference

Generated from the tools. If a flag is here it exists; if it exists it is
here.

### `case-close`

unmount and lock the encrypted case container.

    Usage:  case-close [-f]

### `case-guard`

wrapper that refuses to launch Claude Code while the case container is mounted.

### `case-guard-desktop`

the same refusal as case-guard, for the desktop app.

### `case-init`

create the encrypted case container. Run once.

    Usage:  case-init [size]        default 40G

### `case-open`

unlock and mount the encrypted case container.

    Usage:  case-open

### `case-status`

is the case container open, and is it safe to start Claude Code?

### `tr-docx`

translate a .docx, preserving paragraph and table structure.

    Usage:  tr-docx <input.docx> <output.docx> [--from sl] [--to en]

| Flag | Meaning |
|---|---|
| `--from` | — |
| `--to` | — |

### `tr-fixtures`

generate synthetic Slovene legal documents for development.

    Usage:  tr-fixtures [outdir]      default: ./fixtures

### `tr-hwsurvey`

collect the hardware facts that determine which local LLMs a machine can realistically run. Read-only; installs nothing.

    Usage:  tr-hwsurvey            (prints to screen)

### `tr-inventory`

inventory every file in source/ and detect its language.

    Usage:  tr-inventory [--rescan] [--no-ocr] [--limit N]

| Flag | Meaning |
|---|---|
| `--rescan` | re-examine files already in the manifest |
| `--no-ocr` | do not sample-OCR scanned PDFs; mark them instead |
| `--limit` | stop after N files, in walk order. For trying the run on a large drop before committing to it |
| `--count` | read every file in full and count words and segments, for estimating the volume of work |
| `--accept-revisions` | take the detector's verdict wherever it disagrees with the recorded language, without asking |
| `--with-ocr` | with --count, run the real OCR pass on scanned PDFs instead of listing them as needing it. Slow, and cached so tr-pdf does not repeat it |

### `tr-lint`

deterministic checks over the translation memory and outputs.

    Usage:  tr-lint [--tsv report.tsv] [--all-versions]

| Flag | Meaning |
|---|---|
| `--tsv` | — |
| `--all-versions` | include segments from superseded models and prompt versions (default: only what tr-run would use now) |

### `tr-model`

register the translation model under a stable short name.

    Usage:  tr-model [hf-tag] [name]

### `tr-ocrstat`

how readable is the OCR, file by file, before anything is translated.

    Usage:  tr-ocrstat [--min-pct N]

| Flag | Meaning |
|---|---|
| `--min-pct` | only list files at or above this unreadable percentage |

### `tr-ocrtext`

OCR text layer with unreadable tokens marked OCR_ILLEGIBLE.

    Usage:  tr-ocrtext <input.pdf> <output.txt>

### `tr-pdf`

OCR a scanned PDF, then translate the text into a .docx.

    Usage:  tr-pdf [--ocr-only] <input.pdf> [output.docx]

### `tr-project`

list, show, or switch the active confidential project.

    Usage:

### `tr-run`

translate everything in source/ that is not yet in translated/.

    Usage:  tr-run                 # translate what tr-inventory found in TR_SRC

### `tr-setup`

one-time provisioning. Safe to re-run.

### `tr-status`

what is translated, what is missing, what is stale.

    Usage:  tr-status [--missing]      # --missing prints bare paths only

### `tr-terms`

find terms the model rendered inconsistently, and propose entries.

    Usage:  tr-terms [--min-count N] [--top N] [--pin-all] [--write]

| Flag | Meaning |
|---|---|
| `--min-count` | a term must appear in this many segments (default 3) |
| `--top` | how many candidates to report (default 25) |
| `--write` | append accepted lines to the project glossary |
| `--pin-all` | propose an entry for every frequent term with a clear rendering, not only the ones already disagreeing |

### `tr-txt`

translate a plain text file into a .docx deliverable.

    Usage:  tr-txt <input.txt> <output.docx> [--from sl] [--to en] [--bilingual]

| Flag | Meaning |
|---|---|
| `--from` | — |
| `--to` | — |
| `--bilingual` | — |

### `tr-xlsx`

translate a spreadsheet via unique-string deduplication.

    Usage:  tr-xlsx <input.xlsx> <output.xlsx> [--from sl] [--to en]

| Flag | Meaning |
|---|---|
| `--from` | — |
| `--to` | — |
| `--cols` | — |
| `--survey` | — |

### `tools/calibrate_lang.py`

measure trlang detection accuracy at a given sample length.

    Usage:  calibrate_lang.py <corpus-dir> [--lengths 25,40,60,100] [--stride N]

| Flag | Meaning |
|---|---|
| `--lengths` | comma-separated sample lengths in words |
| `--stride` | word step between samples (default: no overlap) |

### `tools/cycle-test.sh`

exercise a full container cycle with a throwaway project.

### `tools/gen-docs.py`

write the documented tables from lib/registry.py.

    Usage:  gen-docs.py [--apply]      (default: check only, exit 1 on drift)

| Flag | Meaning |
|---|---|
| `--apply` | write the files; without it, report drift only |

### `tools/highlight-docx.py`

colour the command blocks in the operating documents.

    Usage:  highlight-docx.py <file.docx> [more.docx ...] [--apply] [--strict]

| Flag | Meaning |
|---|---|
| `--apply` | write the files; without it, only report |
| `--strict` | also repaint blocks that merely differ from these rules, not just ones painted a single colour |

### `tools/install-desktop-guard.sh`

route the desktop launcher through case-guard.

### `tools/ocr-check.py`

is the OCR good enough to build a translation on?

| Flag | Meaning |
|---|---|
| `--pages` | — |
| `--no-vision` | — |
| `--gate` | skip the vision pass where fewer than this percent of Tesseract words were doubtful (default 3). 0 disables the gate and checks every page |

### `tools/phase1-setup.sh`

prepare the Phase 1 comparison on a real subset.
