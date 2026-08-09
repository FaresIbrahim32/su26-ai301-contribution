# Contribution #2: Type unordered stepper lists as `Set` instead of casting to/from `list`

**Contribution Number:** 2
**Student:** Fares Ibrahim
**Issue:** https://github.com/ai2cm/ace/issues/579
**Fork:** https://github.com/FaresIbrahim32/ace
**Status:** Phase IV Complete — PR merged into `ai2cm/ace`

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

### Step 1: Contribution Guidelines Reviewed

- **Style:** `ace` lints with `ruff` (`select = ["D", "E", "F", "I", "W", "UP"]`, Google-convention docstrings, line length 88) and type-checks with `mypy` (`--ignore-missing-imports --check-untyped-defs` per `.pre-commit-config.yaml`), both enforced via `pre-commit` and CI (`.github/workflows/pre-commit.yml`, `tests.yaml`).
- **Testing:** `CONTRIBUTING.md` asks bug-fix PRs to include a unit test that would fail without the fix; `tests.yaml` runs the full suite with `pytest` + coverage on Python 3.11 via `uv`.
- **Commit messages:** recent merged history (`git log --oneline`) uses short, imperative, present-tense summary lines with no enforced prefix — I followed that style rather than inventing a Conventional Commits format the project doesn't use.
- **PR description:** `.github/pull_request_template.md` asks for a why/what description, a bulleted list of changed symbols, a "Tests added" checkbox, and `Resolves #<issue>` — I've been drafting Phase III's commit message and this README against that template so Phase IV is just copy-paste.

### Implementation Notes

**What I built:** Retyped the three unordered-name properties in `fme/ace/stepper/single_module.py` from `list[str]` to `set[str]`, and removed the now-redundant `list()`/`set()` casts at their definitions and at every call site, per the Phase II plan.

**Files modified:**
- `fme/ace/stepper/single_module.py` — `StepperConfig.input_only_names` and `.all_names`, and `Stepper._input_only_names`, retyped `-> set[str]`; `get_forcing_window_data_requirements`, `get_evaluation_window_data_requirements`, and `predict_generator` updated to consume them as sets, wrapping in `list(...)` only at the one real boundary that needs it (`DataRequirements.names: list[str]`).
- `fme/coupled/stepper.py` — removed the redundant inner `set(...)` re-cast at all 8 call sites of `input_only_names`/`all_names`; kept the outer `list(...)` wrapper everywhere, since I confirmed (see Challenges below) that every one of `CoupledStepperConfig`'s own forcing-name attributes is either concatenated with `+`, appended to with `.append(...)`, or declared `-> list[str]` on an abstract property elsewhere — i.e. genuinely order/list-shaped per the issue's own caveat, not safe to retype.
- `fme/ace/inference/evaluator.py` — widened `resolve_variable_metadata`'s `stepper_all_names` parameter from `Sequence[str]` to `Collection[str]`.
- `fme/ace/stepper/test_single_module.py` — added `test_stepper_config_input_only_and_all_names_are_sets`.

**Key commits** (on my `579-set-typed-names` branch):
- [`7d2b455`](https://github.com/FaresIbrahim32/ace/commit/7d2b45568) — Add reproduction notes for issue #579 (Phase II)
- [`dedf006`](https://github.com/FaresIbrahim32/ace/commit/dedf00624) — Type unordered stepper names as `set[str]` instead of casting `list<->set`

### Challenges Faced

- **A property I almost broke silently.** `all_names` also feeds `resolve_variable_metadata(..., stepper_all_names: Sequence[str])` in `fme/ace/inference/evaluator.py` (via `fme/ace/inference/inference.py` and a second call site). A `set` is not a `Sequence` — mypy would have caught this as a real type error the moment I retyped `all_names`, but only if I actually type-checked every call site rather than assuming the two I found first were the only ones. I re-grepped the whole repo for `.all_names`/`.input_only_names` (not just the two files from my plan) and found this third consumer. Since the function body only ever does `set(stepper_all_names)` and `for name in stepper_all_names` — never indexing or slicing — I widened the parameter to `Collection[str]` instead of leaving `all_names` as `list[str]` or wrapping it in a `list(...)` cast at the call site (which would've reintroduced exactly the boilerplate the issue is about).
- **Distinguishing which `coupled/stepper.py` attributes were safe to retype.** My Phase II plan flagged this as unresolved ("to be confirmed per-field in Phase III — I'll verify each one rather than assume"). On inspection, `CoupledStepperConfig._all_ocean_names.append(...)` is called later in the same method, and several sibling attributes (`_ocean_forcing_exogenous_names`, etc.) have abstract property declarations elsewhere annotated `-> list[str]` and are concatenated with `+` at other use sites (e.g. `line 429`). Sets don't support `.append()` or `+`, so retyping those would have broken real behavior — this is exactly the "base config values / concatenation-derived values stay list" caveat from the issue, just one level removed from where the issue pointed. I left every `coupled/stepper.py`-owned attribute as `list[str]` and only removed the now-redundant inner `set(...)` re-cast around the two *upstream* properties I did retype.
- **Considered opening a fork-internal PR to trigger real CI** (the approach that worked well for verifying Contribution #1, where a local build wasn't viable either). Opened one, then decided against keeping it open — a PR is a visible action on a shared platform and I didn't want to create fork/CI noise just to get a test run, so I closed it and relied on local `mypy`/`ruff` plus manual per-call-site tracing instead.
- **Couldn't run the real pytest suite locally.** `ace` requires torch, cartopy, torch-harmonics, xarray, and more, installed via `make create_environment` (conda, pinned to Python 3.11). My local environment is Python 3.14, and a from-scratch install of that stack was too heavy/slow to do safely here. This is a real gap, not a solved problem — see Testing Strategy below for exactly what is and isn't verified yet.

### Testing Strategy

**What's actually verified:**
- [x] `mypy --ignore-missing-imports --check-untyped-defs` (the exact flags `.pre-commit-config.yaml` uses) against all three modified source files: zero new errors. (One pre-existing, unrelated error in `fme/core/cli.py` was confirmed present on `origin/main` before my changes too — not something I introduced.)
- [x] `ruff check` and `ruff format --check` (project config from `pyproject.toml`): all checks pass on all four modified files.
- [x] Repo-wide grep for every remaining reference to `.input_only_names` / `.all_names` / `_input_only_names` (not just the files named in my plan) to make sure no consumer was missed — this is what surfaced the `Sequence[str]` issue above.
- [x] Manually traced every non-test consumer of the retyped properties to its actual usage (concatenation, `.append()`, `Iterable`/`Collection`/`Sequence` parameter types) rather than assuming `Iterable`-like usage everywhere.
- [x] Re-ran the Phase II standalone equivalence check (`repro_current_behavior.py` vs `repro_proposed_fix.py`) to confirm the retyped logic is behavior-preserving before touching real source.

**Added:**
- [x] `test_stepper_config_input_only_and_all_names_are_sets` in `fme/ace/stepper/test_single_module.py` — asserts `input_only_names`/`all_names` are `set` (would have failed against the pre-fix code, since they were `list`) with correct membership, and asserts `input_names`/`output_names` are still `list` (guards the issue's caveat from regressing).

### Branch Link

[FaresIbrahim32/ace @ `579-set-typed-names`](https://github.com/FaresIbrahim32/ace/tree/579-set-typed-names)

## Phase IV — Pull Request & Review

**PR Link:** [ai2cm/ace#1391](https://github.com/ai2cm/ace/pull/1391)

**PR Description:** Retyped the genuinely-unordered stepper name properties (`input_only_names`, `all_names`, `_input_only_names`) from `list[str]` to `set[str]` and removed the now-redundant `list()`/`set()` casts at their definitions and 8+ call sites, while leaving order-sensitive base-config values as `list[str]`. Includes a regression test.

Before opening the PR I rebased onto `upstream/main` (34 commits ahead across the touched files, clean rebase, re-verified the diff still applied correctly), reviewed `git diff upstream/main` line by line to confirm no unrelated changes, and confirmed CI (`pre-commit`, `docs`, `tests` cpu/gpu) all passed on the real PR.

### Maintainer Feedback Log

- **2026-07-28 — @mcgibbon (contributor):** "PR looks good and can merge. It doesn't close the issue because I didn't make something clear in writing that issue - the StepABC and its implementations also use many list[str] that should be set[str] since they're unordered. Those will also need to be translated over to fully close it out. Nonetheless, this PR does what it says and it's an improvement. Please remove the 'Resolves' line from the PR description."
  - **How I addressed it:** Edited the PR description to say "Related to #579 (partial...)" instead of "Resolves #579", so merging this PR won't auto-close the issue before the `StepABC` follow-up work is done. Replied on the PR thanking him, confirming the edit, and offering to take on the `StepABC`/implementations follow-up as a separate PR if useful.
  - **Takeaway:** The issue as originally written undersold its own scope — a "good first issue" label doesn't always mean the *literal* text of the issue is the complete fix. Worth surfacing scope gaps like this back to the maintainer rather than silently doing (or silently skipping) more than what was asked.

- **2026-08-05 — @mcgibbon (contributor), approving review:** "I've deleted the dev notes. LGTM! Thank you."
  - **What this meant:** He removed my `dev-notes/issue-579/` directory (the standalone reproduction/equivalence scripts from Phase II) directly on the branch before merging — reasonable, since those were scratch verification files for my own process, not something that belongs in the shipped codebase, and I hadn't flagged that they should be dropped before merge myself.
  - **Takeaway:** Next time, I'll clean up my own dev-only scratch files out of the PR diff before requesting review, rather than leaving that cleanup for the maintainer to notice and do.

**Merged:** [`1c62fd1`](https://github.com/ai2cm/ace/commit/1c62fd1325cf437099689e0e5a359bfb01cf804a) on 2026-08-05, by @mcgibbon.

**Status:** Merged — PR #1391 is closed and merged into `ai2cm/ace:main`. Follow-up scope (retyping the equivalent unordered `list[str]` values in `StepABC` and its implementations, per @mcgibbon's first comment) remains open; I offered to pick it up as a separate PR and am waiting to hear if he wants that.
