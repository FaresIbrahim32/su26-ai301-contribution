# Contribution #1: Migrate Makefile `install` target into CMake

**Contribution Number:** 1
**Student:** Fares Ibrahim
**Issue:** https://github.com/ponylang/ponyc/issues/3898
**Status:** Phase 2 — fix implemented, pending full build verification

---

## Why I Chose This Issue

This issue interests me because it sits at the intersection of build systems and toolchain reliability—making sure a configured build setting actually survives all the way through to installation is exactly the kind of correctness problem I want to get better at. I have C/C++ and Python experience from my research (implementing algorithms, building Flask/Django apps), and I've worked with Make and CMake on systems-level projects, so the Makefile-to-CMake migration here builds directly on what I already know. I'm hoping to learn more about how a mature compiler project like Pony structures its build and install pipeline, and how to keep configuration parameters consistent across `configure`, `build`, and `install`. It's labeled "good first issue" and "help wanted," which makes it a great first contribution to ponyc.

---

## Understanding the Issue

### Problem Description

The `install` target in the Unix Makefile is not integrated with CMake, so it ignores the `arch` parameter that was passed during the `configure` step. When a user runs `make configure arch=not-our-default`, then `make build`, then `make install`, the install step falls back to the default architecture instead of the one the user configured. The result is a broken `ponyc` binary that cannot locate its library dependencies.

### Expected Behavior

Architecture (and other configuration) settings should persist from `configure` through `build` and into `install`. Running `make configure arch=<something>` followed by `make build` and `make install` should produce a working `ponyc` installation built for the configured architecture—without the user having to re-specify `arch` at install time.

### Current Behavior

`make install` uses the default architecture regardless of what was configured. The current workaround is to pass the `arch` parameter again during the install step. Without that workaround, the installed binary is broken and can't find its libraries.

### Affected Components

- `Makefile` — the `install` target; all of its current functionality needs to be migrated into CMake so configuration state is preserved
- `CMakeLists.txt` — the install logic should live here so that values set during `configure` (e.g. `arch`) are carried through to `install`

---

## Reproduction Process

### Environment Setup

I worked from a fork of ponyc at version `0.64.0` (commit `ab91e5cf`). The important realization during setup was that I did **not** need a full compile to reproduce this bug. A real ponyc build pulls in LLVM and takes hours, which made iterating painful. But #3898 is not a code-path bug—it's a **build-variable drift** bug—so it can be demonstrated without compiling anything. That insight saved a huge amount of time and is the core of my reproduction strategy.

### Root Cause: two stores for one value, never synced

The architecture lives in **two separate places that don't stay in sync**:

| Step | How it reads `arch` | Where the value lives |
|------|--------------------|------------------------|
| `make configure arch=X` | make variable → **writes** `-DPONY_ARCH=X` | CMake cache (`CMakeCache.txt`) — persistent |
| `make build` | reads the CMake cache | CMake cache — **correct** |
| `make install` | re-reads make's `$(arch)` | make variable — **re-defaults to `native`** |

`Makefile:2` declares `arch ?= native`. Because make has no memory between invocations, that default silently reapplies on every command unless you re-type `arch=`. `configure` (Makefile:226) saves the real arch into CMake's cache, and `build` (Makefile:231) reads it back—so the build is correct. But the old `install` target was plain shell driven by make and rebuilt its destination from `$(arch)` (Makefile:447, `$(ponydir)/lib/$(arch)`), so it pointed at `lib/native` while the binary expected `lib/<configured-arch>`.

### Steps to Reproduce

1. Simulate `make configure arch=skylake` by writing `PONY_ARCH:STRING=skylake` into a stub `CMakeCache.txt` (this is exactly what configure persists).
2. Simulate `make build` by placing a stub `libponyrt.a` in the build tree.
3. Run the **real** `make install` recipe with **no** `arch=`, staging into a throwaway `DESTDIR` so nothing touches `/usr/local`.
4. Observe where the library lands.

### Reproduction Evidence

A self-contained, re-runnable script (`repro-3898.sh`) drives the steps above. Output:

```
### configure persisted (CMake's source of truth):
PONY_ARCH:STRING=skylake

### running REAL 'make install' with NO arch= (DESTDIR -> sandbox):

### libponyrt.a actually landed at:
<SANDBOX>/lib/native/libponyrt.a

### lib/ subdirectory install created:
native

>>> BUG REPRODUCED: artifacts built for 'skylake' installed into lib/native/.
>>> ponyc will look in lib/skylake/ and find nothing — broken install.
```

The drift is also visible with a pure dry run, no stubs at all:

```
$ make -n install arch=skylake | grep lib/   # workaround: -> lib/skylake
$ make -n install              | grep lib/   # bug:        -> lib/native (always)
```

- **My findings:** The bug is the *relationship* between two correct-looking lines, not a single broken line. `build` reads the cache (so it "just works" without re-passing `arch`), which lulls you into expecting `install` to behave the same—but `install` reaches for the make variable instead. This is what makes the bug sneaky.

---

## Solution Approach

### Analysis

There is no single buggy line; the defect is that `install` uses a *different source of truth* than `build`. `build` reads the CMake cache; `install` re-reads make's ephemeral `$(arch)`. The fix is to collapse the two stores into one by letting **CMake own the install step**, so install reads the same cached `PONY_ARCH` the build used.

### Proposed Solution

1. Replace the shell `cp` logic in the Makefile `install` target with a thin wrapper: `cmake --install $(buildDir)`. Because `cmake --install` consumes the configured cache, the arch can no longer drift between `configure` and `install`.
2. Add the matching `install()` rules to `CMakeLists.txt` so `cmake --install` actually installs everything the old shell logic did (libraries, headers, packages, tools), placing the runtime into `lib/${PONY_ARCH}`.
3. Make the Makefile's optional `symlink=yes` block read the cached `PONY_ARCH` (not make's `$(arch)`) so its symlinks point at the same per-arch directory CMake installed into.

### Implementation Plan (UMPIRE)

**Understand:** `make install` ignores the architecture chosen during `make configure` because install is shell/make logic, not CMake logic, and the two systems don't share state. Make CMake own install so the cached `PONY_ARCH` is always used.

**Match:** Reuse CMake's native install machinery already present elsewhere in the project—`install(TARGETS …)` for real targets (`ponyc`, `libponyc`, `libponyrt`), `install(FILES …)` for headers and generated artifacts, `install(DIRECTORY …)` for `packages/`. `CMAKE_INSTALL_PREFIX`/`--prefix` replaces make's `prefix`/`ponydir`, and `DESTDIR` staging is supported natively.

**Plan:**
1. Inventory everything the old `install` target copied: static libs → `lib/`, binaries → `bin/`, headers → `include/`, `packages/`, plus the optional `symlink=yes` block.
2. Replace the Makefile `install` body with a `cmake --install` wrapper.
3. Translate each copied group into `install()` rules in `CMakeLists.txt`.
4. Use `OPTIONAL` for artifacts that only exist in certain configs (pic libs, crt objects, dtrace).
5. Install into `lib/${PONY_ARCH}` and make the symlink block read the same cached `PONY_ARCH`.

**Implement:** Changes made on the ponyc fork:

- **`Makefile`** — `install` target rewritten from ~20 lines of shell `cp` into:
  ```makefile
  install: all
      $(SILENT)cd '$(buildDir)' && env CC="$(CC)" CXX="$(CXX)" cmake --install '$(buildDir)' --prefix $(ponydir) --config $(config)
  ```
  The `symlink=yes` block was updated to read the configured arch from the CMake cache (`grep '^PONY_ARCH:' CMakeCache.txt`) so its symlink sources match the per-arch directory CMake installs into — instead of make's drifting `$(arch)`.

- **`CMakeLists.txt`** — install rules completed:
  - Added `libponyc` to the lib install (was only `libponyrt`), with destination **`lib/${PONY_ARCH}`**.
  - Added `install(FILES … OPTIONAL)` for the generated static libs and crt objects (`libponyc-standalone.a`, `libponyrt-pic.a`, `libdtrace_probes.a`, `crtbeginS.o`, `crtendS.o`, `crtbeginT.o`, `crtend.o`), also into `lib/${PONY_ARCH}`.
  - Added header installs for `pony.h` and `pony/detail/atomics.h`.
  - **Removed** the test/benchmark binaries (`libponyc.tests`, `libponyc.benchmarks`, `libponyrt.benchmarks`, `libponyrt.tests`) from the `bin` install — the old shell install never shipped these, so installing them was a regression.

**Review (self-checklist):**
- [x] No remaining shell `cp` install logic in the Makefile.
- [x] `arch` is read only from the CMake cache, never re-defaulted at install time.
- [x] Optional artifacts use `OPTIONAL` so they don't break install when absent.
- [x] Makefile `install` target name and recipe indentation (hard TAB) are valid — `make -n install` parses and resolves.
- [x] Symlink layout matches CMake install destinations.
- [x] Release-notes fragment added (`.release-notes/fix-install-honors-configured-arch.md`).

**Evaluate:** Re-run the reproduction scenario and confirm libs land in the configured location; on a real build box, run `make libs && make configure arch=<non-default> && make build && make install` and confirm the installed `ponyc` finds its libraries without re-passing `arch`.

---

## Testing Strategy

### Reproduction / Regression Test

- [x] `repro-3898.sh` — stubs the configured cache + artifacts and runs the real `install` recipe into a sandbox. Exits `0` when the bug reproduces against the **old** Makefile, and is intended to flip to "did NOT reproduce" against the fixed tree. This doubles as a lightweight regression guard.

### Static / Structural Verification (done locally, no LLVM)

- [x] `make -n install` parses and resolves to the `cmake --install` path (caught and fixed an invalid target name + space-vs-tab indentation along the way).
- [x] `grep` confirms no `lib/$(arch)` drift remains in the Makefile.
- [x] `CMakeLists.txt` install block is paren-balanced and every Makefile symlink target maps to a real CMake install destination.

### Ensemble code review (pony-code-review)

Ran ponylang's `pony-code-review` skill (8-persona full mode) against the branch. It surfaced — and I then fixed — two real regressions plus several smaller issues:

- **Critical (Windows):** installing the runtime into `lib/${PONY_ARCH}` broke Windows, where the compiler searches a flat `..\lib` (`add_exec_dir()` in `src/libponyc/pkg/package.c` has no arch component on Windows) and `make.ps1` never sets `PONY_ARCH`. Fixed by gating the destination on platform (flat `lib` on Windows, `lib/${PONY_ARCH}` elsewhere).
- **High (DESTDIR):** the Makefile already folds `DESTDIR` into `--prefix`, but `cmake --install` independently honors `$DESTDIR`, double-staging into `$DESTDIR/$DESTDIR`. Fixed by clearing `DESTDIR` in the cmake environment.
- **Medium/Low:** fail loudly if `PONY_ARCH` can't be read from the cache; quote the symlink shell expansions; mark the tool `install(PROGRAMS)` rules `OPTIONAL`; add this release-notes fragment.
- **Rejected one finding with reasoning:** a reviewer suggested the symlink block use make's `$(arch)` instead of reading the cache. Rejected — `$(arch)` re-defaulting to `native` at install time *is* the original #3898 bug; the cache holds the value the build actually used.

### Full Build Verification — NOT DONE (disclosed)

I did **not** build and run the install end-to-end. Two honest reasons:

1. A local build requires `make libs`, which compiles LLVM from scratch (multiple hours); I did not have a build box to commit to that.
2. **Regular CI would not verify it either** — no PR workflow runs `make install`; only the release/nightly scripts do, and they don't assert artifact placement. So the install path is exercised by neither my machine nor PR CI.

Outstanding, to be verified by a maintainer or a dedicated build before merge:
- [ ] `make libs && make configure arch=<non-default> && make build && make install` → libraries land in `lib/<arch>` and the installed `ponyc` links a sample program.
- [ ] `symlink=yes`, `DESTDIR` staging, and `uninstall` paths.

I will state this verification status plainly in the PR rather than imply the change is build-tested.

---

## Implementation Notes

### Phase 1 Progress — Reproduction

Confirmed the bug live against `0.64.0 @ ab91e5cf` without a full compile, by recognizing it as build-variable drift rather than a runtime code path. Built `repro-3898.sh` as repeatable proof and documented the two-stores-one-value root cause.

### Phase 2 Progress — Fix

Rewrote the Makefile `install` target as a `cmake --install` wrapper and completed the CMake `install()` rules so the wrapper actually installs the full set of artifacts. Caught and corrected an early mistake where the target was accidentally named `cmake --install` (make read everything before the `:` as the target name) and the recipe was indented with spaces instead of a TAB—both broke parsing. The bigger correction came when I checked how the compiler locates its runtime: I had initially flattened the install to `lib/`, but `add_exec_dir()` in `src/libponyc/pkg/package.c` shows the installed `ponyc` looks for `../lib/<arch>` using the compiled-in `PONY_ARCH`. A flat layout would have broken the installed compiler. I corrected the install destination to `lib/${PONY_ARCH}` and made the symlink block read the cached `PONY_ARCH`, so the install location and the compiler's lookup path share one source of truth.

### Code Changes

- **Files modified:** `Makefile` (install target + symlink block), `CMakeLists.txt` (install rules)
- **Approach decisions:**
  - Install the runtime into **`lib/${PONY_ARCH}`**, using `PONY_ARCH` as the single source of truth. This is the *same* cached value the compiler bakes in and uses at link time to locate its runtime (`add_exec_dir()` builds `../lib/<arch>` from the compiled-in `PONY_ARCH` — see `src/libponyc/pkg/package.c`). Tying the install destination to that one variable is what actually closes the drift.
  - The symlink block reads `PONY_ARCH` from `CMakeCache.txt` rather than make's `$(arch)`, so every part of install — `cmake --install`, the symlinks, and the compiler's own lookup — agrees on the arch.
  - Used `install(TARGETS …)` for real targets so CMake resolves their archive paths regardless of output-directory settings, and `install(FILES … OPTIONAL)` for generated artifacts whose existence is platform/config dependent.
  - Dropped `--strip` from `cmake --install`: the original install never stripped, and stripping would damage the static archives and crt object files (it removes symbols the linker needs).

- **Course-correction worth recording:** my first attempt flattened the install to `lib/` (no arch subdirectory), reasoning that per-arch directories *were* the bug. That was wrong. Reading `add_exec_dir()` showed the installed compiler hard-looks-for `../lib/<arch>`, so a flat layout would have left it unable to find its runtime — recreating #3898 from the opposite side. The bug was never the per-arch directory; it was install reading a *different* arch than the build. The fix is one shared `PONY_ARCH`, not removing the directory.

### Known Caveats / Follow-ups

- **DTrace on Linux:** the old Makefile copied `libdtrace_probes.a` from the build tree when present; the CMake rules only stage it into the output dir on FreeBSD. A Linux+DTrace build would miss it — flagged as a follow-up for byte-for-byte parity.
- **`examples/`** is installed by a pre-existing CMake rule even though the old shell install didn't ship it; left untouched as out of scope.
- **Full build not yet run** — the fix is statically validated and the install/symlink layouts agree, but end-to-end proof needs an LLVM build box.

---

## Pull Request

**PR Link:** _Not yet submitted — pending full build verification and a changelog fragment._

**PR Description (draft):** Migrate the Unix Makefile `install` target into CMake so the architecture configured at `make configure` time is preserved through `make install` (fixes #3898). The Makefile `install` target becomes a thin `cmake --install` wrapper, and the corresponding `install()` rules are added to `CMakeLists.txt`.

**Status:** Iterating locally.

---

## Learnings & Reflections

### Technical Skills Gained

- How make variables (ephemeral, per-invocation) differ from the CMake cache (persistent), and why mixing the two creates "works on build, breaks on install" bugs.
- How to reproduce a build-system bug **without** a multi-hour compile by isolating the actual failing mechanism (variable drift) and stubbing the rest.
- Translating shell-based install logic into CMake `install(TARGETS/FILES/DIRECTORY …)` rules, including `OPTIONAL` for config-dependent artifacts.

### Challenges Overcome

- Diagnosing that the bug is a *relationship between two correct-looking lines*, not one broken line.
- Two self-inflicted Makefile mistakes (invalid target name, space indentation) that broke parsing — fixed and now guarded by `make -n`.
- A layout mismatch between the CMake install destinations and the Makefile symlink block.

### What I'd Do Differently Next Time

- Validate Makefile edits with `make -n` immediately after each change, rather than batching edits — it would have caught the tab/target-name errors instantly.
- Set up an LLVM build box earlier so functional verification isn't deferred to the end.

---

## Resources Used

- ponyc issue #3898: https://github.com/ponylang/ponyc/issues/3898
- ponyc `BUILD.md` and `CONTRIBUTING.md`
- CMake `install()` documentation (TARGETS / FILES / DIRECTORY / OPTIONAL, `cmake --install`, `DESTDIR`)
