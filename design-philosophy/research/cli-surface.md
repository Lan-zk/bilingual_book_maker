# CLI surface, option resolution, and the product control model

Research note for `task-1`. Dense facts first; interpretation last.

- **Subject**: `E:\File\self\github\bilingual_book_maker` @ `e1027a4823e98f668ef950740b6f805ec30bd3b2` (`e1027a4823e9`, 2026-09-30).
- **Environment**: Python 3.12.3, pytest 7.4.3, package not pip-installed; pytest run from repo root.
- **Commands actually run** (results quoted inline):
  - `python -m pytest tests/test_flag_compat.py -q` → `146 passed in 167.54s`
  - `python -m pytest tests/test_cli_documentation.py tests/test_legacy_cli.py -q` → `80 passed in 6.70s`
  - `python make_book.py --help` → 534 lines / 36 490 chars
  - `python make_book.py --book_name test_books/animal_farm.epub --plan-dry-run` (hermetic PYTHONPATH) → plan table + `Classifier: …` / `Image model: off` / B3 warning
  - `python make_book.py --book_name test_books/animal_farm.epub --api_format groq --key sk-test --model m --batch` → A1 refusal, exit 1
  - `python make_book.py --book_name test_books/animal_farm.epub --model gemini --gemini_key k --test` → two `deprecated:` notices + B6 warning
  - AST/`grep` counts listed under each inventory item. **One file was created by the dry run (`test_books/animal_farm_plan.json`, gitignored) and was deleted again.**
- **Not done**: no full-suite run (one was already running elsewhere); no network calls; no modification of project source. Everything below is static reading plus the cheap runs above. Line numbers are from this commit.

---

## 0. Shape of the surface

`make_book.py` is 4 lines: `from book_maker.cli import main` (make_book.py:1). Everything a user can type lives in `build_parser()` (cli.py:2182–2856).

| Fact | Value | How verified |
|---|---|---|
| `add_argument` calls in `build_parser` | 70 | AST walk of `build_parser` |
| option strings | 70, all flags (no positional args) | same AST walk |
| long options the docs test requires | 67 | `tests.test_cli_documentation._long_cli_options()` |
| options hidden with `help=argparse.SUPPRESS` | 4 | AST keyword scan |
| options whose help starts with "deprecated" | 1 | AST keyword scan |
| inline help strings vs imported constants | 51 inline / 14 imported | AST keyword scan |
| `--help` size | 534 lines, 36 490 chars | `--help \| Measure-Object -Line -Character` |
| argument groups / epilog / mutually-exclusive groups | 0 | grep for `add_argument_group\|epilog\|formatter_class` → no matches |

Suppressed options: `--plan-classify-model` (cli.py:2408–2414), `--batch_units` (2480–2487), `--with-ocr` (2567–2572), `--no-gpu` (2573–2578). The one `deprecated` help is `--poetry-group-size` (2334–2344).

Naming/grouping conventions (read, not guessed):
- `allow_abbrev=False` with the stated reason "`--model` must not resolve to `--model_list`" (2184–2185).
- Long options are the preferred spelling; short forms exist for `-m/--model` (2217–2219) and `-p/--proxy` (2263–2265).
- Two spellings of one flag are declared in one `add_argument` when neither is deprecated (`--key, --api_key` 2193–2203; `--glossary, --terminology` 2705–2719). Deprecated spellings get their own suppressed argument instead, because argparse would otherwise advertise both in `--help` (`DeprecatedAlias` docstring, 731–738; used at 2480).
- A leftover section banner `########## ENDPOINT ##########` survives at 2192 — the only one in the file (grep). So options are *not* grouped for the reader in `--help`: 67 long options arrive as one flat list.

Alias/retirement mechanisms (three distinct ones, by age):
1. `DeprecatedAlias` (731–742) — same `dest`, records the spelling typed into `{dest}_deprecated_flag`; `deprecation_notices` (2024–2043) turns that into a sentence. Used by `--batch_units` (2480).
2. Value aliases mapped at run time, with a notice: `--plan-classify most` → `all` (3001–3003); `--plan-classify-model` → `--classify-model`, new name wins (normalize_options, 1992–2000).
3. Retired PDF spellings rewritten after parsing: `PDF_ALIASES` + `retire_pdf_aliases` (2889–2917), an explicit `--device` beating `--no-gpu` (2915–2916).

---

## 1. Resolution layer — what the tool computes instead of asking

**No interactive input exists anywhere in the package**: `Select-String` for `input\(|sys\.stdin` over `book_maker/**/*.py` returns nothing. The product is a one-shot batch command; every question is answered by a flag, an environment variable, a config file, or an inference function.

Endpoint resolution (`resolve_endpoint`, 366–422) in precedence order:

| Step | Behaviour | Line |
|---|---|---|
| 1 | Refuse `--model` + `--model_list` together: "Name the model once" | 377–381 |
| 2 | A single model name that is in `ROUTE_DICT` **is** a route (`--model orcarouter`), and takes its own key variable | 388–390 |
| 3 | `--model_list codex` refused with "use `--api_format codex` instead" | 395–403 |
| 4 | `--provider` fills only what the command left out | 405, 291–350 |
| 5 | `--api_format` else inferred from base host, else from the model id | 410–412, `infer_api_format` 198–213 |
| 6 | the format's official base is filled in so *downstream* code sees the real address | 418 |
| 7 | a pasted request path is trimmed (`normalize_api_base`, 221–239) with `customapi` exempted | 419 |
| 8 | the format's default model fills in when none was named | 420–421, `DEFAULT_MODELS` 181–185 |

Key resolution (`resolve_api_key`, 248–274): explicit `--key` → legacy env vars carried by a rewritten command → `BBM_API_KEY` → vendor variable (`FORMAT_ENV_KEYS`, 139–149) → `"local"` for a localhost endpoint (266–267) → hard `SystemExit` naming every variable it looked in, for the nine formats that need one (`FORMATS_REQUIRING_KEY`, 154–166). Order is deliberate: legacy keys come first "so with both `OPENAI_API_KEY` and `BBM_GROQ_API_KEY` exported, a groq command must not hand the OpenAI key to Groq" (251–255).

Provider shorthand (`apply_provider`, 291–350) is a pure precedence rule — "a provider is a shorthand for flags, so every flag actually typed outranks it" (294–295) — plus two guards: the entry's key variable is only consulted while the run still calls the entry's address (314–325, prints a warning when it does not), and on the codex route the entry's model list is ignored because the sidecar resolves its own model (333–342).

Mode resolution:

- `resolve_context_mode` (836–844): `--use_context` bare = `window`; `session` only when typed. Returns `(flag_was_given, mode)`.
- `resolve_classify_mode` (939–960): naming a classifier on an epub *is* naming the mode it belongs to, except `all`/`agent` which "ask nothing" (`NEVER_CLASSIFIES`, 1757).
- `resolve_plan_mode` (874–936): `--plan-classify auto` means `none` unless the book is an epub, the user did not type `--translate-tags`, and either the wire format is `openai` (852) or the route can hold a classifier conversation. On an OpenAI-shaped route elsewhere (groq/xai/litellm/gateway) the answer is decided **at run time by a live capability probe**, not from the command line — the docstring says so and the message says so (896–907, `PLAN_VERDICT_REASONS` 862–866).
- `plan_mode_expected` (968–986) and `session_run_expected` (1012–1025) exist so the *table* can reason about the run without running it. `session_run_expected` deliberately reads one attribute both the CLI and the loader read: "so the rows that talk about a session cannot disagree with the run that has one" (1018–1021); `tests/test_flag_compat.py:1265–1281` asserts `session_run_expected(facts) == loader._session_run` for every format.
- `normalize_options` (1977–2021) is the only place "typed" vs "default" is derived. Several flags parse to `None` specifically so that difference survives argparse (2310 comment: "None, not "p": a typed selection opts out of the automatic plan, even `--translate-tags p`"; 783–787 for the plan-shaping flags).

Reasoned inference (not directly proven): this whole layer is the reason the project can afford 67 options — the cost of each flag is pushed into a resolution function with a stated precedence order, so combinations do not multiply in `main`.

---

## 2. Refusal layer — one declarative table, plus a lot that is not in it

`CompatRule = namedtuple("CompatRule", "id level when say")` (1009). The header comment (989–1007) states the contract: one table, one pass, run after the endpoint is resolved and before any translator is built; `id` is "the audit row it enforces, for the tests to name; never printed"; `level` is `stop` (red, exit 1) or `warn` (yellow); stops win and suppress warnings.

**Measured composition** (`python -c` over the imported module):

- `COMPAT_RULES`: **51 rows — 10 stops, 41 warns**; ids unique.
- Stops: `A1, A4, A7, A10, A12, A13, A14, A15, A16, C9`.
- Warns: `A8, A9, A11, B6, B8, B9, B10, B11, B12, C1–C8 (except C13–C15), C10, C11, C12, C16, C18–C37`.
- `DRY_RUN_RULES`: 4 warns (`B2`, `B3`, plus `C31:dry-run`, `C32:dry-run` constructed by re-labelling two table rows, 1969–1973).
- `PDF_ROUTE_STOP_ROWS = ("A16",)` (2135) — the one stop the `--to-epub` route must ask itself, because it strips its own flags before the inner run sees them (2132–2134, `run_to_epub` 2934–2939).

Rule taxonomy by what the sentence is for (read from the 51 `say` lambdas):

1. **Would spend money and do nothing / do the wrong thing** (stops A1, A4, A7, A10, A13, C9). Example, A1 verbatim: "`--batch / --batch-use` are broken on epub: queueing lives on a path the epub loader never takes, so the run translates the whole book live at full price and then submits an empty batch job — and `--batch` never writes the book at all. Drop the flag." The sentence names the mechanism, the money, and the next keystroke.
2. **Would be accepted and silently ignored** (the warn majority: C1–C8, C10–C12, C16, C18–C37). Each names the flag, the reason on *this* route or book type, and what to use instead.
3. **Degraded quality at the same price** (A8: grouping off means one request per paragraph, each re-reading the whole history; A11: the codex classifier costs "~17k tokens of its own preamble").
4. **A deprecated spelling/route in use** (A9, C12).
5. **Two flags that contradict each other** (A14/A15: a base without its model; B10: `--only_filelist` defeats `--exclude_filelist`; C23: `--no_disclosure` silences `--translation-metadata`).

Three of the 51 rows do not own their text: A14/A15/A16 quote `IMG_BASE_WITHOUT_MODEL`, `CLASSIFY_BASE_WITHOUT_MODEL` (endpoints.py, 88/92) and `OCR_REPLACE_NEEDS_OCR` (pipeline/messages.py:168). The last one is the same object the pipeline raises as its own error (pipeline/stages.py:92), so the sentence a user reads *before* the PDF route starts is literally the one they would read from inside it.

**Counterexample to "one table, one pass".** `cli.py` contains **27 `exit(1)`/`raise SystemExit` sites** (regex over the file). Only 2 belong to the table mechanics; the rest are hand-written refusals interleaved through `main()`, and at least five of them run *before* `check_compatibility` is called at 3270:

| Refusal | Line |
|---|---|
| `--batch` on a route without the Batch API (`SUPPORTS_BATCH_API`) | 3172–3180 |
| `--use_context session` on a format that does not implement it | 3185–3194 |
| `--parallel-workers` on codex | 3198–3204 |
| `parallel_session_conflict` (workers + one history) | 3208–3210, 2868–2881 |
| `--parallel-workers` + `--use_context` on a route with no per-chapter context | 3216–3228 |
| `--plan-classify` on a non-epub book | 3496–3502 |
| `--model` required / not supported for the format | 3584–3590, 3612–3619 |

So the accurate description is: **the table owns the combinations that depend on the resolved route or book type; capability-attribute checks and post-parse wiring checks stay inline.** Whether that split is principled or accreted is not decidable from the current implementation.

Second counterexample: some flags are still silently inert. `--pdf_layout` is only forwarded to a PDF loader (3371–3372) and has **no** compatible-shaped row anywhere (grep: 3 occurrences, all three cited here). `--interval` is only applied on gemini (3594–3595) and likewise has no row; both are handled in help text instead ("PDF output layout for PDF inputs", 2511; "every other route ignores it", 2791–2793). Mitigation for the user is prose, not the table.

**Provenance of the row ids.** The header says an id is "the audit row it enforces, for the tests to name" (995). The ids in the table are a subset of a wider numbering: `A3`, `A5`, `A6`, `B1`, `B4`, `B5`, `B7`, `C13`, `C14` appear only as labels on tests outside the table (`tests/test_loader_compat_guards.py:130,415,521,574`; `tests/test_flag_compat.py:518,541,1544,1571,1592`), and `A2`, `C15`, `C17` do not occur anywhere in this checkout (case-sensitive grep over `tests/`, `book_maker/`, `README.md`, `docs/en/`). The dated audit documents the comments cite (`docs/260923-docs-WIKI_MODERNIZE.md`, `docs/260920-feat-CLASSIFY_LADDER_PROMPT_AUDIT.md`, …) are **not in the repository** (`Test-Path` → False for all five checked). Consequence: the numbering is meaningful to the maintainer's local notes, and a reader of the public repo cannot resolve an id back to its requirement.

**Argument-shape refusals** form a third, earlier layer — `type=` callables that reject a value at parse time, each with the same "what to type instead" style: `coverage_fraction` (804–833, "50% is 0.5, not 50", and warns at both self-defeating ends), `accumulated_tokens` (745–761), `poetry_group`, `batch_unit_cap`, `compact_budget` (657–672, names `MIN_COMPACT_BUDGET` and offers plain `--use_context` instead), `glossary_file` (675–689, checked at parse time "before an endpoint is resolved or a byte of the book is read"), `classify_min_confidence` (790–801), `language_arg` (87–99).

**Prompt-template validation** is its own refusal family (`_unfillable` 436–442, `_unusable` 445–449, `check_user_placeholders` 452–483). The reason is stated as a timing argument: a bad placeholder "used to surface from inside the translator, on the first request, after the book had been opened and the endpoint paid" (456–458). The fields are read out of the template rather than inferred from what `str.format` throws, because the exception depends on the shape (460–466).

---

## 3. Preview layer — saying what will happen before it costs anything

| Mechanism | What it does | Line |
|---|---|---|
| `language_guidance` | one line when `--language` matched no known tag, offering both fixes | 116–133, printed at 3009–3011 |
| `deprecation_notices` / `legacy.notices` / `retire_pdf_aliases` | what was rewritten, in the user's own spelling | 2024–2043, 3039–3040, 3020–3021 |
| `prompt_adoption_line` | which `--prompt` sections this run adopted **and where each one went**, or that a route builds no prompt to carry it | 591–631, printed at 3398–3400 |
| `preview_endpoint` | runs `resolve_endpoint` **on a copy** so the preview neither rewrites the run's flags nor inherits a provider's model | 1805–1831 |
| `endpoint_preview_lines` | `Classifier: …` / `Image model: …` from the same resolvers with `with_key=False`; a choice the run would refuse is printed as `refused: …` instead of stopping | 1838–1888 |
| `dry_run_plan_divergence` + `DRY_RUN_RULES` | prints where the preview can differ from the real run | 1891–1974 |
| `--plan-dry-run` block in `main` | builds and prints the partition, writes `<book>_plan.json`, prints "nothing here is decided yet", returns before any credential is read | 3042–3154 |

Verified by running the dry run: the preview printed `Classifier: gpt-6-luna at the openai endpoint's default host (run)`, `Image model: off`, and B3's "An endpoint that verifies JSON mode but not a strict schema carries half of each, so the real run can make more requests than these" — then `existing plan … kept (may carry your edits); delete it to regenerate` (3136–3140). No credentials were present in the environment.

The interesting part is the honesty of the preview. It is allowed to be wrong in stated directions: B2 enumerates the two ways the run may not be in plan mode at all (1914–1924), B3 says the request count is a floor, and the docstring of `dry_run_plan_divergence` records that the preview used to forecast a run nobody makes (1899–1904). `tests/test_flag_compat.py:1518–1541` pins the parity: for every format, "each turn plan mode off" in the preview must agree with `resolve_plan_mode`'s own answer.

---

## 4. Guardrail tests — the contracts the surface is held to

| Test file | Contract | Notes |
|---|---|---|
| `tests/test_flag_compat.py` (1625 lines) | Every warn row has a fixture with the phrase the operator reads; every row id is unique; stops are exercised through a real subprocess; a noise guard asserts a plain epub/txt/md/MT run trips **nothing** and prints nothing | `test_every_warn_row_has_a_fixture` 1289–1293; `TestNoiseGuard` 934–995; `TestRowsFollowWhatTheLoadersRead` 998–1085 exists because four rows had drifted from the loaders |
| `tests/test_cli_documentation.py` (188 lines) | **Two-way parity between argparse and the user docs.** Long options are read from `book_maker/cli.py` by AST and must all appear in `README.md`, `README-CN.md`, `docs/en/cmd.md` (69–75); conversely, 18 reference files — including `.agents/skills/bbm-plan/**` and `docs/en/*.md` — may name no flag the CLI does not have (121–188) | losers are the 4 `SUPPRESS`ed options and the 1 help starting with "deprecated" (`_is_suppressed`, 20–34). It also pins that `--help` renders at all, because argparse `%`-interpolates help strings (78–97) |
| `tests/test_prompts_documented.py` (269 lines) | Every prompt sent to a model is a hard-coded literal in the test (114–209), the reviewable copy in `docs/…PROMPTS_FOR_REVIEW.md` must contain the same text, and six retired phrasings must not exist anywhere in the package (251–269) | doc half skips when the dated note is absent (220); the tracked half runs in CI. Stated intent: "Revising a prompt is meant to update this file in the same commit — that is the point, not friction to route around" (7–9) |
| `tests/test_classify_prompt_pin.py` (70 lines) | The two classifier instruction texts are pinned byte for byte, captured at `ed369fe` | fails on a one-byte change to what an endpoint is sent |
| `tests/test_classify_prompt_documented.py` (100 lines) | Same two-guard shape for the classify prompt, plus "the verdicts the prompt asks for are the ones it accepts" | |
| `tests/test_loader_compat_guards.py` (636 lines) | The rows the CLI cannot decide — resume fingerprint (A5), codex-is-a-session (B1), what a `--test` slice covers (B7), classification not truncated by `--test` (A3) | |
| `tests/test_pandoc_version.py` (106 lines) | Pandoc older than 3.1.12 is refused **by name with the download page**, because the nav validator would otherwise fail after translation was paid for | |
| `tests/test_docker_pins.py` (158 lines) | The Dockerfile's CUDA base tag must name the lock's torch; stages install the files they claim | |
| `tests/test_pdf_dependencies.py` (123 lines) | The base install carries nothing the PDF route needs; the `pdf` extra declares *what*, never *which* torch build | |
| `tests/test_docs_to_wiki.py` (238 lines) | The docs→wiki converter: anchors, links, nav coverage, refusals | |

Two facts worth separating from the tests' subject matter: (a) the CLI contract tests run the real CLI *except* the network call — `tests/hermetic/sitecustomize.py` replaces the `google` translator at interpreter startup so "a proxy hiccup … failed tests that have nothing to do with translation quality"; (b) there are 120 top-level string constants in `book_maker/pipeline/messages.py` whose docstring claims "Every string a person reads on this path lives here, verbatim, so the wording is owned in one place", and 27 in `book_maker/endpoints.py`. The CLI does not follow that convention fully: 51 of 70 help texts are inline literals in `build_parser`.

---

## 5. Legacy surface — kept, rewritten, and narrated

`book_maker/legacy_cli.py` (359 lines) rewrites old command lines before argparse sees them (`translate_legacy_argv`, 188–359; called from `main` at 2990). Its docstring states two rules: "**Say what happened**" and "**Never guess**" — "An alias with no faithful translation exits with an error naming it. Quietly mapping an unknown model name onto some default would bill a whole book to a model nobody chose." (8–15).

Verified behaviour (run above): `--model gemini --gemini_key k` printed
`deprecated: --gemini_key is now --key` and `deprecated: --model gemini is now --api_format gemini --model gemini-flash-latest`.

Mechanics worth noting: nine per-vendor key flags are recognised (80–90); which key to honour is chosen *after* the alias is known, "Preferring position would hand vendor A's key to B" (210–216); modern flags win and the notice says so, including naming what the user's own flag kept (196–202, 343–353); an alias that is also a live model id is passed through untouched when `--api_format` is explicit (`_ALIAS_IS_ALSO_A_MODEL_ID`, 60–64, 242–243); an Azure base is rewritten to `/openai/v1` (307–321); an unfaithful combination (alias route + different explicit format + that vendor's key flag) is a hard failure with both remedies named (262–269). `tests/test_legacy_cli.py` (486 lines, 9 test classes) covers untouched commands, presets, vendor routes, keys, endpoint flags, faithfulness, notices, OrcaRouter, codex.

Where the docstring overstates: it says "Every flag that change removed is still accepted here", but retired **model values** were deliberately dropped from the alias tables (`gpt4`, `gpt5mini`, `o1`, `o1mini`, `o1preview`, legacy_cli.py:26–28) and now travel verbatim to the endpoint's own model check. So it is flag-level compatibility, and alias-level best effort.

---

## 6. Candidate product-design principles

Each item: **what it looks like** → **problem it solves** → **cost** → **counterevidence / scope**.

### P1. The run is *resolved*, not interrogated

- **Look**: zero `input()`/`stdin` in the package; every decision is a named pure-ish function with a documented precedence order (endpoint 366–422, key 248–274, context 836–844, plan 874–936, classify 939–960, defaults 1977–2021).
- **Problem**: a book is a long, paid, non-restartable job. Stopping mid-run to ask a question is the failure mode; so is deriving the same fact twice and disagreeing.
- **Cost**: the user must trust and read printed derivations; the code needs `plan_mode_expected`/`session_run_expected` and a preview layer to explain itself, and it needs guards so two derivations agree (test_flag_compat.py:1265–1281).
- **Counterevidence / scope**: `--plan-classify auto` on an OpenAI-shaped route is *not* decidable from the command line — a live capability probe settles it (952, 968–975, `PLAN_VERDICT_REASONS` 862–866). The codex route is a session whether or not the flag was typed (1012–1025). So "the tool decides, never asks" holds as a product stance, not as a guarantee that the decision is predictable offline.

### P2. Refuse before spending, and make the refusal instructive

- **Look**: 10 stops + 41 warns; stops exit 1 and silence all warnings ("a warning about a run that is not going to happen is noise in front of the reason it isn't", 1006–1007, 2122–2127); plus 9 custom argparse `type=` validators (besides `str`/`int`/`float`: `language_arg`, `coverage_fraction`, `poetry_group`, `accumulated_tokens`, `batch_unit_cap`, `classify_min_confidence`, `compact_budget`, `glossary_file`, `parse_pdf_image_dpi`) and prompt-template validation.
- **Problem**: the expensive resource is the translated book, and several flag combinations fail *after* paying (A1, A4, A6, A10 are all described that way in their own sentences).
- **Cost**: 51 hand-written sentences become a user-facing compatibility surface nobody can change casually; every warn row needs a fixture; a new flag now has to be considered against route × book-type × mode.
- **Counterevidence**: the "one table" framing is only partly true — 27 exit sites, of which the table is 10; `--pdf_layout` and `--interval` are silently inert off their route with no row. Whether the table is complete is *not* claimed anywhere in the code, and it is not.

### P3. Say what you decided, before it costs anything — in the user's own words

- **Look**: `prompt_adoption_line` (591–631) reports adoption *and destination* per section, including "your `{text}` carries the batch JSON on a grouped request"; `endpoint_preview_lines` (1838–1888) prints the run's real resolvers without a key; `--plan-dry-run` (3042–3154) writes the plan, prints "nothing here is decided yet", and says what each later mode will do to the rows; `GlossaryPath` (692–701) records which of `--glossary`/`--terminology` was typed so the warning names it (`_glossary_flag`, 1143–1145).
- **Problem**: a resolved tool must make its resolution legible, or the user cannot tell a misconfiguration from a decision.
- **Cost**: a second code path that must mirror the run (`dry_run_plan_divergence` exists only to admit where it differs), plus tests pinning the mirror (test_flag_compat.py:1518–1541, 1502–1508).
- **Counterevidence**: the preview is deliberately optimistic in one direction (B3: real run may make *more* requests). The `--plan-dry-run` on a non-openai route must *guess* the grouping budget because no translator exists to measure the prompt overhead (3083–3095) — and prints that guess.

### P4. Old command lines keep working, and the user is told what they became

- **Look**: `legacy_cli.py`'s two stated rules; verified notices on a real run; modern flags always win; an alias with two readings fails loudly rather than picking one (262–269); `--plan-classify most`, `--plan-classify-model`, `--batch_units`, `--with-ocr`, `--no-gpu` all still parse.
- **Problem**: the CLI was rewritten once (`4d864d7 refactor(cli): choose a translator by endpoint, not by model name`), and scripts/CI exist that nobody will update by hand.
- **Cost**: two surfaces to keep coherent; alias tables must be pruned when vendors retire models, and the pruning is a judgement call (26–28).
- **Counterevidence**: model-value aliases were pruned; the compatibility is one-directional (old → new) and does not survive a second rewrite of the same flag name.

### P5. User-facing prose is a tested artifact, and one sentence has one owner

- **Look**: `test_cli_documentation.py` compares argparse against three references in both directions and checks 18 reference files (including the agent skill) for invented flags; `test_prompts_documented.py` pins each sent prompt as a literal and forbids six retired phrasings anywhere in the package; `OCR_REPLACE_NEEDS_OCR` is one object used at cli.py:1224 and pipeline/stages.py:92; `messages.py` holds 120 operator strings with the stated reason.
- **Problem**: two of this tool's audiences — a human and a coding agent following `.agents/skills/bbm-plan/SKILL.md` — both break on a stale flag name; a half-replaced prompt is worse than either version (test_prompts_documented.py:249–250).
- **Cost**: a prompt or flag edit now requires touching tests and docs in the same commit; the reference list is hand-maintained (121–141) and can silently omit a new doc.
- **Counterevidence / limit**: the check is textual (`re.search` on the option string), so a reference can mention a flag and describe it wrongly; and the CLI itself keeps 51 help strings inline, so "one owner for prose" is true of the pipeline and of the docs tests, not of `build_parser`.

### P6. A decision only a human or an agent can make is handed over as a file contract, then the run stops

- **Look**: `--plan-classify agent` writes the plan JSON with samples and prints a self-contained block to paste into a coding agent, then stops before translating (`loader/classify/agent.py`:1–13, 38–106). "The plan JSON is the contract … There is deliberately no editing CLI to learn". The printed rerun command is reconstructed from the user's own argv, with a key replaced by its environment variable and `--extra-headers` values masked (`epub_loader.py:1767–1782`). The plan is fail-closed: `ledger.require_decided` (368–382) refuses to translate while any row is `null`, and `ledger.load` (424–452) rejects a plan whose `book_sha256` or `schema_version` does not match the book.
- **Problem**: deciding which of a book's tag signatures is apparatus is a judgement no cheap rule makes, and the wrong default silently drops or mistranslates text.
- **Cost**: that path cannot run unattended; the operator must edit JSON (or hand it to an agent); `agent` mode never pre-fills, which is a stated owner decision (`NEVER_CLASSIFIES` comment 1751–1756) not an emergent property.
- **Counterevidence**: not all decisions are handed over — `--plan-classify auto` on a probed route decides by itself, and `all` decides by refusing to decide ("translate the whole partition"). The hand-over is one of four modes, not the model.

---

## 7. Unverified / unknown

- **The audit behind the rule ids.** `A2`, `C15`, `C17` appear nowhere in this checkout; the dated documents the comments cite do not exist here. The requirement each id enforces is therefore unknown from the repository, and I did not reconstruct it.
- **Whether the table is complete.** I confirmed two flags (`--pdf_layout`, `--interval`) are silently inert off their route without a row. I did not enumerate every flag × book-type × route combination, so the true size of the "silently ignored" class is unmeasured.
- **Why the seven capability-attribute refusals live inline rather than in the table.** Two readings are consistent with the code: (a) they depend on translator class attributes a stop-`when` predicate could read anyway, so the split is historical; (b) the table is deliberately scoped to combinations that need no instance. Nothing in the code distinguishes these.
- **Real user behaviour.** No user research, telemetry, or issue data was consulted. Every "problem it solves" above is inferred from comments and code structure, not from observed users.
- **The grouping-budget defaults** (`SESSION_BUDGET_FLOOR`/`CEILING`, `GENERAL_GROUP_MAX_UNITS`, `MIN_COMPACT_BUDGET`) are *documented in code as owner choices with partial measurement* (634–654 explicitly says "there is no measured cost knee here to point at … 1500 is where the owner drew that line"). I did not re-derive or re-measure them; that is `task-3`'s territory.
- **`prompt_md.json`, `plan.log`, `test_books/`** contents were not inspected; whether they are fixtures or leftovers is unknown.
- **The dated-doc workflow** (`docs/26MMDD-*.md` local notes pinning prompts and plans) exists in tests as a `pytest.skip` path; I could not run the half that compares against those notes because the notes are absent.
- **No full test-suite result was seen by me**; only the three focused files above (226 tests total) were run.

---

## 8. Checks a third party can rerun

```powershell
cd E:\File\self\github\bilingual_book_maker   # commit e1027a4823e9

# 1. Rule composition (expect: 51 rules; 10 stop, 41 warn; 4 dry-run rules; A16 route-owned)
python -c "from book_maker.cli import COMPAT_RULES as C, DRY_RUN_RULES as D; import collections; print(len(C), collections.Counter(r.level for r in C)); print([r.id for r in C if r.level=='stop']); print([r.id for r in D])"

# 2. The refusal/warn contracts, the noise guard, and the loader/CLI agreement
python -m pytest tests/test_flag_compat.py -q            # 146 passed (~2m50s)

# 3. Two-way parity between argparse and the user-facing references, plus the legacy rewrite
python -m pytest tests/test_cli_documentation.py tests/test_legacy_cli.py -q   # 80 passed (~7s)

# 4. Option counts (expect 70 add_argument calls, 70 option strings, 67 documented after SUPPRESS)
python -c "import ast,sys; sys.path.insert(0,'tests'); from test_cli_documentation import _long_cli_options; t=ast.parse(open('book_maker/cli.py',encoding='utf-8').read()); f=[n for n in ast.walk(t) if isinstance(n,ast.FunctionDef) and n.name=='build_parser'][0]; c=[n for n in ast.walk(f) if isinstance(n,ast.Call) and getattr(n.func,'attr','')=='add_argument']; print(len(c), len(_long_cli_options()))"

# 5. The preview layer, offline and credential-free; then DELETE the plan file it writes
$env:PYTHONPATH="$PWD\tests\hermetic"
python make_book.py --book_name test_books/animal_farm.epub --plan-dry-run
Remove-Item test_books/animal_farm_plan.json   # gitignored artifact; remove after checking

# 6. A stop rule and a legacy rewrite, observed end to end
python make_book.py --book_name test_books/animal_farm.epub --api_format groq --key sk-test --model m --batch
python make_book.py --book_name test_books/animal_farm.epub --model gemini --gemini_key k --test

# 7. No interactive input exists
Select-String -Path book_maker/*.py,book_maker/*/*.py -Pattern "[^_a-zA-Z]input\(|sys\.stdin"   # no output
```

To verify a single claim: the "stops silence warnings" contract is `tests/test_flag_compat.py::TestStops::test_a_stop_silences_the_warnings` (461–483); the CLI/loader session agreement is `::TestTheCodexThreadIsASessionWithoutTheFlag::test_the_cli_and_the_loader_name_the_same_sessions` (1265–1281); the doc parity is `tests/test_cli_documentation.py::test_cli_references_mention_every_long_option`.
