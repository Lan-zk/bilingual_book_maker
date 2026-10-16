# 独立复核：`design-philosophy/report.md`

**复核对象**：`design-philosophy/report.md`（50834 字节，408 行）
**被研究代码版本**：commit `e1027a4823e98f668ef950740b6f805ec30bd3b2`（`git rev-parse HEAD` 实测一致；工作树除 `design-philosophy/` 外干净）
**复核环境**：Windows，Python 3.12.3，pytest 7.4.3，`locale.getpreferredencoding(False)` = `cp936`；`docling`/`docling-core`/`pypandoc` 不可导入、`pandoc` 不在 PATH（与报告 §6.2 一致）
**复核方式**：逐处打开 cited 文件行范围对照原文；用 `python -c` / `ast` 重新导出数字；实跑 `--plan-dry-run`、`--plan-classify agent`、手工改 plan 行、候选"静默忽略"参数、区块测试、一次完整套件（`PYTHONUTF8=1`）。未改动被研究项目的生产实现；所有临时产物（`test_books/animal_farm_plan.json`）已删除，`git status --porcelain` 仅剩 `?? design-philosophy/`。
**结论口径**：下文区分【报告错误】【报告未证实】和【报告正确，已确认】。

---

## A. 引用抽查

抽查 **137 处** `file:line` 引用（覆盖 §1–§7，加权到承重结论：规则表、ledger 状态机、plan 门禁、指纹、PDF 复用、能力探测、披露/脱敏、测试契约）。其中：

- **126 处与引用句相符**（逐行核对通过）；
- **7 处不一致**（下 A-1）；
- **4 处部分支持**（引用范围不能完全支撑该句，下 A-2）。

相符的引用（逐处打开过 cited 行范围）：`docs/en/quickstart.md:47,59,62,85`；`pyproject.toml:72,113`；`make_book.py:1-4`；`translator/__init__.py:22-54`；`loader/plan.py:1069-1106,1235-1249,1417-1601,1466-1474,1615-1640,1643-1654,1681-1682,1847-1924,2101-2158,2178-2226`；`loader/ledger.py:1-37,16-31,43-53,98-164,368-392,402-422,424-530,443-452`；`loader/classify/__init__.py:17-20,43-51,80-84,104-119`；`loader/classify/all.py:1-16`；`loader/classify/agent.py:1-13`；`loader/epub_loader.py:855-880,1327-1365,1367-1400,1508-1517,1598-1614,1701-1736,1739-1767,2356-2438,2507-2545,3138-3140,4214-4239`；`loader/disclosure.py:1-13,14-21,41-44,80-91`；`loader/helper.py:493-505,562-581`；`translation_metadata.py:9-14,28-43`；`redaction.py:1-12`；`cli.py:248-274,366-422,591-631,778-781,836-960,874-936,939-960,1148-1264,1224,1891-1909,1949-1974,1956-1966,1977-2021,2118-2137,2182-2856,2989-2999,3083-3095,3161-3270,3172-3228,3270,3371-3372,3594-3595,3653-3683`；`translator/base_translator.py:56-66,152-156,501-504,1201-1232,1345-1351`；`translator/chatgptapi_translator.py:650-679,1775-1802,1804-1811`；`translator/capabilities.py:76-94,344-448`；`translator/vision.py:63-70`；`translator/gemini_translator.py:449-451,522-558,560-577`；`pipeline/to_epub.py:46-94,171-195,246-258`；`pipeline/stages.py:43,92,168-192,216-229,242-324`；`pipeline/docling_parser.py:1195-1247,1313-1316`；`pipeline/importer.py:36-63`；`pipeline/messages.py:41-47`；`pipeline/pdf_settings.py:122-141`；`pipeline/pdf_figures.py:671-689`；`pipeline/epub_nav.py:190-229`；`pipeline/epub_export.py:138-148`；`pipeline/pdf_common.py:1-9`；`docs/en/features/plan-mode.md:38,40`；`docs/en/evaluation/grouping-batch-size.md:5,70,74`；`docs/en/evaluation/session-compact-budget.md:63`；`tests/test_flag_compat.py:1265-1281,1289-1298,1518-1541`；`tests/test_cli_documentation.py:69-75,121-188`；`tests/test_batch_groups_and_markers.py:298-317`；`tests/test_translate_with_backoff.py:85-86,148-154`；`tests/test_capabilities.py:172-195`；`tests/hermetic/sitecustomize.py:1-42`；`tests/test_provider_loader.py:155`、`tests/test_docling_adapter.py:391`（两者的失败文本与报告描述逐字对上）。

### A-1 不一致（7 处）

1. `[MISMATCH]` §2「注释把理由写成了原则："a warning about a run that is not going to happen is noise in front of the reason it isn't"」| cited `cli.py:1006-1007`、`2122-2127` | 实际是 **"advice** about a run that is not going to happen is noise in front of the reason it isn't"（`cli.py:1006-1007` 与 `cli.py:2123-2124`），报告把 `advice` 引成了 `a warning`。
2. `[MISMATCH]` §2 场景 B「区域角色判定显式排除 `page_header`/`page_footer`（`decisions.py:81-88`）」| cited `decisions.py:81-88` | 实际是 `ELIGIBLE_SOURCE = {text, paragraph, footnote, caption, section_header, title}` 白名单；`page_header`/`page_footer` 在 `decisions.py` 只出现于 `:139-140` 的颜色表，**不存在显式排除**（是白名单里"没有它"）。
3. `[MISMATCH]` §3.4「67 个 help 文本中有 51 个是 `build_parser` 里的内联字面量，只有 14 个引用常量」| 实测（AST）：`add_argument` 70 次，其中 5 次用 `argparse.SUPPRESS`，**带 help 的调用 65 个**；内联字面量 51 + 非常量 14 = 65。51/14 正确，但基数是 65 不是 67（67 是长选项数，51+14=65≠67，句子内部也不自洽）。
4. `[MISMATCH]` §3.1/§6.1「27 处 `exit(1)`/`raise SystemExit`，其中只有 10 处是表」| 实测 27 处正确（16 处 `exit(1)` + 11 处 `raise SystemExit`），但落在 `check_compatibility` 表里的**只有 1 处**（`cli.py:2127`）；"10"指的是 10 条 `stop` 规则，不是 10 个退出点。
5. `[MISMATCH]` §2 场景 B 验证限制「180 个…离线测试通过（… `test_bundle_pipeline`+`test_pdf_source_edit_guard` 110）」| 实测（`PYTHONUTF8=1`）：`test_bundle_pipeline` **18 passed, 48 skipped**；`test_pdf_source_edit_guard` **0 passed, 22 skipped**（22 个全部因 `pandoc is not available on PATH` 跳过）→ 该对合计 **18 passed / 70 skipped**，不是 110。同一句括号里的清单相加为 355，与该句的"180"也不符；11 个文件实测 passed 合计 **263**。
6. `[MISMATCH]` §4.5「`Measured` 出现 97 次，`CHOSEN`/`Chosen` 出现 46 次」| 实测 `book_maker/**/*.py`（不分大小写）：`measured` **99 次 / 97 行**，`chosen` **47 次 / 46 行**；区分大小写：`Measured` **27** 次，`CHOSEN|Chosen` **14** 次。97/46 是"含该词的行数"，不是出现次数，且只对 `book_maker/**/*.py` 成立（全仓库 `Measured` 41 次/37 行）。同句的「`owner ruling`/`lead ruling` 出现在 15 个文件里」与「20 个不同日期戳」实测正确。
7. `[MISMATCH]` §6.3「同一台机器设 `PYTHONUTF8=1` 重跑：**4 failed, 3091 passed**, 333 skipped, 51 deselected, 3 xfailed（22 分 41 秒）」| 我实跑完整套件：**3 failed, 3092 passed, 333 skipped, 51 deselected, 3 xfailed（26 分 39 秒）**。失败的 3 个与报告点名的前 3 个完全一致（`tests/test_cli.py::test_an_unknown_provider_names_both_config_files`、`tests/test_provider_loader.py::TestUnknownProvider::test_the_error_names_both_files`、`tests/test_docling_adapter.py::test_a_picture_reference_is_a_path_beside_the_markdown`，断言文本 `.bbm/providers.json` vs `C:\Users\lzk\.bbm\providers.json`、`![](images/p1.png)` vs `![](/work/x/images/p1.png)`）；报告说的第 4 个顺序相关失败（`test_loader_compat_guards.py::test_a_test_run_is_told_classification_covers_the_whole_book`）**在我这次运行中通过**——与报告自己说它"顺序相关"一致，但报告把它写进了基线数字。

### A-2 部分支持（4 处）

8. §1.2「EPUB 在 LLM 路由上默认开启」| cited `docs/en/features/plan-mode.md:5` | 该行讲的是"分割全书→按 signature 问模型→写 plan JSON→之后每次运行复用"；"默认开启"在**同文件 `:7`**（"Plan mode is on by default for an EPUB on any LLM route"）。结论正确，引用错位一行段。
9. §1.1「输出是同名的 `_bilingual` 文件，放在原文件旁边」| cited `docs/en/index.md:3`、`quickstart.md:47,58,62,85` | `index.md:3` 只支持"bilingual edition（原文段落 + 译文）"，不涉及文件名/位置；`quickstart.md:58` 是**空行**（`- Output: the_little_prince_bilingual.txt` 在 `:59`）。
10. §3.1「专门断言"一条普通的 epub/txt/md/机器翻译命令什么都不触发"（`TestNoiseGuard`，`tests/test_flag_compat.py:934-995`）」| 该类的用例是 epub、txt、pdf(context+glossary)、epub+`--test`、epub+google（MT）、以及一个走 CLI 的 google 默认运行 —— **没有 md 用例**。
11. §7.1 把 `translator/capabilities.py:129-154` 标为"分级" | 该处实际是错误类型分级（`PROBE_FATAL_ERRORS`/`PROBE_TRANSIENT_ERRORS`/`RUNG_REFUSAL_ERRORS`）；§4.4 说的"判定结果按用途分级（strict / shape / json）"在 `chatgptapi_translator.py:650-679`。索引标签不精确，正文正确。

---

## B. 数字重算

| 报告断言 | 实测 | 判定 |
|---|---|---|
| 70 个 `add_argument` | 70（`ast` 统计 `build_parser` 内 `add_argument` 调用） | 正确 |
| 74 个选项串 | 74 | 正确 |
| 72 个长选项 | 72（去重后仍 72） | 正确 |
| 67 个经 `SUPPRESS`/deprecated 过滤 | 67（`test_cli_documentation._long_cli_options()`）；被过滤的 5 个是 `--poetry-group-size --plan-classify-model --batch_units --with-ocr --no-gpu` | 正确 |
| `COMPAT_RULES` 51 行，10 stop / 41 warn | 51；`Counter({'warn': 41, 'stop': 10})`；stop = `A1 A4 A7 A13 A14 A15 A16 A10 A12 C9` | 正确 |
| `DRY_RUN_RULES` 4 行 | 4：`B2 B3 C31:dry-run C32:dry-run` | 正确 |
| 27 处 `exit(1)`/`raise SystemExit` | 27（16 + 11） | 正确 |
| 同上「只有 10 处是表」 | 表内 **1 处**（`cli.py:2127`）；10 是 stop 规则条数 | **错误** |
| `pipeline/messages.py` 125 个大写常量 | 125（AST 顶层赋值全部大写，唯一；单行正则只能匹配 124） | 正确 |
| `Measured` 97 次 | 出现 99 / 行 97（`measured` 不分大小写，`book_maker/**/*.py`） | **说法错误** |
| `CHOSEN\|Chosen` 46 次 | 出现 47 / 行 46（不分大小写；区分大小写 14） | **说法错误** |
| `owner ruling`/`lead ruling` 15 个文件 | 15 个文件（`book_maker/**/*.py`，38 次） | 正确 |
| 20 个不同 `26MMDD` 日期戳（`book_maker/**/*.py`） | 恰好 20：`260807 260811 260813 260827 260830 260902 260904 260905 260906 260907 260913 260914 260920 260921 260922 260923 260924 260925 260928 260929` | 正确 |
| `book_maker/` 下 12 处 `read_text(` 全带 `encoding="utf-8"` | 12 处，AST 校验 `read_text` 调用 12 个、无一带省略 encoding 的 kwarg | 正确 |
| `--help` 534 行 | 534（`python make_book.py --help \| Measure-Object -Line`） | 正确 |
| `--plan-dry-run`：20 documents / 202560 chars / coverage 100.0% / `block:p.calibre_13` 285 units 188081 chars 92.9% / `batches: 402 unit(s) in 51 request(s) (<= 16 units / 1200 tokens per request)` / `skipped characters: excluded-tag=3` | 全部逐字复现（命令与输出见下"复核命令"；额外还有一行 `Classifier: gpt-6-luna …` 与 B3 警告，报告以 `...` 省略） | 正确 |
| 两次 dry run 的 plan JSON 逐字节相同，SHA256 `7BAAFFC7…836` | `7baaffc7e401d78186202722372c9f9c6f5404b41f25ea1bfd3a8650a7c85836`，两次完全相同 | 正确 |
| `--plan-classify agent` 退出码 3、31 个待决行 | 干净 plan 31 行 signature；退出码 **3** | 正确 |
| 手改前 3 行为 `translate/user/prose` 后被保留，提示变 28 行未决 | 实跑输出 `using existing plan test_books/animal_farm_plan.json`、`28 row(s) are still undecided …`、退出码 3 | 正确 |
| 第 4 行 `action:"translate"` + `decided_by:null` 被拒，文案同引用 | 逐字复现（`block:blockquote.calibre_17: action is "translate" with decided_by null. … Fix them in that file, or delete it and rerun to regenerate a fresh plan.`），退出码 1 | 正确 |
| 场景 C 的 A1 拒绝文案与退出码 1 | 实跑 `--api_format groq --key sk-test --model m --batch`：输出与引用逐字一致，退出码 **1** | 正确 |
| §2 场景 B 各文件离线通过数：`test_pdf_headings` 24、`test_pdf_figures` 87、`test_pdf_formula` 24、`test_pdf_settings` 10、`test_pdf_render` 8、`test_pdf_ocr_replace_layer` 5、`test_pdf_structure_wiring` 17、`test_epub_nav`+`test_pdf_to_epub_cli` 70 | 24 / 87 / 24 / 10 / 8 / 5 / 17 全部对上；`test_epub_nav` 6 passed + `test_pdf_to_epub_cli` 64 passed = **70** | 正确 |
| 同句 `test_bundle_pipeline`+`test_pdf_source_edit_guard` 110 | 18 passed + 0 passed（70 skipped） | **错误** |
| 同句「该模块 180 个…通过」 | 括号清单相加 355；11 个文件实测 passed 合计 263 | **错误/自不相容** |
| §6.3 `tests/test_pdf_decisions.py` 的 31 个测试整体跳过 | 该文件 `def test` 共 31 个；实测 pytest 报 **1 skipped**（模块级 `importorskip` 使整模块计 1 个 skip）。报告"31 个测试整体跳过"是对文件内容的静态描述，与 pytest 计数口径不同 | 需注意口径（不算错误） |
| §6.3 默认套件 52 failed / 3043 passed / 333 skipped / 51 deselected / 3 xfailed | **未跑**（按任务约束只跑了一次 `PYTHONUTF8=1`）。机制已验证：不设 `PYTHONUTF8` 时 `tests/test_translation_plan.py` 单文件即 18 failed / 217 passed，失败信息为 `UnicodeDecodeError: 'gbk' codec can't decode byte 0xa6`（`Lib/pathlib.py:1028`） | 未验证（机制相符） |
| §6.3 `PYTHONUTF8=1`：4 failed / 3091 passed | **3 failed / 3092 passed** / 333 skipped / 51 deselected / 3 xfailed，26 分 39 秒 | **数字不符**（见 A-1 第 7 条） |
| §6.3 三个 marker 被 `pyproject.toml:113` 排除 | `addopts = "-m 'not corpus and not live_provider and not pdf_route_ci'"` | 正确 |
| §3.1「`A2`、`C15`、`C17` 在整个仓库中不出现」 | `"C15"`/`"C17"` 0 命中；`"A2"` 仅在一个测试数据字符串 `["a1", "a2", …]` 上大小写不敏感地误命中（`tests/test_batch_groups_and_markers.py:305`）→ 作为规则 id 确实不出现 | 正确 |
| §3.1「规则 id 引用的审计文档 `docs/2609*-*.md` 不在 checkout 里」 | `docs/2609*` 无命中；仓库无 `CLAUDE.md` | 正确 |

### 复核命令（可直接重跑，均在仓库根目录）

```powershell
# A. 结构计数
python -c "from book_maker.cli import COMPAT_RULES as C, DRY_RUN_RULES as D; import collections; print(len(C), collections.Counter(r.level for r in C)); print([r.id for r in D])"
python -c "import ast,pathlib,re; s=pathlib.Path('book_maker/cli.py').read_text(encoding='utf-8'); print(len(re.findall(r'exit\(1\)|raise SystemExit', s)))"

# B. plan 预览与复现性（跑完删掉 gitignored 产物）
python make_book.py --book_name test_books/animal_farm.epub --plan-dry-run | Out-Null
python -c "import hashlib;print(hashlib.sha256(open('test_books/animal_farm_plan.json','rb').read()).hexdigest())"
Remove-Item test_books/animal_farm_plan.json -Force

# C. 交接退出码
$env:OPENAI_API_KEY="sk-fake"; python make_book.py --book_name test_books/animal_farm.epub --plan-classify agent; Write-Output "exit=$LASTEXITCODE"
Remove-Item test_books/animal_farm_plan.json -Force

# D. 完整套件（本次实测 3 failed / 3092 passed / 333 skipped / 51 deselected / 3 xfailed）
$env:PYTHONUTF8="1"; python -m pytest tests/ -q
```

---

## C. 过度解读与自相矛盾

按"最能影响读者的误读"排序：

1. **【报告错误·承重】** §3.1/§6.1 把"静默忽略类"说成只有 2 个成员（`--pdf_layout`、`--interval`），并作为"表是有意收窄"的证据。我另测出一个同类成员：`--context_paragraph_limit` 在**没有** `--use_context` 时既不触发表（B11 的谓词要求 `context_mode == "session"`，`cli.py:1364-1365`）、也没有运行期警告——实跑 `--plan-classify agent --context_paragraph_limit 5` 输出零警告、退出码 3；还有 `--temperature` 在固定机器翻译路由（google/deepl/caiyun/tencent/customapi）被接受后丢弃：`GoogleTranslator.__init__(self, key, language, **kwargs)` 从不读它，表里没有任何一行提到 `temperature`，help 只写了"on the formats that take one"并只点名 codex 会忽略。这一条与报告自己的研究笔记冲突：`research/cli-surface.md:229` 明确写"我没有枚举每个 flag × 书类型 × 路由的组合，'静默忽略'类的真实大小未测"。正文把未测的类写成"2 个"，是把研究笔记的保留意见升级成了结论。
2. **【报告错误】** §3.1/§6.1 的"27 处退出点中只有 10 处在表内"给 §3.1 的反例提供了错误的量化支撑（真实：表内 1 个退出点、10 条 stop 规则）。它不影响"表不是唯一拒绝来源"这个定性结论（§5 列出的 5 处表前拒绝实测就是 5 处），但让读者对"表内外比例"形成错误印象。
3. **【报告错误】** §2 场景 B 的"180 个离线测试通过"及其括号清单（含 110）作为"验证限制"段落里的实测依据出现，但清单相加 355、实测 passed 263。这段是读者判断"PDF 路由有多少行为被真正验证过"的唯一依据；数字偏大意味着报告暗示的验证强度高于我实际复现到的强度（剩余部分靠 pandoc 缺失而跳过）。
4. **【报告错误】** §4.5 的 `Measured`/`CHOSEN` 计数以"出现次数"和"对整个包计数"表述，实际只在 `book_maker/**/*.py` 内、且只有按"行数"口径成立（99/97、47/46）；区分大小写的字面量计数是 27/14。§4.5 的核心论证（"测出来的 vs 选的"在具体处承重、被用来接受更差的数字）不受影响——`session_context.py:76-116` 与 `plan.py:1423-1480`、`docs/en/evaluation/*.md` 都逐字支持——但作为"量化证据"的这两组数字站不住。
5. **【报告未证实·把设计声明读成事实】** §3.5「明确不写的是操作者、机器、命令行、构建号（`translation_metadata.py:9-14`、`28-43`）」只引了"移除"那一侧的裁定；同一个类里的 `_wants_translation_metadata` docstring（`epub_loader.py:829-832`）却写"记录回答的问题（which build, which model, which endpoint, which command）正是几天后会出现的那些"——**同一文件里两段 docstring 互相矛盾**，报告没有发现，也没有交叉核对。另外报告说记录文件"只有两个字段"：`translation_metadata.py:91` 的 `RECORD_KEYS = {generator, model, date, glossary-sha256}` 是**最多四个键**（`epub_loader.py:862` 的"Two facts and no more"说的是"事实"而不是文件键集）。方向的判断（可审计性优先于可追溯性）有充分证据，但"只有两个字段"不准确。
6. **【报告未证实·从单点外推】** §3.4「这一点在代码注释里被反复作为理由写出」（指"人 + 照 skill 操作的编码 agent"两类读者）没有给任何行号；我只在 `tests/test_cli_documentation.py:1-9,118-120` 与 `sitecustomize.py:20-27` 找到同向的注释，够支持"存在这个理由"，不够支持"被反复作为理由写出"（"反复"未被量化）。
7. **【报告自相矛盾】** §3.4 反例句内 51+14≠67；§3.1 的反例段与 §6.1 第 1 条重复同一个错误计数；§2 的"180"与它自己的括号清单不相容。这三处都是能在文内自查发现的算术矛盾。

未见其他跨节矛盾：§1.3 的"14 种路由"与 `FORMAT_DICT`（14 项）一致；§2 的 31 行/28 行/退出码 3 与实跑一致；§5 表格里的"已验证"标记与我的复现结果一致（除"请求数与 plan 完全一致"确认为真、PDF 相关标注为"部分验证/未验证"也恰当）。报告对"未验证"的自我标注（§2 场景 B、§3.5、§6.2）经核对是诚实的：我无法运行的部分（真实抽取、真实翻译、真实 `--resume`）报告也没有声称验证过。

---

## D. 反例搜寻

### D-1 静默忽略、无规则、无警告的选项

- `[已确认] --pdf_layout`：`cli.py:3371-3372` 只在 `book_type == "pdf"` 时下发；`COMPAT_RULES` 全文不含 `pdf_layout`；实跑 `--plan-classify agent --pdf_layout side-by-side`（epub）零警告、退出码 3。报告中此条正确。
- `[已确认] --interval`：`cli.py:3594-3595` 只在 gemini 路由生效；表内无 `interval`；实跑零警告。报告此条正确。
- `[新反例·已确认] --context_paragraph_limit` 在无 `--use_context` 时：唯一的规则 B11（`cli.py:1361-1371`）谓词要求 `context_mode == "session"`；实跑零警告、退出码 3，且该值在窗口模式关闭时不被读取（loaders 只在 `context_flag` 为真时使用）。这属于报告定义的同一类，但报告说"只有 2 个"。
- `[新反例·静态证据] --temperature` 在固定机器翻译路由上：`COMPAT_RULES`/`DRY_RUN_RULES` 全文不含 `temperature`；`endpoints.build_translator` 把它作为 kwarg 传给所有路由（`endpoints.py:536`），而 `GoogleTranslator`/`DeepL`/`Caiyun` 等的 `__init__(self, key, language, **kwargs)` 从不读它；help（`cli.py:2744-2748`）只点名 codex 会忽略。未构造实跑（会真的调用机器翻译端点），标为"未证实但证据强"。
- `[确认无此问题]` 我试过的其他越界组合都有规则或运行期警告：`--pages`、`--device`、`--ocr-engine`、`--img-model`、`--batch_size`、`--no-formula-images`、`--ocr-replace-layer`、`--pdf-image-dpi 300`（C37；改成默认值 200 时本来就无需警告）、`--source_lang`、`--classify-model`（C33/C32）、`--translation_color`/`--translation_style`（C8 + 运行期）、`--context-compact-at`（运行期）、`--extra_body`/`--extra_headers`（运行期）、`--sentence_mode`+plan（表内 stop）。

### D-2 决策已作出却未进入 ledger 记录

- `[已确认，范围很窄] --plan-classify all`：`decide_everything()` 在内存里给每一行写 `decided_by="all"`、`content_type="unclassified"`（`classify/all.py:22-35`），但 `MODE_POLICY["all"] = ModePolicy("all", False, False, 1)`（`classify/__init__.py:81`）意味着 `writes_plan_file=False`，`epub_loader.py:1643-1646` 因此跳过写盘。结果是：**all 模式下"谁做的决定"这一行级记录随进程消失，磁盘上没有任何 ledger 留下**。报告 §3.2 承认了 all 模式的文件策略（"除 `all` 模式外"），但同一段仍把"每一行都写成 `decided_by="all"`、`content_type="unclassified"`"作为可审计性的证据。其余模式会写盘（`plan.record_dispositions` + `save_json`：`epub_loader.py:1540,1587-1588,1648,1661,1906-1907,1935-1936`），这一条不适用于 model/agent 模式。

### D-3 失败静默降级、什么都不打印

- `[已确认] docling 版本不可读时，溯源行静默消失`：`stages._installed_docling()` 在异常时 `return None`（`stages.py:232-239`），`_other_runtime` 只在 `new_version is not None and old_version != new_version` 时返回行（`stages.py:216-229`）→ 版本读不到就**不打印任何东西**，而复用仍然发生。报告 §2 场景 B 第 3 条写"docling 版本与设备只是溯源，**会打印一行**"，在"版本可读"的前提下成立。
- `[已确认] 提示开销测量失败时，分组预算静默落到下限`：`ChatGPTAPI.prompt_overhead_tokens()` 捕获一切异常并 `return None`（`chatgptapi_translator.py:1604-1630`），调用方随后用手里的 floor 分组，**没有任何一行说明探测失败了**。报告 §3.3 强调"预览把不确定度打印出来"（那说的是 dry run，`cli.py:3083-3095`）；真实运行里这条降级路径是无声的（代码注释自认为"A sizing hint, never a gate"，属有意为之，但它确实是"失败静默降级"的一个实例）。
- `[不构成反例]` gemini 的 `[Translation unavailable]` 标记并非静默：写回前会打印 `[Translation failed for this paragraph]`，致命错误会打印 `Fatal translation error detected. Aborting translation.`（`epub_loader.py:2403-2408,2869-2873`）；进度条读数被吞掉的两处（`epub_loader.py:2698,2710`）带有"never a reason to stop the run"注释，属显示层。

### D-4 路由自我修复"坏批量回复"，与"系统里唯一的回退点"相矛盾

- `[已确认·报告已自曝] gemini_translator.py:560-577`：解析不出结果时打印一行并逐个翻译（`_batch_translate` 的 fallback）；报告 §6.1 第 2 条已记录，并诚实标注"没有构造出能到达它的真实调用"。
- `[新反例·已确认，报告未收录] chatgptapi_translator.py:1747-1756`：结构化批量路径在**非** `BatchMismatch` 的异常（拒绝、重试后仍失败的传输错误、以及本路由内解析回复时的普通异常）上打印一行并 `return [self.translate(t, False) for t in text_list]` ——即**在路由内部把一次失败的批量请求降级成逐段翻译**。它与 `epub_loader.py:2359` 自称的"The one fallback in the system"不一致。可区分的是：这条路径**不**吞掉对齐错误（`except BatchMismatch: raise`，1740-1746），所以 §4.3 的"数量/对齐错误只有 loader 的梯子能修"仍然成立；但"唯一修复点"这句话本身在两条路由上都不严格成立（与报告 §6.1 第 2 条同类，只是成员更多）。
- `[已确认] loader 侧的梯子确实只有一处**对齐**修复`：`grep -n "except BatchMismatch"` 全仓库 5 处——`epub_loader.py:2389`（梯子）、`helper.py:576`（tag 模式逐条补救）、`chatgptapi_translator.py:1740`（立即 `raise`）、`gemini_translator.py:82`（重试谓词，不捕获）、`gemini_translator.py:505`（立即 `raise`）；`raise BatchMismatch` 共 10 处、分布在 3 个模块（`base_translator` 2、`chatgptapi_translator` 7、`gemini_translator` 1），与报告 §4.3 的"3 个模块、10 处抛出"及"两个捕获它的路由都立刻重新抛出"一致。

---

## 未能核对的部分

1. **默认（不设 `PYTHONUTF8`）完整套件基线** 52 failed / 3043 passed / 333 skipped / 51 deselected / 3 xfailed：按任务约束只跑了一次（`PYTHONUTF8=1` 那次），默认基线数字未复现；只复现了机制（`test_translation_plan.py` 单文件在 cp936 下 18 failed，报 `UnicodeDecodeError: 'gbk' … byte 0xa6`），因此"47 个是区域设置问题"这一拆分仍属报告自称、未经我核对。
2. **`PYTHONUTF8=1` 下第 4 个失败的顺序相关性**：我这次运行它通过，无法在"只跑一次"的约束下验证它在报告那次运行中确实失败（报告自己说它顺序相关，属可信但未复现）。
3. **任何真实翻译 / 真实端点行为**（探测、会话压缩、价格统计、`--resume` 真续跑）：与报告一致，本环境未验证。
4. **任何真实 PDF 抽取 / EPUB 生成 / 导航校验**：`docling`/`pandoc` 缺失，未验证；相关离线测试通过数我已复现（见 B 表），但 `test_pdf_source_edit_guard` 的 22 个用例在本环境**全部因缺 pandoc 跳过**——报告把它计入"通过"是错的，这同时意味着 §4.8（人改产物护栏）在"已验证（探针脚本）"之外没有可复现证据在报告里给出。
5. **代码注释里的"实测"数字**（`plan.py`、`session_context.py`、`translation_metadata.py` 引用的 260905/260906/260907 eval cell）：被引用的 `docs/2609*-*.md`、`CLAUDE.md` 均不在 checkout（已核实不存在），无法回溯；`docs/en/evaluation/` 两篇可核对，我已核对，与报告引用一致。
6. **`--temperature` 在 MT 路由上的静默丢弃**：只做了静态证据（翻译器签名 + 表内无该行 + help 文本），未实跑端点确认。

---

## 总评（对报告可信度的净影响）

报告的**结构性事实几乎全部站得住**：51/10/41、4 条 dry-run 规则、70/74/72/67 参数面、ledger 两状态机与逐字拒绝文案、plan 门禁三处、对半切分梯子与"唯一回退点"的原话、指纹三件套、披露/脱敏的取舍、能力探测设计、`--plan-dry-run` 的可复现性与全部数字、场景 C 的文案与退出码、agent 退出码 3 与手工改行的行为——这些我都独立复现或逐行核对，**它们支撑报告的五条产品侧原则（§3.1–§3.5）与八条技术侧原则（§4.1–§4.8）**。

它的弱点集中在**数量与计数口径**上：`Measured`/`CHOSEN` 的计数、退出点"10 处在表内"、help 文本 51+14 对 67、场景 B 的 180/110、`PYTHONUTF8` 基线 4/3091、以及"静默忽略只有 2 个"。这些数字都是可被 `python -c` 直接推翻的类型，其中三处还会在文内自相矛盾（51+14≠67、180≠清单和、基线与自述的顺序相关失败冲突）。因此：**报告的定性结论（可审计性优先、决策外化、能力靠探测、契约在最上层、身份/溯源分离、数字标注来源）可以采信；凡是以具体计数作为证据强度的句子，读者应引用本文件 B 表重算后的数字。**
