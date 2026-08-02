**Confidential Legal Translation — Offline Workflow Architecture**

Prepared 2026-08-01 · EN↔SL, later DE · Target machine:
hpelitebook8g1i16

**1. Headline finding**

Slovene has a purpose-built open-weight model, developed at the
University of Ljubljana specifically because general models underserve
the language. Its availability resolves the model-selection question and
displaces the earlier provisional recommendation.

It also relocates the project risk. Model choice is now the easiest
decision in this workflow. The three genuinely difficult problems are
optical character recognition of scanned evidence, terminology
consistency across roughly one hundred documents, and format-preserving
round-trip of the spreadsheet. Each of these will affect the certifying
translator’s editing burden more than the choice between one language
model and another.

> **Scope note.** This document addresses technical architecture only.
> Section 9 raises professional and procedural questions that fall
> outside it and should be resolved with the certifying translator
> before technical work begins, because a negative answer there would
> invalidate the entire approach.

**2. Revised model selection**

**2.1 Primary translator: GaMS3-12B-Instruct**

GaMS (Generative Model for Slovene) is developed by CJVT at the
University of Ljubljana under the PoVeJMo research program. The current
generation adapts Google’s Gemma 3 through three-stage continual
pre-training on approximately 140 billion tokens of Slovene, English,
Bosnian, Serbian, and Croatian text, followed by two-stage supervised
fine-tuning on over 200,000 examples.

Two published results make it the correct default here. It outperforms
the base Gemma 3 12B model across Slovene benchmarks, English-to-Slovene
translation, and the Slovene LLM arena; and in that arena it performs
comparably to the much larger commercial GPT-4o, with a win rate above
60 percent. English-to-Slovene translation was an explicit evaluation
target rather than an incidental capability.

The scale of the underlying problem explains why a dedicated model
matters so much. Available authentic Slovene corpora total roughly 40
billion tokens, against the 30 trillion tokens used to train a
contemporary frontier model — a ratio of about one to a thousand.
General multilingual models are not merely weaker on Slovene; they are
trained on a fundamentally different order of data.

**2.2 Quantization: choose quality over speed**

Quantization degrades low-resource language performance
disproportionately. The linguistic patterns that a model has seen least
often are the first to be lost when precision is reduced, and Slovene
morphology — six cases, dual number, aspectual pairs — is exactly the
kind of fine structure that suffers. Because this is an unattended batch
workload rather than an interactive one, throughput should be traded
away for accuracy.

| **Quant** | **File size** | **Est. speed** | **Assessment**                                                                         |
|-----------|---------------|----------------|----------------------------------------------------------------------------------------|
| Q6_K      | 9.66 GB       | ~6–7 tok/s     | Recommended. Near-lossless in practice, comfortable memory headroom for long context   |
| Q8_0      | ~13 GB        | ~5 tok/s       | Acceptable if a measured quality difference against Q6_K justifies the throughput cost |
| Q5_K_M    | 8.45 GB       | ~7–8 tok/s     | Reasonable fallback if Q6_K proves too slow in practice                                |
| Q4_K_M    | 7.30 GB       | ~8–9 tok/s     | Not recommended for certified legal output at this language pair                       |

*Available memory is 27 GiB, so even Q8_0 leaves ample room. There is no
hardware reason to economize here.*

**2.3 Supporting models**

| **Model**            | **Role**                   | **Rationale**                                                                                                                                                              |
|----------------------|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| qwen3.6:35b-a3b      | English-side utility       | Already installed and roughly four times faster. Use for glossary extraction, consistency checking of English output, and batch orchestration — not for Slovene generation |
| Gemma 3 / 4 12B      | German pairs, later phase  | German is high-resource and well served by the base Gemma family. GaMS continual pre-training may have eroded German relative to base Gemma; verify before relying on it   |
| Gemma 3 12B (vision) | Stamp and seal description | Gemma 3 is multimodal. Useful for producing the descriptive annotations that certified translations require for stamps, seals, and signatures                              |

Licensing differs across these and matters for a legal engagement. GaMS3
inherits the Gemma Terms of Use from its base model, which permit
commercial use but impose use restrictions; the Qwen models are Apache
2.0. If licensing must be defensible on the record, review both texts
rather than assuming equivalence.

**3. Pipeline architecture**

A single model invocation per document will not produce usable output
for this workload. The following five stages should be built and
validated independently, in order. Each stage has a distinct failure
mode and a distinct verification step.

| **Stage** | **Function**                     | **Principal risk**                                             |
|-----------|----------------------------------|----------------------------------------------------------------|
| 1         | Text acquisition (OCR)           | Silent character errors in names, dates, case numbers, amounts |
| 2         | Segmentation and project setup   | Loss of formatting and layout on reassembly                    |
| 3         | Terminology capture              | Inconsistent rendering of the same legal term across documents |
| 4         | Machine translation              | Fluent but unfaithful output; omission without warning         |
| 5         | Reassembly and quality assurance | Numeric and named-entity drift between source and target       |

**4. Stage 1 — OCR is the dominant risk**

This deserves more engineering attention than the translation stage. The
reason is a specific and under-appreciated interaction: a language model
presented with corrupted OCR output will not flag the corruption. It
will silently produce a fluent, plausible translation of the corrupted
text. A misread digit in a date, an account number, or a monetary amount
propagates into the target document looking entirely correct.

> **Consequence.** The OCR text layer must be treated as a verified
> artifact in its own right, checked by a human against the page images
> before translation begins. It cannot be treated as an intermediate
> that only the machine reads.

Slovene diacritics compound this. The characters č, š, and ž are
routinely mangled by OCR engines configured for English, and the
resulting words may still be valid Slovene, so no spell-check will catch
them.

Recommended stack, all offline:

> sudo apt install ocrmypdf tesseract-ocr-slv tesseract-ocr-deu
> tesseract-ocr-eng
>
> \# OCR while preserving the original page image underneath
>
> ocrmypdf -l slv+eng --rotate-pages --deskew --clean \\
>
> --output-type pdfa input.pdf output.pdf
>
> \# extract the text layer for verification
>
> pdftotext -layout output.pdf output.txt

Two practices are worth adopting regardless of tooling. First, keep the
page image alongside every translated segment so the translator can
check against the original without leaving the tool. Second, extract all
numbers, dates, and proper nouns into a separate checklist for targeted
verification, since these carry the highest consequence and the lowest
redundancy — context will not reveal an error in them.

If Tesseract output proves inadequate on poor scans, evaluate a modern
layout-aware alternative such as Surya before accepting the quality
loss. This should be tested on a representative sample of the worst
scans in the corpus, not the best.

**5. Stage 2 — Use a CAT tool, not bespoke scripts**

The instinct to build DOCX and XLSX round-trip handling directly is
understandable but is not the efficient path. Computer-assisted
translation tools already solve format-preserving extraction and
reinsertion, and they solve it better than a purpose-built script will,
because formatting inside a DOCX paragraph is fragmented across runs in
ways that are tedious to reconstruct correctly.

OmegaT is the appropriate choice: free, open source, fully offline,
cross-platform, and it handles DOCX and XLSX natively with formatting
preserved. It provides translation memory, glossary enforcement, and
segment-level review, which is precisely the shape of a
machine-translation post-editing workflow.

The decisive argument is not technical, however. The certifying
translator will need to review, edit, and attest this work. Delivering
it inside a standard CAT project rather than as loose files means they
can work in a familiar environment, the translation memory accumulates
across all one hundred documents, and their corrections propagate to
every later occurrence of the same segment. A bespoke pipeline delivers
none of that.

> **Integration caveat.** The mechanism for feeding a local model into
> OmegaT should be verified before committing. The robust and
> tool-agnostic approach is to pre-translate segments externally and
> import the result as a TMX translation memory, which requires no
> plugin and works with any CAT tool. Direct plugin integration may also
> exist but was not verified for this assessment.

**6. Stage 3 — Terminology is the main quality lever**

For legal translation, consistency of terminology matters more than
sentence-level elegance, and it is the dimension on which machine output
most reliably disappoints. The same Slovene term rendered three
different ways across three documents creates ambiguity that a court may
treat as substantive.

Build the glossary before bulk translation, not after:

- Extract candidate terms from the corpus — charges, statutory
  references, court and agency names, procedural terms, party
  designations, document types.

- Have the certifying translator fix the target rendering for each,
  once, up front. This is a short task that pays back across every
  document.

- Load the glossary into the CAT tool for enforcement, and inject the
  relevant subset into the model prompt for each segment so the model
  produces the agreed term rather than a synonym.

- Leave institutional names, statutory citations, and case numbers
  untranslated by policy, with the convention agreed in advance. These
  are the segments most likely to be silently and confidently
  mistranslated.

**7. Stage 4 — The spreadsheet**

A table of roughly 7,000 rows with up to five text columns implies on
the order of 35,000 cells. Translated naively at the throughput
estimated in Section 2.2, this would run for days.

It almost certainly should not be translated naively. Tabular data of
this kind is typically dominated by repeated categorical values —
statuses, document types, locations, party names, offence codes.
Deduplication before translation is likely to reduce the workload by an
order of magnitude, and it delivers guaranteed internal consistency as a
side effect, because each distinct string is translated exactly once.

Measure the redundancy before designing anything further:

> python3 - \<\<'EOF'
>
> import openpyxl, collections
>
> wb = openpyxl.load_workbook("source.xlsx", read_only=True)
>
> ws = wb.active
>
> cells = \[str(c.value).strip() for row in ws.iter_rows()
>
> for c in row if c.value and not
> str(c.value).replace(".","").isdigit()\]
>
> u = collections.Counter(cells)
>
> print(f"text cells: {len(cells):,} unique: {len(u):,} ratio:
> {len(u)/len(cells):.1%}")
>
> for s, n in u.most_common(15): print(f"{n:6,} {s\[:70\]}")
>
> EOF

If the unique ratio is below roughly 20 percent, build the translation
around a string map: extract unique strings, translate each once, write
results back by lookup. Output remains in XLSX, which is the correct
decision — ten columns will not transfer usefully into Word.

Free-text columns, if any, will need genuine segment-by-segment
treatment and should be identified and separated early, since they will
dominate the remaining cost.

**8. Confidentiality hardening**

The stated requirement is that the documents do not leave the premises.
The model choice satisfies this trivially, since local inference makes
no network calls. Two residual exposures are worth closing, because they
are more likely to cause a breach than the inference stack is.

**Full-disk encryption**

Case material on a laptop is exposed by theft, not by the network. If
the installation is not encrypted, that gap is larger than anything else
in this document. Verify:

> lsblk -o NAME,FSTYPE,MOUNTPOINT \| grep -i crypt
>
> sudo cryptsetup status /dev/mapper/\* 2\>/dev/null

Retrofitting encryption to an existing installation is disruptive. If it
is absent, an encrypted container for the case material alone is the
pragmatic alternative.

**Ollama cloud model routing**

Recent Ollama versions can route requests to hosted models as well as
local ones. A tag pulled or invoked carelessly could transmit document
text off the machine. Confirm that every model in use is local, and
consider blocking outbound network access for the Ollama process
entirely as a structural guarantee rather than a procedural one.

> ollama list \# confirm no cloud-tagged entries
>
> ollama ps \# confirm what is actually loaded
>
> systemctl show ollama -p Environment

A second structural measure worth considering: run the entire
translation pipeline with networking disabled at the container or
namespace level, so that the confidentiality guarantee does not depend
on configuration remaining correct.

**9. Questions outside the technical scope**

These should be settled before significant engineering effort is
invested, because an unfavorable answer to the first would render the
rest moot.

- Whether machine-translation pre-translation is permissible for
  court-certified translations in the relevant jurisdictions, and
  whether its use must be disclosed. Certification regimes vary, and
  some require the certifying translator to attest to personal
  translation rather than review. This is a question for the certifying
  translator and possibly for the court, not one to resolve by
  assumption.

- Whether the certifying translator is willing to work from
  machine-translated drafts at all. Post-editing poor output is slower
  than translating from scratch, and some professionals decline the
  arrangement on those grounds. Their agreement determines whether the
  project has value.

- Whether any evidentiary or disclosure obligation attaches to the
  intermediate artifacts — OCR text layers, translation memories, model
  outputs — in a pending prosecution. If so, they should be retained and
  version-controlled deliberately rather than discarded as scratch.

- Whether handling of the material on this machine is consistent with
  any protective order or undertaking already in force in the
  proceedings.

**10. Recommended sequence**

Do not build the full pipeline first. Validate the assumption that
machine output is faster to edit than translating from scratch, on a
small sample, before committing.

| **Phase** | **Activity**                                                             | **Exit criterion**                                                        |
|-----------|--------------------------------------------------------------------------|---------------------------------------------------------------------------|
| 0         | Resolve Section 9 questions                                              | Certifying translator confirms the approach is permissible and acceptable |
| 1         | Install GaMS3-12B; translate three representative pages by hand-fed text | Translator judges output materially faster to edit than translating fresh |
| 2         | Build and verify the OCR stage on the worst scans in the corpus          | Text layer accurate on names, dates, and numbers without correction       |
| 3         | Extract glossary; agree target renderings                                | Glossary fixed and loaded                                                 |
| 4         | Stand up OmegaT project; establish TMX pre-translation path              | One full document round-trips with formatting intact                      |
| 5         | Measure spreadsheet redundancy; build string map                         | Unique-string count known; approach chosen                                |
| 6         | Batch translation                                                        | Corpus pre-translated; hand off for certification                         |

Phase 1 is the decision point. If the translator does not find the
output helpful at that scale, the correct response is to stop, not to
build more infrastructure around it.

**11. Commands — immediate next steps**

Acquire the primary model. It is not in the Ollama library and must be
pulled from Hugging Face as a GGUF:

> \# Q6_K, recommended quantization (~9.7 GB)
>
> ollama pull hf.co/mradermacher/GaMS3-12B-Instruct-GGUF:Q6_K
>
> \# verify what arrived
>
> ollama show hf.co/mradermacher/GaMS3-12B-Instruct-GGUF:Q6_K

*Note that the repository publishes both static and imatrix-weighted
quantizations; either is acceptable, and exact tag syntax should be
confirmed against the model page at pull time.*

Take a baseline on real material, timing both prefill and generation:

> ollama run hf.co/mradermacher/GaMS3-12B-Instruct-GGUF:Q6_K --verbose
>
> \# suggested system framing for legal source text:
>
> \# "Translate the following Slovene legal text into English. Preserve
>
> \# paragraph structure. Do not translate case numbers, statutory
>
> \# citations, or institution names; reproduce them verbatim. Mark any
>
> \# illegible or uncertain passage as \[ILLEGIBLE\]. Output only the
>
> \# translation."

The instruction to mark uncertainty is not decorative. It is the only
lever available against the silent-hallucination failure mode described
in Section 4, and its effectiveness should be tested deliberately by
feeding the model a deliberately corrupted passage.
