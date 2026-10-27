# OSS_REPORT: pydantic/pydantic-ai#11437

**Status:** DONE (issue was open, unassigned, no comments, no linked PRs when checked; read via the public issue page).

**FLAG: SMALL FIX.** Prompt wording plus one opt-in field in a single capability; the issue is labelled `feature`, and the budget field is new public API, so maintainers should confirm the shape.

## Root cause
1. `_DEFAULT_SUMMARY_PROMPT` gave no length target, so the model chose the size of the summary.
2. The summary `Agent` ran with only the user's `model_settings`, so nothing capped output tokens.
3. The incremental instruction said "merge in new facts" with no size constraint, so each re-compaction could grow the previous summary.

## Diff summary
- `compaction/_summarizing_compaction.py`
  - Default prompt now says: be concise, substantially shorter than the conversation it replaces.
  - Incremental instruction now says: rewrite the summary to stay as short as it was, condensing older details rather than appending.
  - New kw-only field `max_summary_tokens: int | None = None`. When set, the prompt ends with "Keep the summary under N tokens." and `max_tokens=N` is set on the summary call (overrides `model_settings["max_tokens"]`; the user's dict is copied, not mutated).
- Docs: `compaction/README.md` and `docs/harness/compaction.md` explain the budget/retention trade-off.
- Tests: two new tests in `tests/harness/compaction/test_compaction.py` (FunctionModel sees the prompt line and `max_tokens`; default call is uncapped).

## Design choice
- **Default behaviour change is wording only.** No default cap is introduced, because a hard cap truncates summaries mid-section and would change results for every existing user. Users who want a bound opt in with `max_summary_tokens`.
- **Rejected: a fraction-of-input budget.** It needs token estimation of the transcript and adds two knobs; the absolute cap covers the acceptance criteria. This could be a follow-up if maintainers want it.
- **Cap equals the prompt's stated budget**, so an overshooting model is cut off. This is documented.
- The default prompt is byte-different, so cassette-style or snapshot users who pin the exact prompt would see a change. No such test failed in the repo.

## Tests
```
uv run --all-packages pytest tests/harness/compaction -q
  824 passed, 4 skipped
uv run ruff check src tests/harness/compaction   -> All checks passed
uv run ruff format src tests/harness             -> unchanged
pyright _summarizing_compaction.py               -> 0 errors
```
`tests/test_examples.py -k compaction`: 1 failure, `docs/models/typesafe.md` needs the uninstalled `typesafe-sdk` package. It is unrelated to this change.

## Risks
- A prompt wording change shifts summary length and content for all users of the default prompt (intended, but unmeasured). I ran no quality eval.
- `max_summary_tokens` truncates at the cap; a too-small value yields a cut-off summary.
- It is new public API (`max_summary_tokens`); the name and the override of `model_settings` need maintainer sign-off.

## Draft PR description
**Title:** Add a length instruction and `max_summary_tokens` to `SummarizingCompaction`

Fixes #11437

`SummarizingCompaction`'s default summary prompt had no length target, the summary call had no output cap, and incremental updates were told to merge new facts into the previous summary without a size limit, so summaries could grow on each compaction.

- The default prompt now asks for a concise summary, substantially shorter than the transcript it replaces.
- The incremental instruction asks the model to rewrite the previous summary to its previous length instead of appending.
- New opt-in `max_summary_tokens` states a budget in the prompt and sets `max_tokens` on the summary call (taking precedence over `model_settings`).
- Docs cover the budget/retention trade-off, and tests verify the budget reaches the request via `FunctionModel`.

## Estimated cost
About $1-2 of usage.
