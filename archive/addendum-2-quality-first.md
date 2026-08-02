**Addendum 2 — Quality-First Revision**

Prepared 2026-08-01 · Supersedes: Addendum 1 §2 and §7

**1. The optimization target moves**

With wall-clock time removed as a constraint, throughput stops being a
design consideration at all. The correct metric becomes edits avoided
per page — how much correction the certifying translator does not have
to make. Any amount of additional computation is worth spending if it
removes even a handful of corrections from each page, because the
translator’s time is now the only scarce resource in the project.

This inverts several earlier recommendations. It also opens techniques
that were previously ruled out on cost grounds, and those are worth more
than the quantization decision that dominated the last revision.

**2. Quantization: revert to Q8_0**

The reasoning that produced the Q5_K_M recommendation was purely a
throughput trade. Without that trade, the argument runs entirely the
other way.

Quantization loss falls disproportionately on the linguistic patterns a
model has seen least — exactly the Slovene morphology this project
depends on. Q8_0 is where the loss curve flattens; below it, degradation
is measurable, and for a less-resourced language it is larger than the
general benchmarks suggest.

| **Quant**          | **Size** | **Speed**  | **Decision**                                                                                                              |
|--------------------|----------|------------|---------------------------------------------------------------------------------------------------------------------------|
| Q8_0               | ~13 GB   | ~4 tok/s   | Recommended. Effectively lossless; ample memory headroom remains                                                          |
| BF16 (unquantized) | ~24 GB   | ~2 tok/s   | Not recommended. Fits, but leaves under 3 GiB for context and desktop. Gain over Q8_0 is marginal; the memory risk is not |
| Q6_K               | 9.66 GB  | ~5–6 tok/s | Now a fallback only                                                                                                       |
| Q5_K_M / Q4_K_M    | 7–8.5 GB | ~6–8 tok/s | Withdrawn. No longer any reason to accept the loss                                                                        |

Worth confirming empirically all the same: translate one representative
passage at Q8_0 and at Q5_K_M and have the translator compare. If they
cannot tell the difference on legal register, that is useful information
about where the real bottleneck lies.

**3. Multi-pass translation — the substantive gain**

This is where free compute actually buys quality, and it matters
considerably more than the quantization choice above. A single
generation pass is the weakest possible use of an unconstrained compute
budget.

**3.1 Back-translation as an omission detector**

The failure mode most dangerous in this corpus is the fluent, confident,
wrong translation — a dropped negation, an omitted subordinate clause,
an inverted attribution. These are invisible on inspection of the target
text alone, because the target reads perfectly well.

Back-translation catches them. Take the translated output, translate it
back to the source language using a different model, and compare the
result against the original source. Meaning-preserving variation
produces paraphrase; meaning-destroying errors produce visible
divergence. Using a different model for the reverse pass is essential —
the same model will tend to reproduce its own misreading and confirm
itself.

Flag segments where the back-translation diverges materially from the
source, and hand the translator that list. This is the single
highest-value application of surplus compute in the entire workflow.

**3.2 Self-critique refinement**

Present the model with the source, its own draft, and the agreed
glossary, and ask it to identify and correct errors rather than to
retranslate. This reliably catches glossary violations and register
inconsistencies that a first pass misses, and it costs only another
pass.

**3.3 Proposed pass structure**

| **Pass** | **Model**            | **Function**                              | **Output**                  |
|----------|----------------------|-------------------------------------------|-----------------------------|
| 1        | GaMS3-12B Q8_0       | Draft translation with glossary injected  | Draft target text           |
| 2        | GaMS3-12B Q8_0       | Self-critique against source and glossary | Revised target text         |
| 3        | qwen3.6 35B-A3B      | Back-translation to source language       | Reverse text for comparison |
| 4        | Deterministic script | Divergence and consistency checks         | Flagged-segment report      |

Total cost is roughly three to four times a single pass. At Q8_0 that
puts a 500-page corpus in the region of four to six days of continuous
running — comfortably inside the stated tolerance.

**4. Deterministic checks cost nothing — run all of them**

Pass 4 above is not a model invocation. It is ordinary code, it runs in
seconds, and it catches a category of error that language models are
structurally bad at noticing. It should be built regardless of how the
rest of the pipeline develops, and it will likely repay more effort per
line than anything else here.

Checks worth implementing:

- Every number, date, and monetary amount present in the source appears
  in the target. Numeric drift is the highest-consequence error class in
  legal evidence and is trivially detectable by comparison.

- Case numbers, statutory citations, and institution names reproduced
  verbatim per the agreed policy, not translated.

- Glossary terms rendered with the agreed target term. Any deviation
  reported with document and segment reference.

- No empty, untranslated, or duplicated segments; no segment where the
  target is identical to the source unless policy requires it.

- Back-translation divergence above a threshold, from pass 3.

- The same source segment translated inconsistently across different
  documents — a corpus-level check no per-document review will surface.

The report from these checks is what the translator should receive
alongside the drafts. It converts an undifferentiated proofreading
obligation into a prioritized worklist, which is precisely the leverage
this project needs.

**5. Multi-day run hygiene**

A batch that runs for days introduces failure modes a short job does not
have. Three are worth engineering for before starting.

**Checkpointing and resumability**

The job must be resumable at segment granularity. A crash at hour sixty
must not restart from zero. Write each completed segment to disk
immediately, keyed by a stable identifier, and have the driver skip
anything already present on restart. This also makes the run auditable,
which matters given the material.

**Thermal and power**

A 15 W base-TDP mobile processor at sustained full load for days is
outside its typical duty cycle. Keep the machine on mains power,
elevated for airflow, and monitor package temperature. Holding the
battery at full charge continuously also accelerates its degradation; if
the firmware exposes a charge threshold, cap it for the duration.

> \# monitor while running
>
> watch -n5 'sensors \| grep -i package; ollama ps'
>
> \# cap battery charge if supported (path varies by model)
>
> cat /sys/class/power_supply/BAT0/charge_control_end_threshold
>
> echo 80 \| sudo tee
> /sys/class/power_supply/BAT0/charge_control_end_threshold

**Suspend and idle behavior**

Confirm the machine will not suspend mid-run. Lid-close and idle-suspend
defaults will silently halt a multi-day job.

> systemd-inhibit --what=idle:sleep:handle-lid-switch \\
>
> --why="translation batch" ./run_batch.sh

**6. Revised planning arithmetic**

At Q8_0 with the four-pass structure, generation cost per scanned page
rises roughly fourfold against the single-pass estimate in Addendum 1.

| **Scanned pages** | **Single pass** | **Four-pass structure** | **Practical schedule**       |
|-------------------|-----------------|-------------------------|------------------------------|
| 250               | ~12 hours       | ~2 days                 | One weekend                  |
| 500               | ~24 hours       | ~4 days                 | Under a week                 |
| 1,000             | ~48 hours       | ~8 days                 | Two weeks with interruptions |

*Excludes OCR, prefill, and the deterministic checks, which are
comparatively cheap. Prefill remains worth optimizing per Addendum 1 §6
even with time abundant, because a stable cached prefix is free to
arrange.*

**7. What does not change**

Abundant compute does not relieve any of the human-bound constraints,
and it is worth being explicit that the following remain exactly as
stated:

- The OCR text layer still requires human verification. Dual-engine
  cross-checking narrows the work; it does not eliminate it.

- The glossary still requires the translator’s decisions before bulk
  translation begins. No amount of computation substitutes for fixing
  the target terminology once, up front.

- The professional and procedural questions in Workflow Architecture §9
  still gate everything downstream, and remain the correct first action.

- Phase 1 remains the decision point. Three pages, hand-fed,
  translator’s judgment. If machine output is not materially faster to
  edit than translating fresh, running the machine for a week does not
  change that verdict — it only makes the wrong answer arrive more
  expensively.
