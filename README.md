# Contribution [#]: Selenium tests: check console output

**Contribution Number:1
**Student:Fares Ibrahim
**Issue:https://github.com/searxng/searxng/issues/338
**Status: Phase 1

---

## Why I Chose This Issue

This issue interests me because it sits at the intersection of testing infrastructure and frontend quality—catching JavaScript errors in CI before they reach users is exactly the kind of reliability improvement I want to get better at. I have Python skills from my research (implementing algorithms in C++/Python, building Flask/Django apps) and solid JavaScript familiarity from React-based projects. I'm hoping to learn more about Selenium/Robot Framework browser testing patterns and how open-source CI pipelines validate frontend code—this seems like a great first contribution to SearXNG that builds on what I already know while pushing me into infrastructure testing I haven't explored much yet.

---

## Understanding the Issue

### Problem Description

The browser tests (Robot Framework via Splinter) don't check the browser console for JavaScript errors. PR #337 fixed a null pointer in searx.js that was only caught by chance, because the existing tests only assert on DOM text and element state — they'd pass even with JS errors in the console

### Expected Behavior

After each page interaction, the test should query driver.manage().logs().get("browser") and fail if any SEVERE entries (uncaught JS errors) are present. This would catch regressions like the one in #337 automatically.

### Current Behavior

Console output is completely ignored. JS errors (even critical ones like Uncaught TypeError: n.querySelector(...) is null) don't affect test results.

### Affected Components

- tests/robot/__main__.py — the test runner; this is where the Browser instance is created (Firefox, headless) and where console checking could be injected after each test
- tests/robot/test_webapp.py — the individual test functions, or alternatively a teardown hook in the runner
- Splinter uses Selenium under the hood, so console access would be browser.driver.manage().logs().get("browser") — no new library needed
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
