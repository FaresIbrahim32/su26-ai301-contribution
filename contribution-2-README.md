# Contribution #2: Type unordered stepper names as `set[str]`

**Contribution Number:** 2
**Student:** Fares Ibrahim
**Issue:** https://github.com/ai2cm/ace/issues/579
**Fork:** https://github.com/FaresIbrahim32/ace
**Status:** Phase III complete — fix implemented and locally verified

---

## Summary

This contribution fixes a type-hygiene issue in the ACE stepper code. Several properties that represent unordered name collections were annotated and used as lists, which forced unnecessary `list`/`set`/`list` conversions. I updated those properties to use `set[str]` and removed the redundant casts while leaving order-sensitive list values unchanged.

## What changed

- Updated the unordered stepper properties in the ACE stepper code to return `set[str]` instead of `list[str]`.
- Removed redundant casting at the definition sites and downstream call sites.
- Kept list-based values that preserve real ordering unchanged.
- Added a regression test covering the new typing behavior.

## Why this matters

The affected values are inherently unordered, so typing them as lists was misleading and introduced unnecessary boilerplate. Using `set[str]` makes the intent clearer and keeps the code consistent with how the values are actually used.

## Verification

- Ran the project’s mypy checks with the configured settings.
- Ran ruff checks and formatting checks.
- Reviewed the affected call sites to ensure the change stayed scoped and did not alter behavior for ordered list values.

## Status

- PR: pending
- Maintainer feedback: pending
