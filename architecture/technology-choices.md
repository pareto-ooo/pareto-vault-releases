---
id: technology-choices
scope: global
severity: warning
schemaVersion: 1
---

# Architecture standards: Technology choices

How we choose the services and technologies a workload is built on. These are the ground truth rules for every technology decision in a workload, whatever kind of workload it is, and work out the architecture principles. They name no specific services or versions, because the offering changes over time; every decision uses what is available at the moment it is made.

## Architecture philosophy

- **Reuse before buy before build.** A new need is first matched against the existing application landscape and platforms; buying or building only follows when they are not sufficient.
- **Fit for purpose.** A workload uses the services its purpose needs. A custom-built application starts from the default stack; a data platform and an integration each choose what fits them, within these rules.
- **Microsoft first.** Azure is our cloud platform (Cloud Governance 2.1). When a Microsoft service and an alternative both meet the requirements, the Microsoft service is chosen.
- **Managed over self-hosted.** Cloud first, and a managed service (SaaS, PaaS, serverless) is preferred over running and patching it ourselves on virtual machines or containers. Self-hosting needs a reason the managed option cannot meet.
- **Fewer moving parts.** A new service is only added when an existing one in the workload cannot do the job. Every service adds cost, security surface and operational work.
- **The workload team decides, within the boundaries.** Technology choices belong to the workload team (Cloud Governance GS-01), inside these rules and the company policies.

## Selection criteria

A service is chosen when it meets these criteria. A service that fails one needs a reason in an architecture decision.

- **Requirements.** It meets the workload's functional needs and the performance targets of its workload profile.
- **Entra ID.** It supports Entra ID authentication and managed identities, so no local accounts or stored keys are needed (Cloud Governance IM-01, IM-03).
- **Network control.** It supports the network isolation the workload's risk assessment requires, such as private endpoints or network rules (Cloud Governance NS-02).
- **Infrastructure as code.** It can be created and configured fully through infrastructure code (Terraform).
- **Cost at expected scale.** Its cost at the workload's expected load fits the budget, and it can be sized down or paused outside production.
- **Supported lifecycle.** It is generally available and supported by its vendor for the expected lifetime of the workload.
- **Team knowledge.** The workload team can build, operate and troubleshoot it, or can learn to within the project.

## Versions and lifecycle

These rules apply to every runtime, framework, language, database engine and platform a workload uses. They name no version numbers; the current version is looked up at the moment of the decision.

- **Long-term support first.** Where a product has a long-term support (LTS) release line, the latest LTS release is used, not a short-term or preview release.
- **Latest supported major version otherwise.** Where a product has no LTS line, the latest generally available major version that is in the vendor's mainstream support is used.
- **Managed services stay current.** For a managed service that is updated by the vendor, the newest compatibility or feature level it offers is used for new work.
- **New work starts on the current version.** A new project or component never starts on an older version than these rules give, copied from an earlier project or an example.
- **Container images and tool versions name the major version.** They are pinned to the chosen major (or LTS) release, never to a floating `latest` and never to an older version.
- **Upgrades are planned before support ends.** A workload moves to the next LTS or supported major version before its current version leaves mainstream support; the upgrade is planned work, not an emergency.
- **Staying on an older version is a deviation.** A workload that keeps or chooses an older version (for compatibility with an existing system, for example) records it as a deviation, signed off by an architect.

## Decisions

- **A deviation is an architecture decision.** Choosing a non-Microsoft service when a Microsoft one meets the requirements, self-hosting a service that is available managed, or accepting a service that fails a selection criterion is a deviation, recorded and signed off by an architect.
- **Proven technology in production.** A preview feature or service is only used in production when no proven alternative exists, through an approved deviation that accepts the risk.
- **A decision is revisited when its reason changes.** When the offering, the cost or the workload changes so that the original reason no longer holds, the decision is reviewed.
