# The PDF `--to-epub` route: bundle, reuse, and derived structure

> **Post-review corrections (added after the independent review of `report.md`, same session).**
> Two numbers in this note were re-measured and are wrong as written:
> 1. §6 "Verified by running" claims `test_bundle_pipeline`+`test_pdf_source_edit_guard`+… = `110p/151s`. Re-measured: `test_bundle_pipeline` is **18 passed / 48 skipped** and `test_pdf_source_edit_guard` is **0 passed / 22 skipped** (all 22 skip on `pandoc is not available on PATH`), so the pair is **18 passed / 70 skipped**. The per-file passed counts elsewhere in that row were not re-verified; treat the row as indicative, not as an audited tally.
> 2. The section-5 claim that `pdf_formula`/`pdf_figures` are "re-derived" stands, but a silent-degradation counterexample was found in the same area: `stages._installed_docling()` swallows every exception and returns `None`, which makes the provenance line (`EXTRACTION_REUSED_OTHER_RUNTIME`) print `unknown` — or, when the requested device also matches, not print at all — while the reuse still proceeds.
> Everything else in this note was either confirmed by the reviewer or not challenged.

Research note — technical design philosophy of the PDF route.

- Repo: `E:\File\self\github\bilingual_book_maker`
- Commit: `e1027a4823e98f668ef950740b6f805ec30bd3b2` (working tree clean at time of writing)
- Route introduced by `a6c8c2c` (feat(pdf): --to-epub, a bilingual reflowable EPUB with navigation from a PDF), revised by `a160326` (fix(pdf): never overwrite an edited source.md, provider identity by endpoint). Only these two commits touch `book_maker/pipeline/` — the subsystem is new and narrow.
- Method: static reading of the current implementation plus targeted offline probes. Every claim below cites a file:line that was read. Environment limits are listed in §6 and are material.

Labels used throughout: **[V]** verified fact (read in source, or reproduced by a command recorded in §7); **[I]** reasoned inference from several verified facts; **[U]** unknown / not verified here.

---

## 1. What the route is, from the outside

`make_book.py --to-epub book.pdf --model X --language zh-hans` runs four stages over one directory beside the PDF and leaves two things behind: `<stem>_book/` (a bundle kept on disk, editable) and `<stem>_bilingual.epub` (the deliverable). `book_maker/cli.py:2920` (`run_to_epub`) dispatches to `book_maker/pipeline/to_epub.py:139` (`pdf_to_epub`), which composes stages rather than implementing them:

```
                 to_epub.pdf_to_epub                      (to_epub.py:139)
   ┌───────────────────────────────────────────────────────────────────────┐
   │ 0. refusals before anything is read   find_pandoc, check_options,     │
   │    parse_pages, parse_ocr_lang, check_ocr_engine, image_request       │
   │    (to_epub.py:171-195)                                               │
   │ 1. prepare_stage = stages.prepare     (import | extract, reuse-aware) │
   │ 2. figure_stage  = pdf_figures.render_figures   (draw or reuse)       │
   │ 3. translate_stage = translate_bundle → the real BBM CLI (resumable)  │
   │ 4. export_stage = epub_export.export_epub       (always rebuilt)      │
   │ 5. copy bundle EPUB → <stem>_bilingual.epub, via mkstemp + os.replace │
   └───────────────────────────────────────────────────────────────────────┘
```

**Bundle layout** (`book_maker/pipeline/bundle.py:1-14`, verified against the code that writes it):

| Path | Written by | Role |
|---|---|---|
| `source.raw.md` | `importer.import_markdown` (`importer.py:100`) | extractor output / imported file, byte-for-byte |
| `source.md` | `importer.import_markdown` (`importer.py:101`) | the translation input; **the human-editable artifact** |
| `assets/images/`, `assets/figures/` | `importer._relocate_assets` (`importer.py:141`), `pdf_figures.clear_assets` (`pdf_figures.py:445`) | pictures shared by every Markdown version |
| `book_bilingual.md` | `reading_edition` loader via the CLI | the bilingual deliverable source |
| `book_bilingual.epub` | `epub_export.export_epub` (`epub_export.py:148`) | the built book |
| `manifest.json` | `Bundle.write_manifest` (`bundle.py:219`) | stage status, fingerprints, provenance, limitations |
| `.work/` | various | resume state, staging, `docling.json` snapshot, `figures.json`, `translate.state.json`, `translate.result.json`, overlay `decisions.json` |

**The manifest is a ledger, not a copy of the book** — `SCHEMA_VERSION = 1` (`bundle.py:27`) and a wrong version is refused with "use a new output directory" (`bundle.py:212-216`). Every write goes through a temp file + `os.replace` (`bundle.py:226-230`): "A half-written manifest is a bundle that can no longer say what it holds."

Two front doors share one core: `tools/pdf_to_book.py` (staged, per-stage, explicitly "Development tooling, not a stable interface") and `--to-epub`. `stages.py:1-7` states the reason: "One copy of this decision, because there are now two front doors to the same pipeline … A second copy would be a second set of resume rules, and the one that drifted would silently re-extract a book somebody had edited."

---

## 2. Mechanism inventory (file:line)

### 2.1 The reuse decision — `stages._verdict` (`stages.py:152-192`)

The whole of "may I reuse the extraction?" is eight comparisons, in order:

| # | Check | Line | Failing reason |
|---|---|---|---|
| 1 | a manifest exists | `stages.py:156` | `SOURCE_REASON_EXTRACT` / `_IMPORT` |
| 2 | `import` or `extract` recorded `completed` | `stages.py:160-166` | idem |
| 3 | `source.origin_sha256 == sha256_file(input_path)` | `stages.py:168` | "the PDF changed" |
| 4 | `extraction.provider == parser` | `stages.py:173` | "the parser changed" |
| 5 | `extraction.page_range == pages` | `stages.py:175` | "page range changed" |
| 6 | `ExtractionSettings.from_manifest(extraction).identity() == settings.identity()` | `stages.py:177` | "extraction settings changed" |
| 7 | if `--img-model` asked: `structure_status == "complete"` | `stages.py:185` | "the structure pass was {status} and is asked for again" |
| 8 | docling version / device — **reported, never fatal** | `stages.py:189`, `216-229` | prints `EXTRACTION_REUSED_OTHER_RUNTIME`, reuse still stands |

**Identity vs provenance** is the load-bearing distinction. `ExtractionSettings.identity()` (`pdf_settings.py:122-141`) returns exactly: `ocr`, `ocr_engine` (only if OCR), `ocr_mode` (only if OCR), `ocr_lang` (only if OCR), `table_mode`, `formula_images`, `structure`, `structure_rev` (only with a structure), `structure_base`. `pdf_settings.py:7-9` states the rule for device: "The models produce the same text on any processor, so the device is provenance — recorded, reported when it differs, never a reason to extract again." `stages.py:119-121` states the same for the docling version.

OCR fields collapse to defaults when OCR is off (`pdf_settings.py:131-133`) so that a run with OCR off is not invalidated by an engine or language nobody used — and `from_manifest` reads a missing key as its default "because that is what the runs from before the key existed did" (`pdf_settings.py:146-149`).

### 2.2 The edit guard — `importer.guard_source_replacement` (`importer.py:36-63`)

One invariant: an existing `bundle/source.md` may be replaced only while it still matches `source.working_sha256`. No stage status is consulted ("a failed extraction can sit beside a source someone has already edited", `importer.py:43-44`). A missing baseline is `SOURCE_BASELINE_UNKNOWN` — "unknown, not untouched" (`importer.py:46-48`). The baseline is recorded as soon as the file is written, not on success (`importer.py:102-105`), so a source that the preflight then refuses is still the tool's own text and may be replaced.

Called at four points, all *before* anything is mutated: `stages.prepare` (`stages.py:305`, before any print or write), `docling_parser.extract_pdf` entry (`docling_parser.py:948`, before `resolve_device`/`bundle.create`), `extract_pdf` again after the conversion (`docling_parser.py:1276`, "a conversion takes minutes in which the operator can edit it"), and `import_markdown` (`importer.py:94`). The refusal messages name the way out (`messages.py:41-54`).

The parallel guard on the other deliverable is `translate._refuse_to_clobber_edits` (`translate.py:502-512`): `book_bilingual.md` is compared against the hash recorded in `outputs.bilingual_markdown`; a mismatch raises `BilingualEdited`, which for `--to-epub` is re-worded to name the copy beside the PDF (`to_epub.py:230-236`, `translate.py:490-499`).

### 2.3 The extraction adapter — `book_maker/pipeline/docling_parser.py`

- `PARSER = "docling"` (`docling_parser.py:101`), duplicated as `stages.PDF_PARSER = "docling"` (`stages.py:43`) deliberately, with a test holding them equal, so a Markdown import never imports docling/torch (`stages.py:38-42`).
- Device resolution is delegated to docling's own `decide_device` (`docling_parser.py:161-176`); docling's single CUDA refusal is split into "no CUDA build" vs "no NVIDIA card" (`_unavailable`, `docling_parser.py:179-194`) because they need opposite fixes.
- Every docling option is set explicitly from `settings` — "including the ones that match docling's defaults today: what the pipeline asked for is then written down here rather than inherited from whatever a docling release ships" (`_converter`, `docling_parser.py:199-202`). Pictures are generated but never described (`generate_picture_images = True` at `:229`, and `picture_description: False` in the manifest at `:1527`). `do_formula_enrichment = False`, `do_code_enrichment = False` (`:235-236`) — "generated text is not source text".
- Page selection is parsed once as 1-based (`bundle.parse_pages`, `bundle.py:102-121`) and converted per engine (`zero_based_pages` `:124`, `one_based_pages` `:135`). A selection with a gap is converted as the covering run and trimmed afterwards, and the operator is told (`docling_parser.py:1057-1064`, `_selected_only` `:887`).
- OCR is off unless asked. A page the PDF cannot spell out is **refused** without it (`docling_parser.py:988-996`, `OCR_REQUIRED`), and the scan test is computed from the file itself by pypdfium2 (`pdf_common.text_layer_report`, `pdf_common.py:229-292`), including the "invisible text layer" case (`_invisible_only`, `pdf_common.py:85-105`).
- Page markers: docling's own breaks are only written *between* items, so an empty page gets none and every later page is numbered one short. `_export_pages` (`docling_parser.py:367-440`) therefore writes one segment per page from each item's `page_no`, and `_number_pages` (`:867`) stamps them; items with no page are written once after `<!-- unplaced -->` (`:429-439`).
- Heading levels: docling writes every heading, title included, as `##`. `pdf_headings.assign` (`pdf_headings.py:297`) decides levels from numbering first, typography second, then `pdf_headings.promote` (`:333`) lifts every heading one level. The module docstring records the measurement that justifies the replacement of docling's own model: 95.9% exact vs 24.6% (`pdf_headings.py:31-41`) — **[U]** not reproducible here.
- Formulas: `pdf_formula.mark` gives every undecoded formula a text marker with an index; the serializer writes the marker where the equation stood; the marker is then swapped for a picture cropped out of the page (`pdf_formula.py:86-114`, `287-320`, `apply` `:323`). "Placement is by identity, not by position" (`pdf_formula.py:16`). Overlapping regions merge so one equation is not shown twice (`merge`, `:171`). A region covering >80% of the page is not cropped, only warned about (`MAX_PAGE_SHARE`, `:72`, `:270-276`).
- Caching between runs is intentionally shallow: the staging directory `.work/extraction` is deleted and recreated at the start of every extraction (`docling_parser.py:973-976`), which removes the previous `docling.json` snapshot and `figures.json`. What survives is `source.raw.md`, `source.md`, `assets/`, the manifest, and the translate state. docling's snapshot is written for diagnosis only: "evidence for diagnosis and evaluation, read back by nothing in the pipeline" (`docling_parser.py:118-121`).

### 2.4 The region-role pass — `decisions.py` (opt-in, `--img-model`)

A vision model is shown a PNG of the page with every detector region boxed and tagged, and answers one JSON object mapping ids to labels from `{text, section_header, title, caption, footnote, code}` (`decisions.py:61-79`). Boundaries are enforced twice:

- candidate selection (`_is_candidate` `decisions.py:418-440`): only plain `TextItem`/`TitleItem`/`SectionHeaderItem`, only `ContentLayer.BODY`, only labels in `ELIGIBLE_SOURCE` (`:81-88`), not inside a table, not a formula marker, not already owned by a picture/table caption;
- acceptance (`ALLOWED`, `decisions.py:95-102`): lists, formulas, tables, pictures, furniture, key-value regions and document indexes "are neither asked about nor accepted" (`:92-93`).

Application is transactional per page: replacements keep the same `self_ref`, parent, children, provenance, text, `orig`, content layer (`_replacement` `:826-864`), are checked with `_intact` (`:867-879`), and `document.validate_tree(..., raise_on_error=True)` runs before the page's changes stand — otherwise the originals go back and the page is marked `failed_apply` (`apply` `:917-933`). A budget (`Budget.for_pages`, `:192-199`) is checked before every call; when spent, the remaining pages are recorded `unasked` and the detector's labels stand (`decide_roles` `:741-751`, `780-794`). A page where more than 60% of asked items changed is reported but its changes stand, with the reason recorded: the quarantine "helped on no page" (`:104-111`).

`PROMPT_REV`/`POLICY_REV` (`decisions.py:48-49`) enter the extraction identity, so a rerun under another revision re-extracts.

### 2.5 Figures — `pdf_figures.py`

Figures are redrawn from the PDF by our own renderer at a stated policy (`FigurePolicy`, `:100`; default `dpi:200`, `:149`) rather than used at docling's 72 dpi. Reuse is decided by `_still_drawn` (`:671-689`): policy, `FIGURE_REVISION`, the digest of the named figures' records, the absence of failures, and every output file's size **and sha256**. A recorded figure that `source.md` no longer references is skipped, not drawn and not counted (`_read_records` `:611-629`); a *named* figure the record does not cover stops the run (`FIGURE_RECORD_BROKEN`). A figure that cannot be drawn falls back to docling's picture under the same name, with a warning and a limitation, and is retried next run; with neither, the run stops (`:483-505`, `_fallback` `:702`). Pictures get their stable name `figures/p0003-01.png` on a copy of the document, never on `document` itself (`docling_parser._export_pages`, `docling_parser.py:412-414`; rationale at `:393-396`).

Deliberately **excluded from the translation fingerprint**: `translate._translated_assets` (`translate.py:330-348`) hashes every asset except drawn figures, "their pixels change with the figure policy alone — a redraw must not translate the book again".

### 2.6 Translation — `translate.py`

`translation_fingerprint` (`translate.py:314-327`) = sha256 over `{formatter version, sha256(source.md), hashed assets, option identity}`. `_already_translated` (`:515`) requires the stage completed *and* the fingerprint to match *and* the recorded output hash to still match the file. Otherwise `_resume_decision` decides whether to pass `--resume` to the CLI (`.work/translate.state.json`).

Crucially, `translate_bundle` does **not** reimplement translation: it calls the real `book_maker.cli.main` (`translate.py:431-436`) with a `--book_name` pointing at `bundle/source.md`, injecting only a loader subclass (`reading_edition_loader_class`, `reading_edition.py:505`) that changes the *shape* of the output, not the pairing logic (`reading_edition.py:1-23`).

The reading edition's shape rules: blocks separated by a blank line so source and translation stay two blocks (`reading_edition.py:93`); the source heading stays the only heading and its translation becomes an ordinary paragraph inside a fenced div, so "one chapter never appears twice in the TOC" (`:403-414`, `:251-271`); Pandoc's own heading identifiers are re-stamped onto the same headings so an internal link written against the source still resolves (`:357-374`; plumbed in from `translate.py:421-430` and bound at `reading_edition.py:505-522`). A completion record is written only on the success path, because the loader's interrupt path exits zero (`:465-472`).

### 2.7 Export and navigation — `epub_export.py`, `epub_nav.py`

`export_epub` (`epub_export.py:52`) runs Pandoc with `--toc-depth=6` — "The default of 3 drops an H4 and everything under it from the contents without saying so" (`:105-108`) — a `--split-level` computed from the document's own heading shape (`:85-90`), a CSS and a Lua filter (`:110-113`). Pandoc reports an unreachable image on stderr and still exits 0, so four fetch-failure phrases are treated as failures (`:43-49`, `:127-136`).

Nothing is published until `validate_epub` passes (`:138-148`): zip integrity, `mimetype`, container, at least one content document, XML well-formedness, a package document, image counts, and navigation.

`epub_nav.validate_navigation` (`epub_nav.py:45-60`) walks the route a reading system takes — container → package → the item with the `nav` property → the `nav` element with `epub:type="toc"` — and then compares the entries **against the outline Pandoc's own AST reported for the same document** (`preflight.Report.outline`). `_compare_to_outline` (`:190-229`) checks count, text, nesting depth and fragment anchors in one comparison, on the stated ground that "a truncated contents and a reordered one are the same kind of wrong book". `landmarks` is deliberately not followed (`:21-23`). Nesting tolerates heading-level holes by nesting under the nearest shallower heading (`nest`, `:232-248`).

Note `_pandoc_input` (`epub_export.py:316-352`): when everything above the first heading is front matter (page markers, images, short banners), it is *moved below* the heading for the conversion only. "Nothing is dropped, `book_bilingual.md` on disk is not touched." Prose above the first heading is left alone and the export then fails on the nav check — described as "the honest outcome until such a document gets a heading of its own" (`:331-333`).

### 2.8 Preflight — `preflight.py`

One Pandoc JSON-AST walk (`inspect` `:190`, `_Walker` `:211`) produces the report used by every stage: images, heading ids, headings, outline, preserved counts, problems. Refusals: control characters anywhere including code fences (`:69`, `:236-239`), reference-style images, HTML images, raw blocks other than HTML comments and HTML tables, unknown block types, duplicate heading ids, source fenced divs when translating (`:280-330`). Preserved (carried once, never translated): tables, code blocks, HTML tables. Pandoc is resolved and version-checked before any paid work (`find_pandoc` `:121`, `PANDOC_MIN_VERSION = (3, 1, 12)` in `messages.py:86`), "because it writes a table of contents the navigation check refuses, and that refusal would otherwise come after the translation".

---

## 3. Candidate technical design principles

Each principle below is stated, evidenced, then charged: what it costs, where it stops, and what contradicts it.

### P1 — Separate *identity* from *provenance*; only what decides the text invalidates a reuse. **[V]**

**Evidence.** `ExtractionSettings.identity()` (`pdf_settings.py:122-141`) vs the manifest's `extraction` block (`docling_parser.py:1521-1565`), which records identity fields *and* `version`, `device`, `device_requested`, `render_backend`, `page_range_sent`, `pdf_sha256`, `raw_document`, `cost_cents`. `stages._verdict` compares only identity + source hash + parser + page range (`stages.py:168-188`). Dynamic probe (§7, check 6): with a synthetic manifest recorded as `docling 2.129.0 / cpu`, changing the requested device to `cuda` and running under an install with no docling at all both still return `('extract', None)` — reuse — while changing `--formula-images`, `--pdf-ocr`, `--table-mode` or `--pages` each returns `False` with a distinct reason.

**What it solves.** The expensive, non-deterministic step (a model reading hundreds of pages) is not invalidated by things that cannot change the bytes it produced. Without the split, upgrading a library or moving from CPU to GPU would silently re-bill the whole book.

**Cost and limit.** The corollary is that a bundle extracted under docling 2.129 *is* reused under 2.130 and the old text stands. That is a deliberate accept: the mismatch prints once and tells the operator exactly how to force a re-extraction — `messages.py:230-234`: `"Reusing the extraction made with docling {old} on {old_device}; this run would use docling {new} on {new_device}. Delete the bundle directory to extract again."` The single printed line is the whole mechanism: `stages.prepare` prints the `lines` from the verdict and returns without writing anything (`stages.py:297-301`), and `tests/test_docling_adapter.py:1936-1953` asserts `out.count(head) == 1` — terminal output only. The manifest keeps `extraction.version`, so the *fact* is on disk, but the *fact that a later run reused it under a different version* is not added to `limitations`. **[V]** This is the sharpest limit on P1: reuse can serve text produced by a runtime the current run did not use, and only the terminal says so.

### P2 — Derived artifacts are re-derived from the raw input; only the raw extraction is durable. **[V]**

**Evidence.** Four independent instances: (a) figures are redrawn whenever policy, revision, record digest, or any output file's size/sha256 disagrees (`pdf_figures._still_drawn`, `:671-689`); (b) the EPUB is rebuilt on every run — `to_epub.pdf_to_epub` calls `export_stage` unconditionally after the translation (`to_epub.py:237`), and `export_epub` always writes to `book_bilingual.epub.part` then `os.replace`s (`:93-148`); (c) the staging directory holding docling's snapshot and `figures.json` is deleted and recreated at the start of every extraction (`docling_parser.py:973-976`); (d) heading levels are recomputed from the page on every extraction (`pdf_headings.assign`, `docling_parser.py:348`) rather than stored.

**What it solves.** A derived artifact can never be the thing that silently disagrees with its source: there is no second copy of the book to drift. The manifest's own docstring states the stance: "The manifest records run identity and recovery, **not a second copy of the book**" (`bundle.py:12-14`).

**Cost and limit.** Every rerun pays for the cheap derived steps. A run that changes only `--figure-policy` still redraws every figure (but specifically does *not* re-translate — `translate.py:330-348`). The design buys correctness of derivation with local recomputation, and pushes the caching decision up to the *expensive* steps only (extraction, translation), where it is guarded by fingerprints instead.

### P3 — Refuse rather than produce a plausible-looking wrong book. **[V]**

**Evidence** (each refusal names a concrete failure mode behind it):

| Refusal | Line | Failure it prevents |
|---|---|---|
| `OCR_REQUIRED` | `docling_parser.py:988-996` | a page with no text layer skipped as if translated |
| `check_recognised_text` → `OCR_EMPTY` | `pdf_common.py:394-411` | "completed" over a book that is pictures only |
| `OCR_REPLACE_ALL_EMPTY` | `docling_parser.py:1222-1230` | silently falling back to the replaced layer |
| `EXTRACTION_EMPTY` | `docling_parser.py:1231-1232` | a conversion that produced no prose |
| `preflight` problems | `preflight.py:236-330` | control characters, unsupported structures reaching the model |
| `FIGURE_RECORD_BROKEN` | `pdf_figures.py:622-629` | figures drawn wrong or not at all while the run looks fine |
| Pandoc fetch phrases | `epub_export.py:43-49, 127-136` | a book with a hole in it, exit 0 |
| `validate_epub` | `epub_export.py:169-216` | a truncated or unreadable container published |
| `validate_navigation` | `epub_nav.py:45-60` | a book with no usable table of contents |
| `guard_source_replacement` | `importer.py:36-63` | a hand edit destroyed by a rerun |

**What it solves.** Every entry in that table is a *silent success* the project has met: the code cites the incident ("9 of 12 bundles", "arXiv 2609.20519 page 1", "skill field test 260925"). The principle is not "fail safe" in general but specifically: a PDF→EPUB pipeline's plausible failure is to succeed loudly and be wrong.

**Cost and limit.** The price is measured in the same comments. `pdf_common.heading_for_top` (`:313-346`) exists because every page-1 paper failed at export *after the translation was paid for*; the fix was to move the refusal earlier **and add a repair** (insert an `# <title>` heading into `source.md`, where the operator can rename it) rather than relax the nav check. `_pandoc_input` moves front matter rather than dropping it or refusing. So the boundary is not "refuse always" but "refuse, unless a repair exists that keeps every byte and is visible to the operator."

The counterexample is `CHANGE_RATE_WARN`: a page whose region-role pass changed >60% of its items is *reported*, not quarantined, because the one time it fired the quarantine withheld a repair that was 9/10 right (`decisions.py:104-111`). Refusal is applied where the machinery can prove wrongness (structure, anchors, hashes, text-layer counts), not where it can only suspect it (a model's judgement).

### P4 — The expensive step is fenced by cheap checks that run first. **[V]**

**Evidence.** Three nesting levels of "refuse before paying":
- route: Pandoc, translation options, page spec, OCR language list, OCR engine, image endpoint — all resolved before a page is read (`to_epub.py:171-195`; the same order in `tools/pdf_to_book.py:255` and `:313` for `run`);
- route-owned stop rows are evaluated before extraction because `translation_argv` strips those flags from the inner run (`cli.py:2132-2145`, `cli.py:2934-2939`);
- extraction: `resolve_device` runs *before* `bundle.create`, so "an unusable device, or no parser installed at all, must not leave a half-made bundle directory behind" (`docling_parser.py:956-960`); `text_layer_report` runs before the models start (`:978-999`); `structure.vision()` is probed before the models start so an endpoint that cannot see a page is said once up front instead of failing after (`:1001-1033`); `check_ocr_engine` refuses an engine this install cannot run without importing it (`pdf_settings.py:43-75`).

**What it solves.** The bill arrives from two places (translation tokens, and the operator's time on a multi-hour extraction). Ordering refusals cheapest-first means a typo cannot cost an extraction.

**Cost and limit.** Ordering is load-bearing and therefore fragile-looking: the route's own flags must be stripped from the inner argv by an explicit list (`to_epub.py:46-70`), and a new flag that is not added there will reach the inner CLI. The comment at `to_epub.py:66-70` shows the maintainers aware of it: `--figure-policy` is owned here "so that when it comes it can never reach the inner run".

### P5 — The stage owns its artifact; the reuse decision is one function; the CLI is not forked. **[V]**

**Evidence.** `stages._verdict` is the only reuse rule and both front doors call it (`stages.prepare` `:289`; `tools/pdf_to_book.py:43` imports `already_prepared`); `docling_parser._replacement_reason` reaches back into `stages.reuse_verdict` (`:135-145`) rather than reimplementing the reason. `to_epub.pdf_to_epub` composes the four real stage functions as *parameters with real defaults* (`to_epub.py:156-159`, `:161-166`), so the routing is testable without a parser/model/Pandoc "and nothing else is injectable". `translate_bundle` calls the real CLI (`translate.py:431-436`).

**What it solves.** Two front doors that drift is the stated fear (`stages.py:1-7`); the guard against it is that there is one decision function and one translation entry point, with the CLI seam being a *loader subclass* rather than a patched global (`reading_edition.py:505-525`).

**Cost and limit.** The route must maintain a hard-coded list of options it owns (`OWNED_OPTIONS`/`OWNED_VALUE_OPTIONS`, `to_epub.py:46-70`) and a manual argv filter (`translation_argv`, `:73-94`) instead of using argparse remainders — `tools/pdf_to_book.py:63-73` explains why: "argparse cannot be trusted with this: REMAINDER swallows the wrong tokens as soon as an option before it takes a value." Competing explanation for this shape: it may simply be the smallest diff that reuses the existing 3,700-line CLI, not a principle. The evidence that it *is* a principle is that the shared decision was extracted into a separate module with its own justification rather than duplicated, and that `--to-epub` deliberately does not get its own translation loop.

### P6 — A human edit is the deliverable, not scratch space; the system refuses to compete with it. **[V]**

**Evidence.** `guard_source_replacement` (`importer.py:36-63`) with the baseline recorded at write time, consulted before *any* mutation, and — importantly — not gated on stage status. `_refuse_to_clobber_edits` (`translate.py:502-512`) for `book_bilingual.md`. The messages name the recovery path in one pasteable line (`messages.py:41-47`): rerun with the original settings, or translate the bundle as it is, or move the bundle aside.

**What it solves.** The route's product claim is that `source.md` is editable between extraction and translation (the extraction docstring says so: `docling_parser.py:1313-1316` records the limitation "inspect source.md before translating"). An edit guard is what makes that claim safe.

**Cost and limit.** An edited `source.md` plus *any* changed extraction setting is a hard stop: the operator cannot re-extract with a new setting over their edit; they must move the bundle. Verified dynamically (§7 check 6). The direction is deliberate — the refusal message says "Nothing was changed" and offers only non-destructive options.

---

## 4. Counterexamples, competing explanations, and things the brief assumed

**C1 — This route does *not* remove running heads, page numbers or apparatus.** The brief for this research assumed the pipeline decides such regions are apparatus and removes them. For the PDF route, **[V] it does not.** The complete list of textual edits applied to the raw extraction is:

1. page markers inserted (`docling_parser._number_pages`);
2. gapped page selections trimmed (`_selected_only`);
3. C0 control characters removed, counted and named per page (`strip_control_characters`, `docling_parser.py:1202`);
4. heading *levels* re-assigned and promoted (`pdf_headings.assign` / `promote`);
5. undecoded formula markers replaced by cropped pictures (`pdf_formula.apply`);
6. an `# <title>` / `# Page N` heading inserted when none is at the top (`heading_for_top`).

Nothing suppresses page numbers, running heads or copyright apparatus. The region-role pass explicitly does not ask about `page_header`/`page_footer`: they are not in `ELIGIBLE_SOURCE` (`decisions.py:81-88`) and appear on the overlay as context with an `x` tag (`_page_regions`, `decisions.py:452`; prompt text `:63`). The "skip apparatus" behaviour the brief describes lives on the *other* route — the `--plan`/Jev classification path referenced at `cli.py:778-781` ("running heads, page numbers and apparatus are exactly what the plan is for skipping") and `classifier.py:527` — and the PDF route's honest position is the recorded limitation: *"Extraction reading order, headings and diacritics are not verified by this pipeline; inspect source.md before translating"* (`docling_parser.py:1313-1316`).

**C2 — The reuse path bypasses the edit guard entirely, by construction.** `stages.prepare` returns on a `finished` verdict *before* calling `guard_source_replacement` (`stages.py:297-307`). Verified dynamically: a bundle whose `source.md` was hand-edited still returns `('extract', None)` from `reuse_verdict`, and `guard_source_replacement` on the same bundle raises. **[V]** This is correct behaviour (the edit is the point, and the source is not being replaced) but it means protection is attached to *the act of replacing*, not to the file. Consequence: the guard cannot be used as an "is this bundle edited?" query, and a future stage that writes `source.md` for another reason would need its own guard.

**C3 — The stage picks `done[0]`, the first of `("import", "extract")` recorded completed** (`stages.py:160-164`, `:192`). In a normal bundle exactly one of them is completed, so this is inert. If a bundle somehow had both, reuse would report `import`. Static observation only; not exercised. **[U]**

**C4 — `pdf_figures.render_figures` does not itself verify that the PDF matches the bundle's recorded origin.** Reuse is decided from records, file hashes and policy (`_still_drawn`, `:671-689`), and drawing reads whatever `pdf_path` it was handed (`_draw_all`, `:819`). The identity check lives in the callers: `tools/pdf_to_book._extracted_pdf` re-hashes the PDF and refuses (`:219-236`), and `to_epub` runs `prepare_stage` first, which would re-extract or refuse on `origin_sha256` mismatch (`stages.py:168`). So the invariant holds today but is enforced one level up — worth knowing when reading `render_figures` alone. **[I]**

**C5 — Competing explanation for the manifest.** The manifest could be read as a plain resume mechanism. The evidence for the stronger reading ("a ledger of what the run claims, including what it does not know") is that `limitations` is a first-class, appendable, removable list of *operator-facing uncertainty* rather than machine state — `add_limitations`/`drop_limitations` (`bundle.py:255-274`), limitations replaced per stage so "a line about the old languages or headings would otherwise outlive the text it described" (`docling_parser.py:962-967`), and `reading_edition._report_echoes` (`:445-463`) recording "these blocks came back identical to the original" as a limitation. A pure resume mechanism would not need any of that. The reading is well supported but not provable. **[I]**

**C6 — Framework default or deliberate choice?** `_converter` sets every docling option explicitly including current defaults (`docling_parser.py:199-208`). That is a *choice*, not inheriting a framework default. But the *abandonment* of the alternative (the Java route and its hybrid wrapper) is recorded as an evaluation outcome, not a principle: "the 260921 evaluation retired the Java route and its hybrid wrapper, which was docling behind a server that degraded its output" (`docling_parser.py:5-7`). One parser is a current fact, and care should be taken not to promote it to "single-parser philosophy" — the same file's `pdf_common.py:1-9` shows the *shared* rules were deliberately extracted so "a second copy of them inside an adapter would be a second set of answers", which anticipates another adapter more than it forbids one.

**C7 — `_verdict` treats a missing manifest key as its default rather than as unknown** (`pdf_settings.py:146-149`, `stages.py:200-207`). For extraction settings this is a deliberate "that is what the runs from before the key existed did". For `source.working_sha256` the opposite choice is made: missing baseline = refused (`importer.py:59`). So the same "missing data" situation is resolved per field, by what is recoverable: a default that provably matched past behaviour is adopted; an absent hash that *might* match is not assumed.

**C8 — Two hard-coded parser strings held equal by a test** (`stages.PDF_PARSER` `stages.py:43`, `docling_parser.PARSER` `docling_parser.py:101`). This is a deliberate duplication (to avoid importing docling on a Markdown run) with a test as the synchroniser — a DRY exception where the cost of the abstraction (an import side effect) is judged higher than the cost of the duplication.

**C9 — `test_pdf_dependencies.py` fails in this environment for a reason unrelated to the pipeline.** Two tests read `pyproject.toml` with `Path.read_text()` and no encoding (`tests/test_pdf_dependencies.py:48`), and this interpreter's `locale.getpreferredencoding(False)` is `cp936`; the file has non-ASCII bytes at offset 963-965 → `UnicodeDecodeError: 'gbk' codec can't decode byte 0xa6`. **[V]** Every `read_text(` call in `book_maker/` (12 of them) specifies `encoding="utf-8"` — verified by grep. So the failure is a test portability defect on a non-UTF-8-default Windows Python, not an implementation defect and not evidence about design philosophy.

---

## 5. How the technical choices show up as product behaviour

| Product behaviour a user can observe | Technical mechanism that produces it |
|---|---|
| A second `--to-epub` run over a finished bundle is fast and does not re-bill the extraction or the translation | `stages._verdict` + `translate._already_translated` (fingerprint) — `stages.py:152`, `translate.py:515` |
| Editing `source.md` between extraction and translation is safe, and is protected on later runs | `guard_source_replacement` + baseline at write time — `importer.py:36`, `:105` |
| A bundle made elsewhere/earlier is reused and the operator is told, once, with the recovery command | `EXTRACTION_REUSED_OTHER_RUNTIME` — `stages.py:216-229`, `messages.py:230-234` |
| Wrong or empty input fails with a sentence, not a traceback, and before money is spent | `PipelineError` per stage + `cli.run_to_epub` printing one line — `cli.py:2957-2964`; ordering in `to_epub.py:171-195` |
| The EPUB the reader opens always has a table of contents that matches the book | `validate_navigation` against Pandoc's own AST outline — `epub_nav.py:190-229`; publication only after validation — `epub_export.py:138-148` |
| The file beside the PDF is never a half-built book | build into the bundle, then `mkstemp` + `os.replace` — `to_epub.py:246-258`; `epub_export.py:93, 148` |
| Equations survive as pictures rather than being deciphered or lost | `pdf_formula.mark`/`apply` — `pdf_formula.py:86`, `:323`; formula enrichment off — `docling_parser.py:235-236` |
| Page selection is honest about cost: a gapped `--pages` prints the run it actually converted | `PAGES_SPAN_CONVERTED` / `_span` — `docling_parser.py:1057-1064` |
| A user who edits `book_bilingual.md` then reruns is told to export and copy, by exact command | `BilingualEdited.naming_copy` — `translate.py:490-499`, `to_epub.py:230-236` |
| **Not** offered: running heads, page numbers and apparatus are not stripped from the PDF route | C1 above; recorded limitation at `docling_parser.py:1313-1316` |

The same route also constrains the product: because extraction identity is a small typed value, a user cannot say "reuse the extraction but redo the figures at higher DPI" as *one* operation on a bundle with changed settings — the figure policy is deliberately outside the extraction identity (`pdf_figures.FigurePolicy`, `:100-149` is not an `ExtractionSettings` field), which is exactly what makes that combination work. Conversely, a user cannot re-extract with a new OCR setting while keeping an edit to `source.md`; that is P6's cost, and the refusal names the only non-destructive alternative.

---

## 6. Verified / unverified split

### Verified by running (this machine, this commit)

| Fact | How |
|---|---|
| `docling`, `docling-core`, `pypandoc` are **not importable**; `pandoc` is **not on PATH**; `pypdfium2 5.12.1` and `Pillow 12.2.0` are present | `importlib.metadata.version` for each; `pandoc --version` → not found |
| Focused offline tests pass: `test_pdf_headings` 24p/3s, `test_pdf_figures` 87p/17s, `test_pdf_formula` 24p/1s, `test_pdf_settings` 10p, `test_pdf_render` 8p/8s, `test_pdf_ocr_replace_layer` 5p/43s, `test_pdf_structure_wiring` 17p/9s, `test_epub_nav` + `test_pdf_to_epub_cli` 70p/29s, `test_bundle_pipeline`+`test_pdf_source_edit_guard`+… 110p/151s | `python -m pytest <files> -q -p no:randomly` |
| `test_pdf_decisions.py` (31 tests) is **entirely skipped**: module-level `pytest.importorskip("docling_core.types.doc")` at `tests/test_pdf_decisions.py:27` | `-q -rs` → `1 skipped` |
| Skips are `docling`/`docling_core` imports and `pandoc is not available on PATH` (`tests/pipeline_helpers.py:116`) | `-q -rs` skip summary |
| `reuse_verdict` keeps reuse when only the device changes, and when the recorded docling version differs from the installed one (none); it refuses on page-range, `formula_images`, `ocr`, `table_mode`, and structure change, each with its own reason string; and it does **not** consult the edit guard | standalone probe script (§7 check 6) |
| A hand-edited `source.md` is refused by `guard_source_replacement` with the `SOURCE_EDITED` text | same probe |
| `test_pdf_dependencies.py` fails with `UnicodeDecodeError: 'gbk' codec` reading `pyproject.toml` (non-ASCII at byte 963); locale encoding is `cp936` | `python -m pytest tests/test_pdf_dependencies.py` |
| All 12 `read_text(` calls under `book_maker/` pass `encoding="utf-8"` | grep count 12 vs 12 |

### Not verified — could not be run here

- **Any real extraction.** `docling` is absent, so nothing that calls `decide_device`, `DocumentConverter`, `PdfPipelineOptions`, `export_to_markdown`, `ContentLayer`, `TableItem` etc. was executed. All of §2.3, §2.4 and the docling-dependent parts of §2.5 are static reading.
- **Anything requiring Pandoc**: `preflight.inspect` on real documents, `export_epub`, `validate_epub`, `validate_navigation` end-to-end, the reading-edition loader, and `to_epub`'s copy-and-publish step. The `test_epub_nav.py` cases that build a synthetic archive directly (part of the 70 that passed) do exercise `validate_navigation` on constructed packages, but not on a Pandoc-produced book.
- **The measurements quoted in docstrings** (95.9% heading levels exact on a 20-paper corpus; `117 s` per page for formula enrichment; region-role token/latency figures). Treated as claims by the code's authors, not as findings of this note.
- **The end-to-end user journey** `PDB --to-epub` → `_book/` → `_bilingual.epub`. The composition is verified statically at `to_epub.py:139-264` and by the 70 non-Pandoc tests of that module, but no EPUB was produced here.
- **`_verdict`'s `done[0]` ordering** under a manifest with both `import` and `extract` completed (C3).
- **Behaviour of `decisions.apply` on a real `DoclingDocument`** — the transactional rollback path (`decisions.py:917-933`) is read, not run.

### Confidence

High confidence (multiple independent citations plus, where possible, execution): P1, P2, P3's inventory, P4, P6, C1, C2, C9. Medium confidence: P5 (the principle is stated in code but a simpler "smallest diff" explanation survives), C4, C5. Low / flagged: the corpus-level numbers in P3's and §2.3's docstrings, and anything about behaviour under docling versions other than the pinned one.

---

## 7. Rerunnable checks

All commands run from the repo root. Nothing below modifies project source; check 6 writes only into a temp directory and deletes it.

1. **Environment truth.**
   `python -c "import importlib.metadata as m; [print(n, (lambda: m.version(n))() if True else '') for n in ('docling','docling-core','pypdfium2','Pillow')]"` — note the `ImportError` path is the informative one; `pandoc --version`.

2. **The reuse table, statically.** Read `book_maker/pipeline/stages.py:152-192` and `book_maker/pipeline/pdf_settings.py:122-141`; confirm that every field `identity()` returns is the field set (`ocr`, `ocr_engine`, `ocr_mode`, `ocr_lang`, `table_mode`, `formula_images`, `structure`, `structure_rev`, `structure_base`) and that `version`/`device` appear only in `_other_runtime`.

3. **The identity/provenance test that pins it.**
   `python -m pytest tests/test_pdf_settings.py tests/test_docling_adapter.py -q -k "identity or reuse or version or device" -p no:randomly` — 3 pass, 15 skip without docling.

4. **Edit-guard invariant.**
   `python -m pytest tests/test_pdf_source_edit_guard.py -q -rs -p no:randomly` — note that every test needing Pandoc skips here, and read `tests/test_pdf_source_edit_guard.py:152`, `:238`, `:556` for the invariants the code is pinned to.

5. **Navigation vs outline.**
   `python -m pytest tests/test_epub_nav.py -q -p no:randomly` — 70 of the combined run's passes come from here and `test_pdf_to_epub_cli`; the corpus-shaped cases skip.

6. **Dynamic reuse probe** (the one that produced §3-P1's result). Build a synthetic bundle in a temp dir: `Bundle(root); create()`, write `source.md`, set `manifest.source = {origin_sha256: sha256_file(pdf), working_sha256: sha256_text(src), kind: "pdf"}`, set an `extraction` block, `set_stage("extract","completed")`, then call `stages.reuse_verdict(b, pdf, stages.PDF_PARSER, pages, settings, device, report=False)` for each variation. Expected: `('extract', None)` for unchanged settings, for a changed device, and for a docling version that does not match the installed one; `(False, "page range changed" | "extraction settings changed")` for the others.

7. **Textual edits to the raw extraction.** Read `book_maker/pipeline/docling_parser.py:1195-1247` and list every transformation; confirm no rule inspects a page header, footer or page number. Cross-check `book_maker/pipeline/decisions.py:81-88` (no `page_header`/`page_footer` in `ELIGIBLE_SOURCE`).

8. **Ordering of refusals.** Read `book_maker/pipeline/to_epub.py:167-227` and `book_maker/pipeline/docling_parser.py:936-976`; list, in order, the checks that run before the first model call and before the bundle directory is created.

9. **Manifest atomicity and limitation lifecycle.** `book_maker/pipeline/bundle.py:219-274`; then `docling_parser.py:962-970` (previous limitations and figure records dropped) and `1346-1348` (this extraction's limitations added and listed as its own).

10. **Confirm nothing was changed by this research.** `git status --porcelain` — the only new path is `design-philosophy/`.
