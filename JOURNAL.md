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

**Reproduction commit link:** see the commit that added this section (linked in the PR/branch history)

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

**PLAN.md link:** [PLAN.md](PLAN.md)

**Walkthrough video (recommended):** _(not recorded)_

**Blockers or open questions:**
Main open question is how aggressive to make TypeScript detection without causing false
positives on plain JavaScript. I also noticed a pre-existing wart: the Python check
`:\s*(int|str|float|bool|list|dict)` has no word boundary, so a TypeScript annotation like
`: string` matches `str` and the sample gets tagged as Python. That's outside this issue,
so I don't plan to fix it, but I want to make sure my change doesn't make it worse.
