# Contribution 1: Respect GH_HOST in GitHub Gist resolution

**Contribution Number:** 1  
**Student:** Adiba Akter  
**Issue:** https://github.com/astral-sh/uv/issues/15109  
**Fork:** https://github.com/AdibaAdi/uv  
**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose this issue because it gives me the chance to work with Rust in a real, high-impact open-source project while still connecting to Python tooling, which is directly relevant to my background and career goals. The issue is about making uv respect the `GH_HOST` environment variable when resolving GitHub Gist URLs, which matters for users working with GitHub Enterprise or custom GitHub hosts.

This issue interests me because it combines CLI behavior, environment variable handling, URL resolution, and developer workflow support. Since uv is a widely used Python tooling project, contributing here would help me learn how production-level developer tools handle configuration and compatibility across different environments.

---

## Understanding the Issue

### Problem Description

uv currently supports resolving GitHub Gist URLs, but the issue asks for support for the `GH_HOST` environment variable when resolving Gists. This would make uv work better for users who use GitHub Enterprise or custom GitHub hosts, similar to how the GitHub CLI supports host configuration.

### Expected Behavior

When a user sets `GH_HOST`, uv should use that configured GitHub host while resolving GitHub Gist URLs, instead of assuming the default public GitHub host.

### Current Behavior

Based on the issue, uv does not currently respect `GH_HOST` for GitHub Gist resolution. This means users working with GitHub Enterprise Gists may not get the expected behavior.

### Affected Components

The likely affected components are the parts of uv that handle GitHub Gist URL parsing, GitHub host resolution, environment variable configuration, and related tests.

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
