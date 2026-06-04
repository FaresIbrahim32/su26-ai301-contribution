# Contribution [#]: Migrate Makefile `install` target into CMake

**Contribution Number:1
**Student:Fares Ibrahim
**Issue:https://github.com/ponylang/ponyc/issues/3898
**Status: Phase 1

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
- CMake build configuration — the install logic should live here so that values set during `configure` (e.g. `arch`) are carried through to `install`
---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
