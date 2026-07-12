# Contribution #2: Type unordered stepper lists as `Set` instead of casting to/from `list`

**Contribution Number:** 2
**Student:** Fares Ibrahim
**Issue:** https://github.com/ai2cm/ace/issues/579
**Fork:** https://github.com/FaresIbrahim32/ace
**Status:** Phase II Complete — Reproduced the issue and wrote the solution plan

---

## Why I Chose This Issue

I picked this issue because it's a small, self-contained type-hygiene fix in a real Python research codebase (Ai2's climate emulator, `ace`), which lines up well with the Python experience I already have. The task is narrow and clearly scoped: some values in the stepper code are genuinely unordered collections, but they're typed and passed around as `List[str]`, forcing the code to cast to `set` and back to `list` just to satisfy the type checker. Retyping those specific values as `Set[str]` removes the boilerplate casting and makes the code's intent (this collection has no meaningful order) explicit at the type level. It's labeled "good first issue," has no assignees, no existing linked PR, and only needs careful reading of the stepper module plus attention to one important nuance called out in the issue itself: config values and anything derived from them by concatenation must stay as `list`, since ML channel order matters there. That nuance is exactly the kind of "read carefully, don't over-apply the change" detail I want practice with, and `ace`'s `CONTRIBUTING.md` explicitly calls out type-hinting consistency as a code review criterion, so this fix maps directly onto something the maintainers already care about.

## Problem Summary

In the stepper code, several values that are truly unordered collections are still typed and threaded through as `List[str]`. Because the type constraint says `List[str]`, the code repeatedly casts these values `list -> set -> list` (or vice versa) just to do set-like operations (e.g., membership checks, unions) before casting back to satisfy the declared type. This is unnecessary boilerplate and also slightly misleading — the `List[str]` type implies an ordering guarantee that doesn't actually exist or matter for these values. The fix is to change the return/parameter types of the genuinely-unordered values to `Set[str]` so the cast dance goes away and the type signature honestly reflects that order is not preserved or meaningful. The one thing to be careful about: base config values, and any values derived from them via concatenation, must remain `List` type, because for those specific values the order corresponds to ML channel order and changing them to `Set` would be a correctness bug, not a cleanup.

"Fixed" looks like: the identified unordered `List[str]` values in the stepper code are retyped as `Set[str]`, the now-redundant `list(set(...))` / `set(...)` casts around them are removed, type checking (mypy or equivalent) still passes, and any list-typed values that preserve real ordering (config values, concatenation-derived channel lists) are left untouched as `List`.

---

## Phase II — Understanding the Issue & Solution Approach

### Reproduction Process

#### Environment Setup

This issue is a type-hygiene / code-quality issue, not a runtime crash, so there's no error message or stack trace to trigger. "Reproducing" it means locating every place the codebase pays the `list -> set -> list` casting tax and confirming, with the actual type checker, that the tax is real (not just a naming quibble) and that removing it is type-safe.

`ace` pulls in a heavy dependency stack (torch, xarray, dacite, etc. via `make create_environment` / conda), which isn't necessary just to inspect and type-check these specific properties. Instead of installing the full environment, I:

1. Forked and cloned the repo (`https://github.com/FaresIbrahim32/ace`) and read the real source directly.
2. Created a Python 3.14 virtualenv (`python3 -m venv repro-venv`) and installed only `mypy` into it — no ace dependencies needed, since the pattern I'm isolating is pure standard-library `list`/`set` logic.
3. Extracted the exact property definitions (same names, same annotations, same logic) into a small standalone script so I could run `mypy --strict` and the interpreter directly against them, without fighting missing-import errors from torch/xarray.

The one snag: `pip install mypy` refused to run against the system Python (PEP 668 "externally managed environment" error). Fixed by creating an isolated venv first (`python3 -m venv repro-venv && ./repro-venv/bin/pip install mypy`) rather than passing `--break-system-packages`.

#### Steps to Reproduce

1. Clone the fork and grep the stepper code for `set(` outside of tests:
   `grep -rn "set(" fme/ace/stepper/*.py | grep -v test`
2. This surfaces the exact sites named in the issue, in `fme/ace/stepper/single_module.py`:
   - `input_only_names` ([single_module.py:561-562](https://github.com/FaresIbrahim32/ace/blob/main/fme/ace/stepper/single_module.py#L561-L562)): `return list(set(self.input_names) - set(self.output_names))`, annotated `-> list[str]`.
   - `all_names` ([single_module.py:693-695](https://github.com/FaresIbrahim32/ace/blob/main/fme/ace/stepper/single_module.py#L693-L695)): `return list(set(self.input_names + self.output_names))`, annotated `-> list[str]`.
   - `get_forcing_window_data_requirements` ([single_module.py:568-570](https://github.com/FaresIbrahim32/ace/blob/main/fme/ace/stepper/single_module.py#L568-L570)) immediately re-wraps `input_only_names` in `set(...).union(...)` before casting back to `list(...)`.
   - A second, near-identical private property `_input_only_names` ([single_module.py:1110-1112](https://github.com/FaresIbrahim32/ace/blob/main/fme/ace/stepper/single_module.py#L1110-L1112)), whose result is immediately re-cast into a `set` again two lines later in `predict_generator` ([single_module.py:1187-1189](https://github.com/FaresIbrahim32/ace/blob/main/fme/ace/stepper/single_module.py#L1187-L1189)).
3. Grep the downstream caller, `fme/coupled/stepper.py`, for the same properties:
   `grep -n "input_only_names\|all_names" fme/coupled/stepper.py`
   This shows **8 separate call sites** ([stepper.py:297, 302, 323, 336, 343, 358, 481, 495, 524](https://github.com/FaresIbrahim32/ace/blob/main/fme/coupled/stepper.py#L296-L524)) that each take the `list[str]` returned by `input_only_names`/`all_names` and immediately re-wrap it in `set(...)` to run `.difference(...)` / `.intersection(...)` / `.union(...)`, then wrap the result back in `list(...)` again to store it. This is the literal round-trip the issue describes, and it's consistent — every single call site does it, not a one-off.
4. To confirm the cast is genuinely unnecessary (not silently relied upon for order), I isolated the three properties above into a standalone script (`repro_current_behavior.py`, committed on my working branch) mirroring the real annotations, and ran it twice — once through `mypy --strict` and once through the interpreter:
   - `mypy --strict repro_current_behavior.py` → `Success: no issues found in 1 source file` (confirms the current `list[str]` + cast version is what's actually type-checking clean today — there's no existing type error, just needless ceremony).
   - Running the script prints `input_only_names`, `all_names`, and a simulated coupled-stepper forcing-name computation; re-running the same property call 5 times in the same process, and running the whole script fresh 3 separate times, produces the same output each time — the values are correct and stable, they just have no *guaranteed* order (the `list[str]` annotation implies an ordering contract that the implementation never actually provides).
5. I then wrote the proposed fix in miniature (`repro_proposed_fix.py`): the same three properties retyped as `set[str]`, with the redundant `list(...)`/`set(...)` casts removed entirely. `mypy --strict repro_proposed_fix.py` also returns `Success: no issues found in 1 source file`, and the runtime output is equivalent (as a `set` instead of a `list`) — confirming the retype is type-safe and behavior-preserving for these specific properties.
6. Checked for order-sensitive regression risk in existing tests: `grep -rn "input_only_names|\.all_names\b" fme/coupled/test_stepper.py` shows the existing tests only ever compare these values via `set(...) == set(...)` or `"x" in names` membership checks ([test_stepper.py:403, 918, 1042](https://github.com/FaresIbrahim32/ace/blob/main/fme/coupled/test_stepper.py#L403)) — never index into them or compare them positionally. No test currently depends on these properties preserving list order, which lowers the risk of the retype.
7. Confirmed the important **caveat from the issue itself still holds**: `input_names`, `output_names`, and `prognostic_names` (e.g. [single_module.py:688-712](https://github.com/FaresIbrahim32/ace/blob/main/fme/ace/stepper/single_module.py#L688-L712), and their definitions in `fme/core/step/*.py`) come straight from base config and represent real ML channel order — these must stay `list[str]` and are explicitly out of scope for this fix.

I ran steps 4-5 twice on different days to confirm the mypy result and output were reproducible, not a fluke.

#### Branch Link

Working branch (reproduction scripts + notes, no source changes yet): [FaresIbrahim32/ace @ `579-set-typed-names`](https://github.com/FaresIbrahim32/ace/tree/579-set-typed-names)

### Solution Approach

#### Implementation Plan (UMPIRE)

**Understand:** Several stepper properties compute genuinely unordered collections (set differences/unions of variable names) but are typed and returned as `list[str]`, forcing `list(set(...))` casts at the definition site and matching `set(...)` re-casts at every call site. This is pure boilerplate with no functional purpose — nothing downstream relies on the order. The fix is to retype these specific properties as `set[str]` and delete the now-redundant casts, while leaving genuinely order-sensitive values (base config lists, and lists derived from them by concatenation, e.g. `input_names`, `output_names`, `prognostic_names`) untouched as `list[str]`, per the issue's own caveat.

**Match:** `fme/core/step/step.py` and its subclasses already define `input_names`/`output_names`/`prognostic_names`/`next_step_input_names` consistently as `list[str]` properties feeding from config — that's the existing pattern for "order matters" values, and it's the pattern I will *not* touch. The pattern I *will* follow is simpler: Python's own `set` type already supports `.difference()`, `.union()`, `.intersection()` directly, so removing the cast is a matter of changing the annotation and deleting two wrapper calls per site, not introducing new abstractions.

**Plan:**
1. In `fme/ace/stepper/single_module.py`, retype `input_only_names` (line 561) and `all_names` (line 693) from `-> list[str]` to `-> set[str]`, and simplify their bodies to plain `set` operations (no `list(...)` wrapper).
2. Retype the duplicate private `_input_only_names` (line 1110) the same way.
3. Update `get_forcing_window_data_requirements` (line 568-570) and `predict_generator` (line 1187-1189) to consume these as sets directly, only converting to `list(...)` at the one remaining boundary that genuinely requires a list — `DataRequirements.names: list[str]` (`fme/ace/requirements.py:33`) — since that dataclass field's contract is unchanged by this issue.
4. In `fme/coupled/stepper.py`, update the 8 call sites (lines 297-524) that currently do `list(set(self.X.stepper.input_only_names).difference(...))` etc. to drop the outer `set(...)` re-cast, since the input is already a `set`; keep the outer `list(...)` only where the *result* is stored in a field that's actually order-sensitive downstream (to be confirmed per-field in Phase III — most of these, e.g. `_ocean_forcing_exogenous_names`, appear to be consumed only via further set/membership logic, but I'll verify each one rather than assume).
5. Leave `input_names`, `output_names`, `prognostic_names`, `next_step_input_names`, and `DataRequirements.names` as `list[str]` — these are base config values or built by concatenation and their order is real ML channel order (per the issue's explicit caveat).
6. Run the full stepper + coupled-stepper test suites and `mypy` locally after each file's change, rather than batching all edits, to catch a type error or broken assumption immediately rather than at the end.

**Implement:** (Phase III — branch is up at [`579-set-typed-names`](https://github.com/FaresIbrahim32/ace/tree/579-set-typed-names); no source edits yet, only the reproduction scripts under `dev-notes/issue-579/`.)

**Review:** Before opening a PR I'll re-read `ace`'s `CONTRIBUTING.md`, which asks that PRs (a) stay focused — I'll keep this PR to the type-hygiene change only, not a broader stepper refactor; (b) include a unit test for bug fixes — since this is a type/cleanup change rather than a behavioral bug fix, I'll instead make sure existing tests still pass and add a small regression test asserting the retyped properties are still functionally equivalent to their pre-change values; (c) follow Conventional Comments if a maintainer reviews with that convention; (d) keep type-hinting consistent, which is the explicit motivation for this issue.

**Evaluate:** `ace` runs `mypy` (configured via `[[tool.mypy.overrides]]` in `pyproject.toml`) and pytest as part of its CI. My verification plan: run `mypy` over the touched files and confirm no new errors; run the existing `fme/ace/stepper/test_single_module.py` and `fme/coupled/test_stepper.py` suites and confirm they still pass unmodified (per my Phase II check, none of them assert list order for these properties); and re-run my standalone `repro_current_behavior.py` / `repro_proposed_fix.py` comparison to double check the retype is behavior-preserving before touching the real source files.

## Phase III — Testing Strategy & Implementation Notes

_To be completed in Phase III: implement the type changes, remove the redundant casts, and verify via the existing type-checker/test suite that nothing relying on list ordering broke._

## Phase IV — Pull Request & Review

_To be completed in Phase IV: open the PR against `ai2cm/ace`, link it here, and log maintainer feedback._
