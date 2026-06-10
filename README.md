# Contribution 1: Add Expression scripting support for knn_vector

**Contribution Number:** 1  
**Student:** Adiba Akter  
**Issue:** https://github.com/opensearch-project/k-NN/issues/459  
**Fork:** https://github.com/AdibaAdi/k-NN  
**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose this issue because it connects directly to vector search, OpenSearch, scripting engines and backend search infrastructure. Since I am interested in software engineering work involving data systems and AI infrastructure, this issue gives me a chance to understand how a real search engine plugin handles vector field behavior across different scripting languages.

The issue asks for Mustache and Expression scripting support for the `knn_vector` field. I noticed there is already an open PR focused on Mustache support, so my planned scope is to investigate the remaining Expression scripting behavior and understand what is missing compared with the existing painless scripting path.

---

## Understanding the Issue

### Problem Description

OpenSearch k-NN currently supports painless scripting with the `knn_vector` field type, but the issue asks for support across other OpenSearch scripting languages as well. Since Mustache support already has an open PR, the remaining useful scope appears to be understanding and potentially adding Expression scripting support for `knn_vector`.

### Expected Behavior

Users should be able to use the `knn_vector` field with supported OpenSearch scripting behavior beyond only painless scripting, specifically Expression scripting if that scope is still valid for the project.

### Current Behavior

Based on the issue, `knn_vector` scripting support is currently limited and does not fully support the other scripting languages mentioned in the issue.

### Affected Components

The likely affected components are the k-NN plugin scripting implementation, the existing painless scripting support path, script engine integration and tests around `knn_vector` script behavior.

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
