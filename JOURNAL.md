# Module 3 Journal — PathReview

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/148

**Issue title:** Skill extractor fails to detect JavaScript and TypeScript

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The skill extractor is supposed to read code/text and figure out what skills it shows,
but right now it just doesn't pick up JavaScript or TypeScript at all. When I dug into
`ingestion/parsers/skill_extractor.py`, the `_detect_languages()` method only actually
has logic for Python — file extension, `import`, `def`, type hints, and so on. The
strange part is there's already a `JS_TS_KEYWORDS` set defined right there in the class,
but nothing in the method ever uses it. So if you feed it a JavaScript file you get
nothing back, and a TypeScript file only comes back as "React." A good fix means adding
a JS/TS detection block to `_detect_languages()` — checking for `.js`/`.ts`/`.tsx` files,
using the keyword set that's already defined, and catching TypeScript-only markers like
`interface`/`type` — so the tests in `tests/unit/test_skill_extractor.py` (like
`test_text_with_typescript_files`) finally pass.

**Selection notes — "Is this right for me?" checklist:**
- *Can I reproduce it?* Yeah — I ran `test_text_with_typescript_files` and it fails. It
  hands the extractor TypeScript code and expects a "typescript" skill back, which never
  shows up.
- *Do I understand the fix?* Pretty much. The Python block is basically a template I can
  follow, and half the work (the keyword set) is already sitting there, so I mainly need
  to wire in the JS/TS side.
- *Is the scope right for me?* I think so. It's all in one file, one method, and I can
  prove it works with `make test-unit` without spinning up the whole app. It doesn't reach
  into other modules, so it shouldn't snowball — feels like a solid Tier-1 to start on.

**Branch name:** `fix/148-skill-extractor-js-ts`

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

---

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/fortdominz/pathreview/commit/a9b035f333d381f2186e575deade3d49fe091ec5

**Reproduction summary:**
I reproduced it by running the extractor's unit tests on a clean checkout of my branch:

```bash
python -m pytest tests/unit/test_skill_extractor.py -v
```

Five tests fail, and two of them are exactly this issue:

```
FAILED tests/unit/test_skill_extractor.py::TestSkillExtractor::test_javascript_detection
FAILED tests/unit/test_skill_extractor.py::TestSkillExtractor::test_text_with_typescript_files
```

(Three others fail too — `test_database_technology_detection`, `test_devops_tool_detection`,
`test_docker_compose_detection` — but those are separate gaps, not this issue, so I'm
leaving them alone to keep my change scoped.)

`test_javascript_detection` feeds `const fs = require('fs')` and expects a "javascript"
skill back; `test_text_with_typescript_files` feeds an `export interface User { ... }`
sample and expects "typescript". Neither shows up in the results.

Digging in also sharpened my Week 7 read of the cause. I'd said `_detect_languages()` only
had logic for Python — that's not quite right. There *is* a JS/TS block, it's just too
narrow to ever fire:

- It looks for imports with `re.search(r"\b(import|require)\s+", text)`, which needs
  whitespace after `require`. Real code is `require('fs')` — bracket, no space — so it
  never matches.
- TypeScript is only ever picked by filename (`".ts" in filename`). These tests pass no
  filename at all, so TS can't be identified from the code itself.
- The class already defines a `JS_TS_KEYWORDS` set (`const`, `let`, `function`, `export`…)
  but nothing in the method uses it, so the most obvious content signal is ignored.

**PLAN.md link:** https://github.com/fortdominz/pathreview/blob/fix/148-skill-extractor-js-ts/PLAN.md

**Walkthrough video (recommended):** _(not recorded)_

**Blockers or open questions:**
Main open question is how aggressive to make TypeScript detection without causing false
positives on plain JavaScript. I also noticed a pre-existing wart: the Python check
`:\s*(int|str|float|bool|list|dict)` has no word boundary, so a TypeScript annotation like
`: string` matches `str` and the sample gets tagged as Python. That's outside this issue,
so I don't plan to fix it, but I want to make sure my change doesn't make it worse.

---

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
Implemented the fix for #148. I rewrote the JavaScript/TypeScript branch in
`_detect_languages()` (`ingestion/parsers/skill_extractor.py`) so it detects from code
*content* instead of almost never firing: I broadened the import check to match
`require('fs')` and `import … from`, added ES `export` / arrow-function / `console.log`
signals, wired in the previously-unused `JS_TS_KEYWORDS` set (requiring 2+ hits to avoid
false positives), and added TypeScript-specific markers (interface / type / enum /
implements, type annotations, generics) that win the label since TypeScript is a superset
of JavaScript. PLAN.md sub-tasks 1–3 are done.

**Next steps:**
Finish the regression tests (sub-task 4), run the full quality gate (sub-task 5), open the
PR, and document the repo's pre-existing failures.

**Blockers:**
The repo ships with heavy pre-existing failures (182 ruff, 103 mypy, 53 failing unit tests
on a clean checkout), and the pre-commit hook type-checks `tests/` even though the
project's own `make check` does not — so it trips on ~20 pre-existing un-annotated test
functions. Not a blocker to the fix itself; I'm handling it by documenting the pre-existing
state and confirming my change adds no new failures.

---

### Check-in 2 (end of week)

**PR link:** _(to be filled in when the PR is opened)_

**Branch:** `fix/148-skill-extractor-js-ts`

**What you built:**
Content-based JavaScript and TypeScript detection in the skill extractor. The old JS/TS
branch could almost never fire — its `require` check needed trailing whitespace,
TypeScript was chosen only by filename, and the `JS_TS_KEYWORDS` set was unused. My fix
detects both languages from real code signals, and TypeScript wins the label when
TS-specific markers are present.

**Tests added or updated:**
`tests/unit/test_skill_extractor.py` — the two issue tests (`test_javascript_detection`,
`test_text_with_typescript_files`) now pass, and I added five regression tests: CommonJS
`require()`, ES-module `import … from`, TypeScript detected from content with no filename,
TypeScript by `.ts` extension, and plain JavaScript is not mislabelled as TypeScript.

**Self-review confirmation:** [x] make check passes  [x] make test-unit passes
Interpreted per the module guidance for a repo with documented pre-existing failures —
"passes" means my change introduces **no new failures**. Baseline on a clean checkout:
**182 ruff / 103 mypy / 53 failing unit tests**. After my change: identical pre-existing
counts, **zero new ruff/mypy errors in the source files I touched** (verified by stashing
my changes and re-running), and failing unit tests dropped from **53 → 51** — I fixed 2 and
added 5 passing tests. Full detail is in the PR description.

**Draft PR feedback received from:** none (solo; open to Slack review before the deadline)
