# JOURNAL

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/153

**Issue title:** Faithfulness checker crashes when a context chunk has `text: None`

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Selection reasoning:**
This is my first time working in a codebase this size, so I deliberately looked
for something small and self-contained rather than reaching for a Tier 2 or
Tier 3 issue. #153 fit that: it's confined to a single method (`check()`) in
one file, the root cause is a one-line default-value mistake
(`chunk.get("text", "")` doesn't catch a key that's present but explicitly
`None`), and the issue body already includes a minimal repro I can run on its
own — no database, API calls, or ingestion pipeline needed to reproduce or
verify it. There's also already a named failing test
(`test_none_context_chunk_text`) sitting in the suite, so I have a concrete,
objective way to confirm the fix is correct instead of guessing when I'm done.
Narrow blast radius, a runnable repro, and an existing test to check my work
against — that combination is exactly the kind of tightly scoped entry point
I wanted for a first issue in an unfamiliar project.

**Problem summary:**
The RAG evaluation subsystem includes a `FaithfulnessChecker` that scores whether
generated feedback is actually backed by the retrieved context. To build that
context, it pulls the `text` field out of each context chunk using
`chunk.get("text", "")`. That default only kicks in when the key is *missing* —
if a chunk has `text` explicitly set to `None` (which can happen upstream when a
document produces an empty or unparseable section), `.get()` returns `None`
instead of an empty string, and the following `" ".join(...)` call crashes with
a `TypeError` instead of degrading gracefully. A successful fix makes the
context-building step treat a `None` text value the same as a missing one, so a
single malformed chunk can't take down the whole faithfulness check. The affected
code lives in `rag/evaluator/faithfulness_checker.py`, and there's already a
named failing test for it (`test_none_context_chunk_text` in
`tests/unit/test_faithfulness_checker.py`) to confirm the fix.

**Branch name:** fix/153-faithfulness-checker-none-text

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/as5161/pathreview/commit/a6e345966a76200f9d575c410a820954b4f6e06b

**Reproduction summary:**
Ran the repro from the issue directly (`FaithfulnessChecker().check('Knows Python.', [{'text': None}])`)
and the named failing test `test_none_context_chunk_text` — both raise
`TypeError: sequence item 0: expected str instance, NoneType found` at
`faithfulness_checker.py:34`, confirming the root cause described in #153.

**PLAN.md link:** https://github.com/as5161/pathreview/blob/fix/153-faithfulness-checker-none-text/PLAN.md

**Walkthrough video (recommended):** [not recorded]

**Blockers or open questions:**
None currently — `relevance_scorer.py` has the same defensive-default pattern
but is out of scope for this issue; flagged in PLAN.md as a possible follow-up.