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

The goal of reproduction is to demonstrate the current limitation: with only a
single **global** sample rate, you cannot sample high-volume operations
aggressively while keeping full visibility into low-volume operations. Lowering
the global rate to reduce quota cost causes low-volume operations to be missed
entirely.

1. Clone the fork and check out the working branch:
   ```bash
   git clone https://github.com/ChaoyangS/console.git
   cd console
   git checkout feat/operation-level-sample-rates
   ```
2. Install the pinned toolchain. The repo requires Node `>=24.14.1` (see
   `.node-version`) and pnpm `10.33.2`, and enforces this via `engine-strict`:
   ```bash
   fnm install        # reads .node-version (24.14.1)
   fnm use
   corepack enable
   corepack prepare pnpm@10.33.2 --activate
   ```
3. Install dependencies from the repo root:
   ```bash
   pnpm install
   ```
4. Run the demonstration script, which simulates a mixed-volume workload (two
   high-volume operations plus several low-volume ones) and prints how many
   operations each sampling strategy would report:
   ```bash
   cd packages/libraries/core
   pnpm exec tsx demo/operation-sample-rates.ts
   ```
5. Observe the output and confirm the problem:
   - In the **`Global sampleRate = 0.05`** section, the low-volume operations
     (`AdminAuditReport`, `RareMigrationQuery`) report `sent=0` and are marked
     `MISSED` — a low global rate saves quota but loses all visibility into rare
     operations.
   - In the **`Global sampleRate = 1.0`** section, every operation is billed at
     100%, including the two high-volume operations that dominate the total —
     this reproduces the "overpay for quota" side of the trade-off.

This confirms the issue: a single global sample rate forces a choice between
overpaying on quota and losing visibility into low-volume operations.

### Reproduction Evidence

- **Branch in my fork:** https://github.com/ChaoyangS/console/tree/feat/operation-level-sample-rates
- **Commit showing reproduction:** https://github.com/ChaoyangS/console/commit/8e7f50f10
- **Screenshots/logs:** Output of `pnpm exec tsx demo/operation-sample-rates.ts`:
  ```
  === Global sampleRate = 0.05 (low-volume ops missed) ===
    AdminAuditReport     volume=     40  sent=      0 (  0.0%)  MISSED
    RareMigrationQuery   volume=      8  sent=      0 (  0.0%)  MISSED
    TOTAL                volume= 751248  sent=  37581 (5.0% of quota)

  === Operation-level: /^HighVolume/ -> 1%, fallback 100% ===
    HighVolumeFeed       volume= 500000  sent=   5125 (  1.0%)  visible
    AdminAuditReport     volume=     40  sent=     40 (100.0%)  visible
    RareMigrationQuery   volume=      8  sent=      8 (100.0%)  visible
    TOTAL                volume= 751248  sent=   8998 (1.2% of quota)
  ```
- **My findings:** Sampling lives in `@graphql-hive/core`
  (`packages/libraries/core/src/client/`). The existing `randomSampling` in
  `sampling.ts` applies one global rate to every operation, and `usage.ts`
  selects it in the `shouldInclude` logic. There is already a per-operation
  `exclude` list and a dynamic `sampler` callback, but no declarative way to set
  different rates per operation. The reproduction confirms that lowering the
  global rate drops 100% of low-volume operations, while operation-level rates
  cut billed volume to ~1.2% of quota and still report low-volume operations at
  100%.

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** A single global sample rate cannot serve a workload that mixes
high-volume and low-volume operations. Users need a way to set sample rates per
operation, where an operation-level rate takes priority and the global rate is
the fallback.

**Match:** The usage plugin in `@graphql-hive/core` already has closely related
patterns to follow: the `exclude` option matches operations by exact name or
regex, and the `sampler` callback computes a rate dynamically from a
`SamplingContext` that includes `operationName`. The new config mirrors
`exclude`'s name/regex matching and reuses the existing `shouldInclude`
selection point in `usage.ts`.

**Plan:** Step-by-step implementation plan:
1. Add an `OperationSampleRate` type (`name` or `regex` + `sampleRate`) and a new
   `sampleRates?: OperationSampleRate[]` option to `HiveUsagePluginOptions` in
   `packages/libraries/core/src/client/types.ts`.
2. Add an `operationSampling()` factory in `sampling.ts` that returns a
   `shouldInclude(context)` function: first matching rule wins, unmatched
   operations fall back to the global `sampleRate`, and rules are validated
   eagerly (exactly one of name/regex, rate within `0..1`).
3. Wire `sampleRates` into the `shouldInclude` selection in `usage.ts`, keeping
   precedence: `exclude` > `sampler` > `sampleRates` > global `sampleRate`.
4. Add unit tests for the precedence and validation logic, plus an end-to-end
   usage test, and a runnable demo that quantifies the quota savings.
5. Add a changeset describing the new option (semver `minor`).

**Implement:** Branch:
https://github.com/ChaoyangS/console/tree/feat/operation-level-sample-rates

**Review:** Followed existing code style and the `exclude`/`sampler` patterns;
added a changeset per the project's Changesets workflow; kept the change scoped
to `@graphql-hive/core` with no breaking changes (the new option is optional and
defaults to existing behavior).

**Evaluate:** `pnpm --filter @graphql-hive/core typecheck` passes, the new and
existing tests pass (`vitest run` on the sampling and usage suites), and the
demo script shows operation-level sampling billing ~1.2% of quota while keeping
low-volume operations at 100% visibility.

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
