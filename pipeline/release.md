---
id: release
scope: scoped
severity: warning
applies-to:
  - "**/*.yml"
  - "**/*.yaml"
schemaVersion: 1
---

# Pipeline standards: Release

How a release moves through its stages and gates. Whether a workload needs a specific release strategy (such as blue/green) is decided by the workload; when it does, it is built according to these rules.

## Gates

- **Every approval gets its own stage.** Approvals are checked when a stage (Azure DevOps) or an environment-bound job (GitHub Actions) starts, so each moment a person must approve (before a deploy, before a swap) is the start of its own stage or job.

## Repeatable steps

- **An action that is not safe to repeat checks the state first.** Most steps are idempotent through the tool. An action that toggles state, such as a slot swap, checks the current state before it acts, so a re-run never undoes a release.

## Performance test

- **No first go-live of a critical workload without a performance test.** Before a workload tagged `BusinessCritical` or `MissionCritical` goes to production for the first time, a performance test runs against acceptance with production tiers. The production tier is the one that passed. For other workloads a performance test is optional.
- **Targets come from the workload profile.** The test applies the expected peak load and checks the performance targets recorded in the workload profile.
- **A significant change to a critical workload is tested again.** A change that affects load or capacity (a new peak, a new heavy feature, a smaller tier) gets a new performance test before it reaches production.
- **Load testing uses the Microsoft service first.** Azure Load Testing is preferred over a self-hosted or third-party tool.
