# Contribution #2: Type unordered stepper lists as `Set` instead of casting to/from `list`

**Contribution Number:** 2
**Student:** Fares Ibrahim
**Issue:** https://github.com/ai2cm/ace/issues/579
**Fork:** https://github.com/FaresIbrahim32/ace
**Status:** Phase I Complete — Issue selected, repo forked, intro comment posted

---

## Why I Chose This Issue

I picked this issue because it's a small, self-contained type-hygiene fix in a real Python research codebase (Ai2's climate emulator, `ace`), which lines up well with the Python experience I already have. The task is narrow and clearly scoped: some values in the stepper code are genuinely unordered collections, but they're typed and passed around as `List[str]`, forcing the code to cast to `set` and back to `list` just to satisfy the type checker. Retyping those specific values as `Set[str]` removes the boilerplate casting and makes the code's intent (this collection has no meaningful order) explicit at the type level. It's labeled "good first issue," has no assignees, no existing linked PR, and only needs careful reading of the stepper module plus attention to one important nuance called out in the issue itself: config values and anything derived from them by concatenation must stay as `list`, since ML channel order matters there. That nuance is exactly the kind of "read carefully, don't over-apply the change" detail I want practice with, and `ace`'s `CONTRIBUTING.md` explicitly calls out type-hinting consistency as a code review criterion, so this fix maps directly onto something the maintainers already care about.

## Problem Summary

In the stepper code, several values that are truly unordered collections are still typed and threaded through as `List[str]`. Because the type constraint says `List[str]`, the code repeatedly casts these values `list -> set -> list` (or vice versa) just to do set-like operations (e.g., membership checks, unions) before casting back to satisfy the declared type. This is unnecessary boilerplate and also slightly misleading — the `List[str]` type implies an ordering guarantee that doesn't actually exist or matter for these values. The fix is to change the return/parameter types of the genuinely-unordered values to `Set[str]` so the cast dance goes away and the type signature honestly reflects that order is not preserved or meaningful. The one thing to be careful about: base config values, and any values derived from them via concatenation, must remain `List` type, because for those specific values the order corresponds to ML channel order and changing them to `Set` would be a correctness bug, not a cleanup.

"Fixed" looks like: the identified unordered `List[str]` values in the stepper code are retyped as `Set[str]`, the now-redundant `list(set(...))` / `set(...)` casts around them are removed, type checking (mypy or equivalent) still passes, and any list-typed values that preserve real ordering (config values, concatenation-derived channel lists) are left untouched as `List`.

---

## Phase II — Understanding the Issue & Solution Approach

_To be completed in Phase II: identify every unordered `List[str]` site in the stepper code, confirm which lists are safe to retype vs. which must stay ordered, reproduce/observe the current casting boilerplate, and propose the specific type changes._

## Phase III — Testing Strategy & Implementation Notes

_To be completed in Phase III: implement the type changes, remove the redundant casts, and verify via the existing type-checker/test suite that nothing relying on list ordering broke._

## Phase IV — Pull Request & Review

_To be completed in Phase IV: open the PR against `ai2cm/ace`, link it here, and log maintainer feedback._
