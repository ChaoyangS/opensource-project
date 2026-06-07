# Contribution 1: Operation-Level Sample Rates for Hive Usage Tracking

**Contribution Number:** 1  
**Student:** Chaoyang Shen  
**Issue:** https://github.com/graphql-hive/console/issues/3556  
**Status:** Phase I – In Progress

---

## Why I Chose This Issue

This issue interests me because it involves designing a configuration-driven backend feature that balances observability, cost efficiency, and user experience. The feature requires thinking about how telemetry and usage data are collected at scale, which is an area I find particularly interesting given my background in backend software engineering and distributed systems. I enjoy working on problems where there are tradeoffs between performance, resource utilization, and visibility into system behavior.

From a learning perspective, this issue provides an opportunity to contribute to a production-grade open-source project while gaining a deeper understanding of GraphQL infrastructure and telemetry systems. The proposed feature also involves API and configuration design decisions, such as precedence rules, matching strategies, and validation logic. These are the kinds of engineering decisions that are common in large-scale backend systems, and I hope to learn how experienced maintainers approach these design challenges in a real-world codebase.

---

## Understanding the Issue

### Problem Description

Currently, GraphQL Hive supports a global sampling rate for usage reporting. This means that every operation executed through a service is sampled using the same probability. While this works well for systems with relatively uniform traffic patterns, it becomes problematic when a service contains both very high-volume and low-volume operations.

If the global sampling rate is set too low, low-volume operations may not be sampled frequently enough, resulting in poor visibility into their usage and performance. On the other hand, if the global sampling rate is increased to ensure visibility for low-volume operations, the large number of high-volume operations can quickly consume the organization's usage quota and increase costs unnecessarily.

The proposed enhancement introduces operation-level sample rates that allow specific GraphQL operations to be sampled differently from the global default. For example, a frequently executed operation could be sampled at 10%, while a rarely used operation could be tracked at 100%. This provides more granular control over observability and usage costs.

Based on the issue discussion, my current understanding is that operation-level sample rates should override the global sample rate when a matching rule is found, while the global sample rate serves as the fallback for all other operations. The feature would support both exact operation name matching and regex-based matching patterns, allowing users to define sampling behavior for individual operations or groups of operations.

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

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
