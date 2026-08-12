# Solution plan

**Issue:** [#153 — Faithfulness checker crashes when a context chunk has `text: None`](https://github.com/ascherj/pathreview/issues/153)

### Understand
`chunk.get("text", "")` only supplies the default `""` when the `"text"` key
is *absent* from the dict. If the key exists but its value is explicitly
`None`, `.get()` returns `None` as-is, and that `None` lands directly in the
list passed to `" ".join(...)`. `str.join` requires every item to be a
string, so it raises `TypeError: sequence item 0: expected str instance,
NoneType found`.

Expected behavior: a chunk with `text: None` should be treated the same as a
chunk with missing or empty text — it should contribute an empty string to
the concatenated context, not crash the whole check.

Actual behavior: `check()` raises an unhandled exception. Since `check()` is
called synchronously inside `EvalSuite.run()`, one malformed chunk takes down
the *entire* evaluation run for that review, not just the faithfulness score.

### Map
- `rag/evaluator/faithfulness_checker.py` (lines 34–36, inside `check()`) —
  the actual bug site; primary file to fix.
- `tests/unit/test_faithfulness_checker.py` — already contains
  `test_none_context_chunk_text`, currently failing red. This becomes the
  test that turns green. I plan to add 1–2 more cases beyond the one given.
- `rag/evaluator/eval_suite.py` (line 43, `EvalSuite.run()`) — the caller
  that currently crashes end-to-end if a `None`-text chunk reaches it. No
  code changes expected here, but I'll test through this path to confirm the
  crash doesn't surface further up.
- `rag/evaluator/relevance_scorer.py` (line 32) — has the *identical*
  `chunk.get("text", "")` pattern, but is **not** part of issue #153's
  scope. Noting it, not touching it (see Risks below).

### Plan
1. Confirm `test_none_context_chunk_text` fails for the documented reason (done — see reproduction commit).
2. Fix the context-building line in `faithfulness_checker.py` so a `None` value is treated the same as a missing key, e.g. `chunk.get("text") or ""`.
3. Re-run `tests/unit/test_faithfulness_checker.py` in full to confirm the target test now passes and nothing else regresses.
4. Add 1–2 new test cases beyond the given repro: a mix of `None` and real-text chunks, and a list where every chunk has `text: None`.
5. Manually exercise `EvalSuite.run()` with a `None`-text chunk to confirm the crash no longer propagates to the caller either.

### Inputs & outputs
**Input:** `context_chunks: list[dict]`, where each dict may or may not have
a `"text"` key, and that key's value may be a string, `None`, or (in theory)
missing entirely.
**Output:** unchanged in shape — a `float` faithfulness score between `0.0`
and `1.0`. After the fix, a `None` text value should not raise; it should
contribute `""` to `context_text`, same as a missing key would.

### Risks & unknowns
- **Unknown:** why does a chunk ever get `text: None` in the first place?
  That's a question about the upstream ingestion/chunking pipeline, not
  these evaluator files. This fix makes the checker resilient to it, but
  doesn't address a possible root cause further upstream — worth a
  follow-up issue if it recurs.
- **Risk:** `chunk.get("text") or ""` also treats an already-present empty
  string `""` the same as `None` (both fall through to `""`). That's
  harmless here since both cases should contribute empty context either way,
  but worth stating explicitly since it's a slightly different condition
  than "key is missing vs. key is `None`."
- **Related but out of scope:** `relevance_scorer.py` line 32 has the same
  pattern and could hit the same crash on `None` text. Not fixing it in this
  PR to keep the change focused on #153, but flagging it in case a shared
  `_get_chunk_text()` helper becomes worth introducing later.
- Need to confirm no other existing test in the suite depends on a `None`
  chunk value actually raising (unlikely, but worth a full-suite run, not
  just the targeted file).

### Edge cases
- Single chunk with `text: None` (the issue's given repro).
- Multiple chunks, a mix of `None` and real strings — confirm only the
  `None` ones contribute empty text, not that the whole context gets dropped.
- Every chunk has `text: None` — `context_text` should end up empty (or
  whitespace-only) rather than crashing; `check()` should degrade to a low
  score, not error out.
- Chunk missing the `"text"` key entirely (the original, already-supported
  case) — must not regress.
- Empty `context_chunks` list — already guarded earlier in `check()`
  (`if not feedback or not context_chunks: return 0.0`); just confirm this
  still holds after the change.