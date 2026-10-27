---
id: release
scope: scoped
severity: warning
audience: all
applies-to:
  - "**/*.yml"
  - "**/*.yaml"
schemaVersion: 1
---

# Pipeline standards: Release

How a release moves through its stages and gates. Whether a workload needs a specific release strategy (such as blue/green) is decided by the workload; when it does, it is built according to these rules.

## Gates

- **Every approval gets its own stage.** Approvals are checked when a stage (Azure DevOps) or an environment-bound job (GitHub Actions) starts, so each moment a person must approve (before a deploy, before a swap) is the start of its own stage or job.

## Promotion

- **A project uses one of two promotion patterns,** environment branches or tag promotion, and records which one in its project definition (`project-records`). Both are standard; neither is a deviation. Either way, a change is promoted between environments, never rebuilt (`compliance`).
- **Tag promotion: build once, promote the tag.** A push to `main` deploys the first environment. A release tag on `main` (`v1.4.0`) starts one run that builds once and deploys that artifact to each later environment in order (`acc`, then `prd`), each behind its own approval.
- **Environment branches: one branch per environment, and only the first one builds.** A push to the first environment's branch builds the image and its migration artifact and deploys them. A push to a later environment's branch builds nothing: it deploys the image that runs in the previous environment, by digest, with the migration artifact of the run that built that image.
- **A later environment deploys only what the previous one runs.** The deploy reads the digest running in the previous environment and checks that the image's source tree (`containers`) equals the tree of the commit being deployed. A merge creates a new commit but not a new tree, so the check holds across a promotion pull request. When the trees differ or no image is found, the deploy fails; it never falls back to a build.
- **An environment branch only takes changes from the previous environment's branch,** so it always holds what the previous environment tested. A hotfix enters through the first environment's branch and is promoted like any other change.
- **Build artifacts are kept until the release reaches production.** The pipeline's artifact retention is at least the longest time a change can wait between the first environment and production.
- **Infrastructure is re-planned in each environment,** not promoted as an artifact (`terraform`).

## Repeatable steps

- **An action that is not safe to repeat checks the state first.** Most steps are idempotent through the tool. An action that toggles state, such as a slot swap, checks the current state before it acts, so a re-run never undoes a release.

## Performance test

- **No first go-live of a critical workload without a performance test.** Before a workload tagged `BusinessCritical` or `MissionCritical` goes to production for the first time, a performance test runs against acceptance with production tiers. The production tier is the one that passed. For other workloads a performance test is optional.
- **Targets come from the workload profile.** The test applies the expected peak load and checks the performance targets recorded in the workload profile.
- **A significant change to a critical workload is tested again.** A change that affects load or capacity (a new peak, a new heavy feature, a smaller tier) gets a new performance test before it reaches production.
- **Load testing uses the Microsoft service first.** Azure Load Testing is preferred over a self-hosted or third-party tool.
