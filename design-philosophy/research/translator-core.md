# The translation core: batching, alignment, capability gating, session state

> **Post-review corrections (added after the independent review of `report.md`, same session).**
> 1. §2.9 says `translation_metadata.py` allows "a frozenset of four keys". Confirmed: `RECORD_KEYS = frozenset({RECORD_MARK_KEY, "model", "date", GLOSSARY_SHA_KEY})` at `translation_metadata.py:91`. Note however that the code's own prose disagrees with itself: `loader/epub_loader.py:828-832` claims the record answers "which build, which model, which endpoint, which command", which the key set forbids. The report treats that as a documentation defect and so should this note.
> 2. §2.4's "the one fallback in the system" (quoting `epub_loader.py:2359`) needs its scope stated: it is the one **alignment** fallback. `translator/chatgptapi_translator.py:1747-1756` separately degrades a whole batch to per-paragraph translation on any *non-*`BatchMismatch` exception — that is a transport fallback, consistent with the note's own §2.5 retry table, but it makes the quoted sentence read more broadly than it is.
> 3. Keyword counts in the report were re-measured case-insensitively: `measured` 99 occurrences / 97 lines, `chosen` 47 / 46. If you quote those numbers from this note, use the occurrence counts, not the line counts.

Research note for the design-philosophy report. Scope: `book_maker/translator/*`, plus the
parts of the loader, planner and session modules the translator core actually depends on.

- **Commit**: `e1027a4823e98f668ef950740b6f805ec30bd3b2` (2026-09-30, `feat(docs): Integrate Jev
  through cloudflare ai gateway as epub tag classifier (#580)`). Working tree clean except the
  untracked `design-philosophy/` output directory.
- **Method**: read the implementation; run focused tests and two throwaway in-process probes
  (outside the repo, since deleted). No project source was modified.
- **Evidence labels**: **[V]** read in the cited file:line · **[R]** produced by a command run on
  this machine at this commit · **[I]** inference from several pieces of evidence · **[U]**
  unknown / not verified.
- **Commands actually run** (§6 lists them again): the focused pytest set in §5, `git log/status`,
  `Select-String` counts over `book_maker/**.py`, and one temp script exercising
  `Base._check_batch` and `structured.extract_json_object`.

---

## 1. What the "translator core" is, from the outside

A run picks one *route* by wire format, not by model name: `FORMAT_DICT` maps `openai`,
`anthropic`, `codex`, `gemini`, `qwen`, `google`, `caiyun`, `deepl`, `deeplfree`, `tencent`,
`customapi`, `groq`, `xai`, `litellm` to classes, and `ROUTE_DICT` maps the `orcarouter` model
name to one (`book_maker/translator/__init__.py:17-54`). Every route is asked for two things by
the rest of the program: `translate(text)` (one paragraph) and `translate_list(texts)` (a group),
plus an optional arbitrary-prompt channel for plan classification.

The core's job is therefore not "call an API". It is to keep a **book's paragraph sequence** and a
**non-deterministic remote reply** in correspondence, while the reply arrives through whichever wire
shape the chosen route speaks — OpenAI-shaped JSON, anthropic messages, Gemini's SDK, Qwen-MT's
language-pair protocol, codex's JSON-RPC sidecar, or one of the four fixed MT services — and while
the endpoint may or may not honor the request features the code would like to use. Everything below
is machinery for that.

Two structural facts set the frame:

1. The **grouped path is the normal path**. Plan mode packs consecutive units into one request
   (`book_maker/loader/plan.py:1657-1719`), so most requests carry N>1 paragraphs and the reply
   has to be split back apart.
2. The **prefix must be byte-stable in session mode**, which forbids most of the obvious ways of
   smuggling per-request information into a request
   (`book_maker/session_context.py:16-21`).

---

## 2. Mechanism inventory (file:line)

### 2.1 The route contract, and how much of it the base already owns

`Base` is an ABC with exactly two abstract methods, `rotate_key` and `translate`
(`book_maker/translator/base_translator.py:1054-1060`). Everything else has a default:

| Mechanism | Where | What it is |
|---|---|---|
| `translate_list` default | `base_translator.py:1194-1199` | loops `translate`; trivially length-aligned |
| batch prompt assembly | `base_translator.py:1259-1305` | delimiter-joined payload + "Output EXACTLY N" instruction |
| reply splitting | `base_translator.py:1307-1351` | `(1)…(2)` numbering, then the `@@` delimiter |
| alignment check | `base_translator.py:1201-1232` | `_check_batch`: count, then empty-slot |
| one-request batch driver | `base_translator.py:1353-1475` | installs batch prompt/system msg, restores in `finally` |
| arbitrary-prompt channel | `base_translator.py:1069-1081` | `_chat_completion`, `NotImplementedError` by default |
| schema-question ladder | `base_translator.py:1122-1192` | prompt rung + `_descend` + rung retirement |
| session compaction helpers | `base_translator.py:511-677` | record/compact/report/restore |
| capability declaration | `base_translator.py:274-341` | six `SUPPORTS_*` flags + two session flags |

Only 5 of the 12 translator modules define `translate_list` (`base` + `chatgptapi`, `claude`,
`gemini`, `codex`); the other 7 inherit the loop, which means the fixed MT engines (google, deepl,
caiyun, tencent, customapi) are aligned by construction and cannot misalign **[R]** (`Select-String`
count over `book_maker/translator/*.py`).

`supports_structured_json()` is *derived from the implementation*, not declared:
`cls._chat_completion is not Base._chat_completion or cls.structured_json is not Base.structured_json`
(`base_translator.py:1108-1120`). The stated reason is that "a new translator cannot advertise a
capability it never implemented, or implement one it forgot to advertise."

### 2.2 Batching: who decides how many units share a request

Three different sizes are in play and they are deliberately not the same number:

- **Partition-time** (what the plan says): `assign_batches`
  (`plan.py:1657-1719`) packs consecutive units greedily to a token budget and to
  `max_units=GENERAL_GROUP_MAX_UNITS = 16` (`plan.py:1465`). Two modes: short-run grouping by
  `group_size`/`GROUP_MAX_CHARS` when no budget is given, and general packing when
  `token_budget` is not None. Deterministic in unit order and text (`plan.py:1681-1682`).
- **Request-time, strict endpoints**: the plan's budget applies.
- **Request-time, everything else**: `SUBSTRICT_GROUP_MAX_UNITS = GENERAL_GROUP_MAX_UNITS // 2 = 8`
  (`plan.py:1481`) and `substrict_token_budget` (`plan.py:1573-1584`). The comment is explicit that
  the halving is "measured in its *reason* and unmeasured in its *size*" and that the budget is a
  "chosen margin over an unobserved edge, sized by judgement" (`plan.py:1478-1480`, `1545-1547`).

The plan-time and request-time caps are enforced twice: the loader sizes the request
(`epub_loader.py:1104-1129`, `_structured_degree` → `_plan_schema_verified`), and the OpenAI
translator re-checks per request because `--model_list` rotation can move execution to a
different-capability model mid-run (`chatgptapi_translator.py:1716-1730`, `1843-1848`). The
mirror constant is pinned by
`tests/test_general_grouping_and_json_degree.py:911`
(`assert ChatGPTAPI.substrict_batch_cap == SUBSTRICT_GROUP_MAX_UNITS`) and is spelled again as a
class attribute because "an instance built without `__init__` still needs the value"
(`chatgptapi_translator.py:459-471`).

The budget also grows with the run's own measured prompt overhead, so a fat custom `--prompt`
does not silently become most of the bill (`plan.py:1551-1570`, `chatgptapi_translator.py:1604-1630`).

### 2.3 Alignment: three checks, in order of how much they can see

The wire format for a grouped reply is either delimiter-joined text
(`BATCH_DELIMITER = "\n\n@@\n\n"`, `base_translator.py:33`) or a **JSON envelope with ids**
(`{"paragraphs": [{"id": n, "text": …}]}`, `chatgptapi_translator.py:1632-1649`). The id envelope
exists for one stated reason: "Ids rather than positions, because position is not something a
reply can be *checked* against" (`chatgptapi_translator.py:1635-1640`).

1. **`Base._check_batch`** (`base_translator.py:1201-1232`) — wrong count, or an empty slot for a
   non-empty source. Both raise `BatchMismatch`. Its docstring states the residual blind spot
   outright: "A wrong alignment at the right count with no empty slot is not detected, by design:
   there is no signal to detect it by." **[R]** I confirmed all three behaviours by calling it
   directly: wrong count and empty-slot raise; `_check_batch(["170 verse","blank"], ["(filler)","一七零"])`
   returns `None`.
2. **`_align_batch_items`** (OpenAI structured path, `chatgptapi_translator.py:1775-1802`) — the
   echoed id set must equal `set(range(n))` exactly: no duplicate, no stranger, none missing; then
   `_check_batch` on the assembled order. Note this does *not* trust the model's ordering — it reads
   by id and re-sorts.
3. **`_marker_slot_mismatch`** and **`_numeric_slot_shift`** (`epub_loader.py:2441-2505`,
   `2508-2578`) — the two checks for a same-count shift, i.e. the blind spot in (1). The marker
   check needs plan-mode `units` (ownership sets); the numeric check needs nothing but the texts,
   because digits travel with their source across a shift. Both are in the loader, not the
   translator, because they need the *units*, which the translator never sees
   (`epub_loader.py:2370-2375`).

`_numeric_slot_shift` carries three explicit non-rules with the false positive each prevents, and
they are the clearest single artefact of this project's style: absence alone is never evidence
(one clean cell lost 6 of 24 verse numbers and rendered others as CJK numerals); one witness is not
enough; the witnesses must agree on direction (`epub_loader.py:2527-2538`). The measured cell that
motivated it is named (`ds-waste-u96-b4800`, `o4m-waste-u256-b1600`).

### 2.4 What happens after a mismatch: one ladder, two repairs

The translator's job ends at `BatchMismatch`. The docstring at `base_translator.py:56-66` says
nobody repairs it where it is raised; the loader's ladder halves the chunk
("8 + 4 + 2 + 1 + 1", `epub_loader.py:2356-2365`) and does not retry the same group, because "a
model that miscounted once usually miscounts again, and each retry re-pays the whole group".

- The ladder is `_translate_texts_aligned` → `_divide_and_translate` (`epub_loader.py:2356-2438`,
  `2592-2611`), pinned by `tests/test_batch_groups_and_markers.py:298-317`, which asserts the
  halving shape (`batch_calls[0] == texts`, then `["POISON","a4"]`, then singles) and that
  `"a1" not in fake.single_calls` — i.e. the clean half is not re-paid.
- A third recurrence prints an operator hint naming the two knobs that shrink a batch
  (`epub_loader.py:2580-2590`, pinned at `tests/test_batch_groups_and_markers.py:319-340`).
- Tag mode reaches `translate_list` by a different path and has its own one-line repair instead
  (`loader/helper.py:562-581`), with the reason spelled out: "tag mode has no such ladder … One
  local sweep of `translate()` calls is what that repair was; it stays here, where the ladder
  cannot reach."

Fatal errors are separated from mismatches: a clone's `_fatal_error_detected` flag is propagated
to the shared model so other workers stop (`epub_loader.py:2398-2415`), and the run substitutes
`TRANSLATION_ERROR_MARKER` — which the insertion path then refuses to write
(`epub_loader.py:2036-2042`, `2624-2629`). Only one route defines the marker at all:
`gemini_translator.py:143`.

### 2.5 Failure taxonomy: five kinds of "no", each answered differently

The core never treats failure as one thing. The classes and their dispositions:

| Class | Where declared | Disposition |
|---|---|---|
| `ModelUnavailable` | `capabilities.py:35-43` | user-facing; printed, not a traceback |
| `ProbeDeferred` | `capabilities.py:68-69` | outage: record no verdict, ask again later |
| `StructuredOutputUnsupported` | `capabilities.py:46-51` | capability answer: demote, fall back for this call |
| `StructuredRefusal` | `capabilities.py:54-61` | model declined *this text*: not a capability failure |
| `RungRejected` | `structured.py:19-27` | this request shape refused; descend |
| `BatchMismatch` | `base_translator.py:56` | divider's business, never a retry |
| `VisionRequestFailed` | `vision.py:40-48` | not a rung rejection; the same image would be refused again |

Retry policy is likewise split rather than uniform:

- **Transport weather** on the per-paragraph path is retried *forever*: `translate_with_backoff`
  uses `retry_if_exception(_is_retryable)`, `wait_exponential(max=RETRY_WAIT_CAP=300)`,
  `before_sleep=_say_retrying`, and deliberately **no `stop`** (`loader/helper.py:23-27`,
  `77-81`, `493-505`). `FATAL_ERROR_NAMES` (auth, 400, 404, …) and `FATAL_ERROR_TYPES`
  (`TypeError`, `KeyboardInterrupt`, `SystemExit`) propagate on the first attempt
  (`helper.py:34-54`). The spec is `tests/test_translate_with_backoff.py` — "no attempt cap at
  all" asserts `stop is stop_never` (`:85-86`); a `TypeError` is never waited out (`:148-154`)
  because it "is an infinite loop rather than a failed run".
- **Transport weather** on the OpenAI structured batch is retried **three times**
  (`chatgptapi_translator.py:1804-1811`), and the retry explicitly does *not* cover
  `BatchMismatch`, `StructuredRefusal` or `StructuredOutputUnsupported`.
- **A compact turn** gets `COMPACT_ATTEMPTS = 3` before the window is thrown away
  (`base_translator.py:509`, `595-646`), with a second arm that only arms from the second failure.
- **A capability probe** that hits an outage records nothing and re-probes later
  (`capabilities.py:141-145`, `254-291`), so a gateway restart cannot permanently demote the run.
- The vision probe re-uses the patient policy but bounded by a caller deadline
  (`chatgptapi_translator.py:926-962`).

### 2.6 Capability gating: ask the endpoint, not the model name

`capabilities.py` states its premise in the module docstring: "A model's name says nothing about
whether the server behind it applies a JSON Schema, accepts an explicit temperature, or serves the
model at all." Three separate negotiations are implemented on that premise:

1. **Does it apply a strict schema?** `PROBE_PROMPT` asks for plain text while the schema pins a
   single-value enum, so the only way `PROBE_EXPECTED` comes back is that the server really applied
   the schema (`capabilities.py:76-94`). `grade_probe_response` yields `strict` / `shape` / `json` /
   `unsupported` (`:220-251`). The probe deliberately sends no temperature and no token cap —
   "the probe must test exactly one capability" (`:254-261`), pinned by
   `tests/test_capabilities.py:143-158` including `assert "schema_ok" not in prompt`.
   Verdicts are *graded per purpose*: a single translation schema pins a target-language **value**,
   which only `strict` applies, so only `strict` may translate via schema
   (`chatgptapi_translator.py:650-662`); the batch contract pins no values, and the measured eval
   held at the `json_object` degree, so `shape` and `json` batch too (`:664-679`).
2. **Does it serve this model?** `probe_model_route` sends one tiny request and reads only the
   errors that are *about the model* (`capabilities.py:344-397`). `names_missing_model` distinguishes
   a 404, a `model_not_found`, a 400 whose `param` field names the model, and a 400 about a
   parameter that merely mentions a model. Any other failure keeps the model and "the run finds out
   from the real request, where transport failures belong" (`:416-448`). The listing is "a hint
   under a refusal, never a gate in front of one" (`:450-453`).
3. **Does it read the image?** `vision.py` shows the model a random 4-character string rendered into
   a 200×80 image and counts it as image-capable only if the string comes back
   (`vision.py:63-70`), because "an endpoint in front of a text-only model can drop the image part
   and answer the text part alone" and every answer would then look legal.

The results land in a `CapabilityLedger` keyed by model (`capabilities.py:485-513`), under an
`RLock` so N parallel workers cost one probe ("`test_one_probe_under_parallel_workers`",
`tests/test_capabilities.py:172-195`). Demotion needs a **streak** of
`STRUCTURED_FAILURE_THRESHOLD = 2` (`capabilities.py:207-209`, `600-629`), and a success clears it
(`:594-598`), so one garbled proxy reply does not disable structured output for a multi-hour run.
Vision has "its own dict and no streak" (`:507-510`) — a schema demotion says nothing about images.

Declared capabilities are then enforced at the CLI, *before* anything is spent: `--batch/-batch-use`
on a route without the Batch API is a hard error (`cli.py:3165-3180`), `--use_context session` on a
route that does not implement it is a hard error (`cli.py:3182-3194`), and the flags a route cannot
carry (glossary, `--no-thinking`, extras, `--translation-metadata`) are warned about with the reason
(`cli.py:1517-1542`, `1588-1599`) rather than accepted in silence. The route→capability facts are
pinned by contract tests (`tests/test_cli.py:1429-1473`).

### 2.7 Session state: an append-only prefix, a bounded seed, and a snapshot

Session mode replaces window mode by keeping **one append-only message list**; two rules follow and
are labelled "load-bearing everywhere below": nothing already sent may be rewritten (a moved prefix
is a cache miss, "strictly worse than window mode"), and anything per-unit rides in the fresh tail
(`session_context.py:1-27`).

- `SessionHistory` is a list of user/assistant pairs plus a token estimate
  (`session_context.py:349-379`); compaction triggers at `--context-compact-at`, default
  `DEFAULT_COMPACT_BUDGET = 8192` (`:117`).
- The compact turn asks for a fixed-shape handoff report with English protocol headers
  (`:400-419`, `:482-505`); the reply is split into summary / style / renderings
  (`:886-940`), trimmed by shape, and written to `<book>_handoff.md` as an **overwritten snapshot,
  never a growing log** (`:1007-1170`).
- **The seed is bounded three times**, weakest first: the prompt asks for ~300 tokens, the request
  carries a token cap where the endpoint accepts one, and the client truncates head-first at
  `seed_cap(budget) = min(512, budget // 2)` (`:120-180`, `:212-310`). The reason given is not
  efficiency: "an unbounded seed is not an inefficiency — it is a death loop"
  (`:120-140`), diagnosed upstream in PR #569 and fixed at the source rather than in accounting.
  `truncate_seed`'s postcondition is stated as absolute: "The layers above this one can be ignored
  by a model; this one cannot be, so it may not have exceptions of its own."
- A failed compact keeps the window and retries on the next paragraph, unless the failure names a
  reason retrying cannot fix (`base_translator.py:595-646`).
- `--resume` restores the summary, the style and (only with `--glossary-auto on`) the learned
  renderings from the snapshot, and continues the window counter rather than restarting it
  (`base_translator.py:535-593`). A `<book>_handoff.md` beside a book is otherwise inert.
- Compaction on each route re-requests the report through that route's own client
  (`chatgptapi_translator.py:1401-1514`, `claude_translator.py:458-520`) but assembles it through
  the shared `_handoff_report` / `_learn_from_handoff` / `_show_handoff`
  (`base_translator.py:358-499`, `663-677`).

### 2.8 Per-unit context on a cache-stable prefix: glossary, markers, preambles

Three mechanisms exist only because of rule (2) above:

- **Glossary** — `Glossary.prompt_block(source)` emits *only* the terms that occur in the unit being
  translated, and the module docstring says why that is not a token micro-optimisation: a varying
  block can only ride in the fresh tail, "and dumping the whole file there would be paid for on
  every single request at full input price" (`glossary.py:14-19`, `187-210`). Matching is
  script-aware (CJK has no word boundary; a mixed-script term is guarded per edge,
  `glossary.py:41-96`). `merge` returns conflicts for loud reporting and "never silently resolved"
  (`glossary.py:214-232`).
- **Marker and structure contracts** — both ride in the *user* message, not the system message,
  because they vary per request and "a system message that changes between requests moves the
  prefix session mode caches" (`base_translator.py:901-915`, `945-993`). Both are only said to a
  request that has the thing being protected: `_carries_markers` / `_carries_structure`
  (`:893-899`, `956-972`) — "a model told to preserve tokens in a text that has none is being
  taught to invent them" (`:724-725`).
- **Batch recording shape** — `BATCH_CONTEXT_PER_LINE` and `BATCH_SYS_MSG_PER_REQUEST` are
  properties that turn **off** while a session is open
  (`chatgptapi_translator.py:1075-1097`), because borrowing the system message or splitting a
  grouped request into N pairs would move or misdescribe the cached prefix. The base drives the
  decision and the restore in a `finally` (`base_translator.py:1406-1462`, `1471-1473`).

### 2.9 Provenance: what the run records about what it did

`translation_metadata.py` states two rules for every field: "Nothing is invented" (an unestablished
fact is omitted, never written empty) and "Nothing derived travels" (a user glossary is embedded
verbatim with its checksum; a tool-derived one is not recorded at all)
(`translation_metadata.py:28-43`). The allowed key set is a frozenset of four keys, pinned here and
in a test "so that adding a fifth key is a decision somebody makes on purpose" (`:88-91`). Nothing
identifies the operator, the machine or the command — an explicit owner ruling
(`:9-14`). `redact` keeps the endpoint's words and removes the run's own secrets, because those
words are usually the only explanation of what went wrong (`redaction.py:1-12`, `22-35`), and
`Base.__init__` registers the keys before the first request on the grounds that "an endpoint that
refuses a key routinely quotes it back" (`base_translator.py:1017-1022`).

### 2.10 The codex boundary: a subprocess instead of an HTTP request

The `codex` route spends a ChatGPT subscription instead of API credits by driving
`codex app-server` over newline-delimited JSON-RPC (`codex_client.py:1-29`). It is a genuinely
different boundary and the translator class says so with flags: `SUPPORTS_STRUCTURED_OUTPUTS = False`,
`SESSION_CONTEXT_ALWAYS_ON = True`, `BATCH_SYS_MSG_PER_REQUEST = False`, and `PROMPT_SECTION_SLOTS`
routing the system section to "the thread instructions"
(`codex_translator.py:176-204`). Every feature that would let a turn *act* rather than answer is
disabled at spawn and then **verified off** via `config/read` before the first turn
(`codex_client.py:67-97`, `408-448`), because "book text is untrusted input". `--parallel-workers`
is refused on this route outright (`cli.py:3196-3204`).

---

## 3. Candidate technical design principles

### P1 — The output contract is "exactly N aligned items or a named exception"; repair happens once, above every route. **[V]**

**Evidence.** `BatchMismatch`'s docstring is the contract statement
(`base_translator.py:56-66`). It is raised from four independent carriers:
`Base._check_batch` (`:1201-1232`), `ChatGPTAPI._align_batch_items` and
`_parse_json_object_batch` (`chatgptapi_translator.py:1784`, `1791`, `1795`, `1845`, `1949-1956`),
`Claude.translate_list` via the shared `_do_batch_translate` (`claude_translator.py:601-622`), and
`Gemini._parse_batch_response` (`gemini_translator.py:592-595`). The single repair site is the
loader's halving ladder (`epub_loader.py:2356-2438`), whose docstring calls it "The one fallback in
the system". The counterexample check is what makes this credible rather than aspirational: while
reading I found that the translation side is *not* uniformly self-repairing (see §4.1–4.2), but in
each case the mismatch still escapes to the ladder rather than being silently patched.

**Why this shape.** The alternative — each route repairing its own bad reply — was actually tried
and removed. `_extract_paragraphs` carries the tombstone: a final rung that split on every non-blank
line "manufactured a plausible *wrong* count out of a perfectly correct multi-line reply", and
"a reply carrying no delimiter now yields one item, which is a mismatch, which the loader's ladder
divides" (`base_translator.py:1345-1351`). `translate_list_or_singles` carries the same record for
tag mode (`loader/helper.py:562-571`).

**Cost.** Every misalignment costs a full extra round of requests for the affected subtree
(`~2x` the batch). A route that could cheaply re-ask would still pay the ladder.

**Limit / where it stops.** The contract is only as strong as its checks. A same-count, no-empty-slot
shift is invisible to `BatchMismatch` and was *measured* to reach the book
(`epub_loader.py:2447-2453`, `docs/en/evaluation/grouping-batch-size.md:38`). The follow-up
detectors (`_marker_slot_mismatch`, `_numeric_slot_shift`) only fire under conditions the code
names, and `_numeric_slot_shift` needs at least `_SHIFT_MIN_WITNESSES` agreeing witnesses. Routes
that cannot batch (the MT engines) keep the contract trivially, so the principle covers them by
degeneracy rather than by design.

### P2 — "Would not survive" is not accepted silence: a flag a route cannot carry is refused or warned, with the reason and the fix. **[V]**

**Evidence.** Six class-level flags plus two session flags on `Base`
(`base_translator.py:274-341`), each with a comment saying what "False" means and why accepting the
flag anyway would be wrong — e.g. `SUPPORTS_BATCH_API`: "the loader calls
batch_init/add_to_batch_translate_queue/is_completed_batch on the translator, so accepting it would
be an AttributeError partway in" (`:310-315`). The enforcement is `exit(1)` at the CLI for the two
that would crash or silently cost more (`cli.py:3165-3194`) and a stated warning for the rest
(`cli.py:1517-1542`), plus a `set_request_extras` that raises rather than "storing attributes
nothing reads" (`base_translator.py:679-688`). The `--prompt` sections go further: every route
declares where each section lands (`native`/`appended`/`none`, `base_translator.py:747-765`) and
`prompt_adoption_line` prints at run start which of them this route adopted and appended where —
"a run that is quietly translating under a style the endpoint never saw is exactly what this line
exists to make impossible" (`cli.py:591-631`).

**Cost.** A large declaration surface: 6 `SUPPORTS_*` flags × 15 registered route classes (14 in
`FORMAT_DICT` plus `orcarouter` in `ROUTE_DICT`, `book_maker/translator/__init__.py:22-54`), kept
honest by hand-written comments and contract tests (`tests/test_cli.py:1429-1473`,
`tests/test_claude_session_context.py:677`, `tests/test_structured_classify.py:657-698`). Every new
route must answer all of them.

**Limit.** The flags are declarations, so they can drift from the implementation, and nothing
mechanical keeps them in step — only comments and hand-written contract tests
(`tests/test_cli.py:1429-1473`, `tests/test_claude_session_context.py:677`,
`tests/test_structured_classify.py:657-698`). I looked for a flag that contradicts its route's
behaviour and did **not** find one: the two candidates I checked are both explicitly consistent
(`claude_translator.py:466-468` states the route carries no glossary; `qwen_translator.py:19-26`
states that Qwen-MT takes a window in `context_list` but has no turn for a prompt). So this is a
residual risk on the evidence I have, not a demonstrated defect.

### P3 — Trust the endpoint's behaviour, not its acceptance, its listing, or its name. **[V]**

**Evidence.** Three probes each engineered so acceptance cannot produce a false positive: the schema
probe asks for plain text while pinning an enum (`capabilities.py:76-94`); the vision probe renders
a random string into an image that is never named in the prompt
(`vision.py:9-15`, `63-70`); the route probe ignores the listing unless the endpoint refuses
(`capabilities.py:373-397`, `450-453`). The *grading* is deliberately asymmetric: a fatal error
propagates, an outage defers, an ambiguous failure grades as unusable, and a 400 grades as a
capability answer (`capabilities.py:129-154`, `280-291`; pinned in
`tests/test_capabilities.py:100-141`). `classify_bad_request` exists because otherwise "a
temperature rejection is misread as 'no schema support' … and the real cause never reaches the
user" (`capabilities.py:294-312`). `reasoning.py` applies the same discipline to `--no-thinking`:
seven candidate spellings, and only an *explicit rejection of the field just sent* advances the
ladder — a 400 naming another parameter, a 500, an auth failure, or a message that merely mentions
the field all propagate untouched (`reasoning.py:11-25`, `114-150`).

**Cost.** One extra round trip per model per question, plus code volume. `prompt_overhead_tokens`
spends a real request to size the grouping budget (`chatgptapi_translator.py:1599-1630`).
The whole mechanism exists to avoid paying for a wrong assumption, so it is a bet that the probe is
cheaper than the mistake.

**Limit / counterevidence.** `set_model_list` explicitly does *not* probe, so "a run that writes a
plan and exits pays for nothing" — the cost is deferred to the first paid call, not eliminated
(`chatgptapi_translator.py:1975-1986`). And the probes can be wrong in the other direction: the
comment at `plan.py:1466-1474` records that both corrupting arms *probed* `strict`, which is why
the sub-strict margin exists as a margin rather than as a gate.

### P4 — Operator-facing knobs are chosen numbers with their evidence recorded next to them; measured and chosen are labelled separately. **[V/I]**

**Evidence.** This is the most unusual thing in the code and it is systematic in the translator
core. `'Measured'` appears 97 times and `CHOSEN`/`Chosen` 46 times across `book_maker/**.py` **[R]**,
usually in the same comment block:

- `plan.py:1493-1512` — "Measured: the 260906 weak-model B sweep … ran 1600–4800 **clean** … Also
  measured, and pointing the other way: per-content-token cost falls monotonically with request
  size … **Chosen**: 1200, owner ruling 260907."
- `session_context.py:76-116` — a table of the actual numbers (`C=4096 63 requests 361k tokens
  (+27.3%) 19 compactions`; `C=8192 52 requests 396k tokens (+39.8%) 9 compactions`) with the
  admission "the honest record is that it did not work as intended".
- `plan.py:1534-1547` — "**Typed, not derived, and this is the one place the arithmetic is broken on
  purpose**" and "Nothing here is measured except that cost".
- `translation_checks.py:27-40` — three thresholds each annotated "chosen, not measured" or "A
  chosen margin, not a measurement".
- `capabilities.py:164-177` — `REASONING_ALLOWANCE_TOKENS`: "CHOSEN, not measured: 1500 covers the
  reasoning budgets observed there with room over."

Fourteen files contain the phrase "owner ruling" **[R]**, which marks a decision attributed to a
human rather than derived from data. The corpus of eval write-ups that these comments cite is
mostly *not* in this checkout (see §5), but the two that correspond to the translator core's
biggest numbers are: `docs/en/evaluation/grouping-batch-size.md` and
`docs/en/evaluation/session-compact-budget.md`, and their abstracts match the code comments
("**These defaults are an owner-set safety margin, not a measured optimum**",
`grouping-batch-size.md:5`; "**The default stays at 8192 anyway, for continuity, not price.**",
`session-compact-budget.md:5`).

**Cost.** Comments carry more interpretive text than code; the numbers cannot be re-derived from
the repository alone; and it makes the file hostile to a drive-by change (by design — e.g.
"Do not move either number without re-running those two cells",
`session-compact-budget.md:63`).

**Limit / competing explanation.** Much of this could be a *convention of this codebase's
AI-assisted workflow* rather than a deliberate product philosophy. What argues for the stronger
reading is that the convention is load-bearing in specific places — the "measured vs chosen"
distinction is used to justify accepting a *worse* number (`8192`, dearer than `4096`), and the
evaluation docs have their own "Limits" sections that downgrade their own findings
(`grouping-batch-size.md`, `session-compact-budget.md:67-72`). A pure style convention would not
need to admit that the shipped default costs more.

### P5 — Reliability mechanisms are scoped to the code path that can carry them, and the seams between those paths are named. **[V/I]**

**Evidence.** The same class of failure is handled by three different mechanisms depending on which
code path the request travels, and each site names the others:

- plan mode: halving ladder (`epub_loader.py:2356-2398`, "Plan mode answers a `BatchMismatch` with
  `_translate_texts_aligned`");
- tag mode: one local per-line sweep (`loader/helper.py:562-581`, "tag mode has no such ladder
  … it stays here, where the ladder cannot reach");
- the route: `BatchMismatch` only, never a repair (`base_translator.py:56-66`).

Retry patience is likewise scoped: unlimited on the per-paragraph path (`helper.py:493-505`),
three attempts on the structured batch (`chatgptapi_translator.py:1804-1811`), three on a compact
turn (`base_translator.py:595-646`), deadline-bounded for vision questions
(`chatgptapi_translator.py:936-939`). And the *batch* path's answer to a transport failure is to
drop to the per-paragraph path, which is itself patient
(`chatgptapi_translator.py:1747-1756`) — so the two policies compose instead of contradicting.

**Cost.** Three implementations of "cope with a bad reply", and a reader has to hold the path
topology in mind to know which applies. This is the strongest competitor to P1: an engineer reading
only the translator package would conclude there is one contract; the loader and helper tell a more
layered story.

**Limit.** I could not find a mechanism that keeps these three in step. The provenance is comments
and tests, not a shared abstraction.

### P6 — The run narrates its own reasoning, and quietly degrading is treated as a defect. **[V]**

**Evidence.** Nearly every negotiated or degraded outcome prints one line explaining what was
learned and what happens next: the rung retirement ("ℹ '{model}' keeps refusing the {name} request
shape; using a simpler one from here on", `base_translator.py:1188-1192`), the schema demotion
("falling back for this one and trying structured outputs once more",
`capabilities.py:624-629`), the deferred probe ("using the delimiter method until the endpoint
answers", `:588-592`), the reasoning-ladder step (`reasoning.py:153-169`, whose docstring says "a
negotiation nobody can see is a negotiation nobody can debug"), the seed truncation
(`session_context.py:304-309`), the compact failure (`base_translator.py:633-641`), every retry
(`helper.py:84-93`, justified as "an hours-long tolerance that prints nothing looks exactly like a
hang"), and the misalign hint after three recoveries (`epub_loader.py:2580-2590`).

The counter-discipline is equally explicit: **one line per event, not per occurrence**
(`base_translator.py:442-447`), `--quiet` suppresses echoes but never warnings or errors
(`:670-673`), endpoint errors are quoted rather than summarized (`redaction.py:1-12`,
`capabilities.py:36-41`), and the numeric-slot detector's design notes read like a false-positive
budget (§2.3).

**Cost.** Verbose output; `rich` markup escaping is a real hazard the code works around
(`escape` at `base_translator.py:663-677`). Every new mechanism has to have a sentence written for
it.

**Limit.** This is a *readout* philosophy, not a gating one. `UsageMeter` is explicit:
"It is shown on the progress bar rather than judged: whether the ratio is worth it is the
operator's call, made while watching" (`base_translator.py:152-156`), and
`_SeedTruncations` says "Nothing here gates anything today" (`session_context.py:188-190`).

---

## 4. Counterexamples, competing explanations, and things the brief assumed

### 4.1 A route that *does* carry its own repair — and it may be unreachable **[V, I]**

`Gemini.translate_list` documents "Falls back to one-by-one translation if batch translation fails
after all retry attempts" (`gemini_translator.py:449-451`) and implements it at `:560-577`. Traced
statically, that branch looks hard to reach: `_batch_translate_with_retry` returns only a truthy
result and otherwise raises (`:475-514`), `BatchMismatch` is re-raised immediately, and a `None`
parse becomes `ValueError` which `_should_retry` accepts (`:74-88`) and tenacity retries
`stop_after_attempt(7)` before re-raising. So the per-line sweep at `:567-575` is not reached via
the mismatch or the parse path. I did **not** construct a live call that reaches it, so this is a
static reading of the control flow, not a demonstrated dead path. Even so, the *intent* recorded in
the code contradicts a clean reading of P1: a route in this codebase is documented as repairing its
own batch.

### 4.2 Gemini also pre-cleans the payload to protect the alignment check **[V]**

`_batch_translate` filters empty/whitespace paragraphs before building the batch and re-inserts
results at their original indices, on the grounds that "the model silently drops empty/whitespace
paragraphs, causing a bogus 'expected N, got N-1' mismatch" (`gemini_translator.py:522-534`,
`550-558`). That is a route-level alignment accommodation the other carriers do not have. **[I]** The
likely mechanical reason is that this carrier joins the group with a plain `"\n\n"` (`:534`) instead
of the collision-safe `BATCH_DELIMITER` the base uses, so an empty unit is not simply recoverable
from the reply. Competing explanation: it may be legacy code predating plan mode — the file gives no
date for it, unlike most of the constants in this package.

### 4.3 The base's own shared-helper comment is not fully accurate **[V]**

`base_translator.py:501-504` introduces that section as helpers that "live here because the openai
and anthropic routes wrote them identically, down to the lines they print", yet neither
`_compact_session` nor `_session_budget` exists on `Base` — both are implemented **separately** on
both routes (`chatgptapi_translator.py:1395-1514`, `claude_translator.py:452-520`); a `grep` over
the package finds no base definition of either. What is actually shared is the report
parsing/assembly (`_handoff_report`, `_learn_from_handoff`, `_show_handoff`) and the failure policy
(`_compact_failed`, `_start_empty_window`); the request-and-seed part is per route. So "one
compaction implementation" would be an over-claim; the accurate statement is "one report model, two
transports".

### 4.4 Is the patience a reliability investment or an inability to bound it? **[I]**

`translate_with_backoff` has **no attempt cap and no total-time cap**
(`helper.py:493-505`, `tests/test_translate_with_backoff.py:85-86`). Two readings are consistent
with the evidence:

- *(a) Deliberate*: the comment says the user base "runs flaky providers where a 429 or a first
  token can be minutes or hours away, and an attempt cap there abandons a paid book over weather"
  (`helper.py:478-488`).
- *(b) Compensating for a missing resume guarantee*: an abandoned run can be resumed
  (`--resume` exists, `base_translator.py:535-593`), so patience is not strictly necessary for
  correctness. But see the discrimination test: the *only* class of error excluded is one that
  "will never clear", and that exclusion is enumerated by name from the provider SDKs
  (`helper.py:29-44`). A design that merely could not bound its retries would not need a fatal
  taxonomy; a design that chose patience would.

I lean (a) — the taxonomy is doing real work, and the cost is explicit (`_say_retrying`,
Ctrl-C exceptions). But (b) cannot be excluded from the code alone, and would need the issue
history to settle.

### 4.5 Is the "measured/chosen" discipline the author's philosophy or the tooling's? **[U]**

The comment style (dated ruling IDs, eval cell names, "this comment is the in-tree record of the
ruling — worktrees do not carry the repo's CLAUDE.md", `helper.py:491-492`) strongly suggests an
agent-assisted workflow with a house writing convention. `CLAUDE.md` is **not in this checkout**,
so I cannot check the convention's actual source. What is verifiable is the *effect*: numbers the
operator can change are annotated with whether they were measured, and several shipped numbers are
documented as knowingly not the cheapest. Whether that counts as a design philosophy or as a
documentation culture is a judgement call, and I flag it as such rather than resolving it.

### 4.6 Mechanism with no gate: `prompt_overhead_tokens` **[V]**

Its own docstring says "A sizing hint, never a gate: anything that goes wrong here answers `None`
and the caller falls back to its floor" (`chatgptapi_translator.py:1613-1615`). So not every
mechanism in the core is a guardrail; some are advisory inputs to other guards. Worth keeping in
mind before reading every piece of machinery as a reliability investment.

### 4.7 The batch API path is a third translation channel and is barely integrated **[V]**

`ChatGPTAPI` implements `batch_init` / `add_to_batch_translate_queue` / `is_completed_batch` /
`batch_translate` (`chatgptapi_translator.py:2013-2292`) against OpenAI's Batch API, and
`SUPPORTS_BATCH_API` is True only on this class (`:429`). It is gated at the CLI
(`cli.py:3165-3180`) but I did not trace it end-to-end, and it does not go through
`_translate_texts_aligned`'s checks in any way I verified. Treat batch-API runs as out of scope for
P1/P5.

---

## 5. Verified / unverified split

### Verified by running **[R]**

```
python -m pytest tests/test_capabilities.py tests/test_translate_with_backoff.py \
  tests/test_batch_groups_and_markers.py tests/test_general_grouping_and_json_degree.py \
  tests/test_batch_layers.py tests/test_translation_metadata.py tests/test_no_thinking.py -q
→ 1 failed, 358 passed in 129.91s
```

The single failure is environmental, not behavioural: `test_the_stored_shape_still_reads_as_this_schema`
(`tests/test_general_grouping_and_json_degree.py:1147`) calls `path.read_text()` without an
encoding, and this machine's default locale encoding is GBK, so a UTF-8 `plan.json` raises
`UnicodeDecodeError`. Re-running that one test with `PYTHONUTF8=1` passes (`1 passed in 5.30s`).
**This is a test-portability defect (a missing `encoding="utf-8"`), not a translation-core
behaviour.** The project's full suite was already running in another process per the brief; I did
not start a second full run.

In-process probes (temp script, run outside the repo, since deleted) confirmed
`Base._check_batch` raising on wrong count (`expected 3 translations, got 2`) and on an empty slot
(`empty translation for non-empty paragraph(s) [1]: batch alignment lost`) while returning `None`
for an equal-count shift with no empty slot — i.e. the documented blind spot is real and reachable
through the public API.

Counts produced by `Select-String` over `book_maker/**.py` (81 files):
`Measured` 97, `CHOSEN|Chosen` 46, `not measured` 6, files containing `owner ruling` 14,
`CompatRule(` entries in `cli.py` 53, `^    def translate(` in `book_maker/translator/*.py` 12,
`^    def translate_list(` 5.

### Verified by reading only **[V]**

All file:line citations in §2 and §3. These are static readings of the implementation at this
commit and are reproducible by opening the cited lines.

### Not verified — could not be run here

- **Any live request against a real endpoint.** No API key or reachable endpoint was available, so
  every probe, ladder, demotion and negotiation behaviour is verified only through its tests and
  its code. The tests I ran are the mock-level ones.
- **`tests/test_vision.py`, `tests/test_chatgptapi_translator.py`, `tests/test_codex_translator.py`,
  `tests/test_session_handoff_snapshot.py`, `tests/test_glossary_wiring.py`,
  `tests/test_session_context.py`, `tests/test_session_mode_translator.py`** — named in the brief or
  used in §6's checks; I read their test-name inventory (where I needed it) but did not execute
  them (suite time budget, and a full run was already in flight). Their claims are therefore **[U]**
  from my side. §6 check 4 in particular is untested by me.
- **The batch-API path** (`--batch` / `--batch-use`), §4.7.
- **The `codex` route end-to-end**, which needs the `codex` binary and a ChatGPT session. Only the
  client's protocol layer and the translator's flags were read.

### Provenance I could not check

The code cites dated documents and eval cells that are **not in this checkout**:
`docs/260925-docs-SKILL_FIELD_TEST_FRICTIONS.md`, `docs/260827-feat-CODEX_TRANSLATOR_PROVIDER.md`,
`docs/260830-fix-CODEX_SIDECAR_HARDENING.md`, `CLAUDE.md` — all absent (`Get-ChildItem -Recurse`
found none). The two evaluation write-ups that *are* present
(`docs/en/evaluation/grouping-batch-size.md`, `docs/en/evaluation/session-compact-budget.md`)
corroborate the two largest numeric constants in the translator core and their measured/chosen
labels. The specific eval cells quoted inside `epub_loader.py:2447-2453` and `2508-2525`
(`s-child-u48`, `ds-waste-u96-b4800`, `o4m-waste-u256-b1600`, `ds-waste-u64-b4800`) have **no
in-repo artefact**, so those measurements cannot be checked from this repository — only their
effects (the shipped detectors and their tests) can.

### Unknown

- Whether the `_numeric_slot_shift` and `_marker_slot_mismatch` thresholds
  (`_SHIFT_MIN_WITNESSES`, the marker-ownership rule) fire too rarely or too often on real books.
  Only a run over a corpus would say. The tests pin the intended behaviour, not the base rate.
- Whether session mode's cost/continuity trade-off (P4's clearest case) holds on the local-device
  models the ruling was written for. The evaluation doc says explicitly it was not measured there
  (`session-compact-budget.md:69`).
- Whether the gemini per-line fallback (§4.1) is truly unreachable.

---

## 6. Rerunnable checks

Anyone can reproduce the load-bearing claims without an API key:

1. **Contract**: `python -m pytest tests/test_batch_groups_and_markers.py::TestBatchMismatchContract tests/test_batch_groups_and_markers.py::TestDivideLadder -q`
   — the count/empty-slot contract and the 8→4→2→1 halving shape.
2. **Capability taxonomy**: `python -m pytest tests/test_capabilities.py -q` — fatal vs deferred vs
   capability answers, one probe per model, one probe under parallel workers, streak demotion.
3. **Patience**: `python -m pytest tests/test_translate_with_backoff.py -q` — no attempt cap, fatal
   classes stop immediately, waits bounded, every retry logged.
4. **Session state**: `python -m pytest tests/test_batch_layers.py tests/test_session_context.py -q`
   (I ran only the first) — one exchange recorded per batch, a record of exactly what was sent, a
   misaligned batch recorded not at all.
5. **Cross-check a constant against its eval doc**: read `book_maker/session_context.py:76-116`
   beside `docs/en/evaluation/session-compact-budget.md:45-51`. The two tables should agree
   (33/283,361/8 → 63/360,681/9 → 52/396,197/9).
6. **Check the fragile test I found**:
   `python -m pytest tests/test_general_grouping_and_json_degree.py -q -k stored_shape` fails on a
   GBK-default Windows box and passes under `PYTHONUTF8=1`.
7. **Check where the contract is raised and whether any route repairs it**:
   `grep -n "raise BatchMismatch" book_maker/**/*.py` returns 10 sites in 3 modules
   (`base_translator.py:1219,1228`; `chatgptapi_translator.py:1784,1791,1795,1845,1949,1952,1956`;
   `gemini_translator.py:593`), and `grep -n "except BatchMismatch" book_maker/**/*.py` returns the
   loader ladder plus `gemini_translator.py:505-508` and `chatgptapi_translator.py:1740-1746`, both
   of which re-raise.
8. **Check the loader is the only repair site**: `grep -n "_divide_and_translate" book_maker/**.py`
   — one definition and three call sites (`epub_loader.py:2592`; calls at `:2397`, `:2429`,
   `:2438`), all inside `_translate_texts_aligned`.
9. **Check the numeric-shift threshold**: `_SHIFT_MIN_WITNESSES = 2` (`epub_loader.py:208`), used
   at `:2567`.
