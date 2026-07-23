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
