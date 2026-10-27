---
id: governance
scope: scoped
severity: critical
audience: all
applies-to:
  - "**/*.tf"
  - "**/*.hcl"
schemaVersion: 1
---

# Infrastructure standards: Cloud governance

How our cloud governance translates into infrastructure code: tenants and subscriptions, tags, identity, network, data protection, logging, backup and cost. These are the ground truth rules for every Azure resource built in infrastructure code. They are based on the Microsoft Cloud Security Benchmark.

## Governance philosophy

- **Compliant by default.** Governance is built into the modules, so a resource is compliant when it is created, not after a review.
- **The workload team owns its workload.** Within the boundaries of these rules, the workload team decides and is accountable for security and cost.

## Tenants and subscriptions

- **Everything is deployed in the main Pareto tenant.** A workload that needs full isolation is deployed in a separate tenant, and nothing is shared between the two. **TODO:** confirm which workloads, if any, run in a separate tenant.
- **Production and non-production never share a subscription.** The production environment has its own subscription; non-production environments use dev/test subscriptions.
- **Management groups are not managed by workload code.** The management group structure is set by the platform owners, and a workload never creates or moves management groups.

## Tags

- **Every resource that supports tags carries the mandatory tags**:
  | Tag | Value |
  |-----|-------|
  | `WorkloadName` | The workload the resource belongs to |
  | `DataClassification` | The classification of the data in the resource |
  | `BusinessCriticality` | `MissionCritical`, `BusinessCritical`, `BusinessSupport` or `NonCritical` |
  | `BusinessOwner` | The owning department |
  | `WorkloadTeam` | The team responsible for the workload |
- **Temporary project resources carry `RoadmapItemUID`.** Resources created only for a roadmap item and removed after the project are tagged with that roadmap item's identifier.
- **Tags are set once per environment** in `env.hcl` and applied by every module. Modules do not add or override the mandatory tags.
- **Tag values are validated.** A module rejects a `BusinessCriticality` value outside the allowed list, so a wrong tag fails at plan.

## Identity and access

- **Entra ID is the only identity system.** Resources use Entra ID authentication; local authentication (shared keys, local SQL logins, access keys) is disabled wherever the resource supports Entra ID.
- **Workloads authenticate with managed identities.** Service-to-service access uses managed identities; pipelines use Workload Identity Federation. A service principal with a secret is only used when neither is possible.
- **Secrets live in Azure Key Vault** and are read through managed identities, never stored in code, configuration or state outputs.
- **Built-in roles only.** A custom role is an exception that needs a documented reason.
- **Least privilege, at the narrowest scope.** A role is assigned on the resource that needs it, not on the resource group or subscription, unless the access genuinely spans them.
- **Roles go to groups and identities, not to people.** Code never assigns standing Owner or production Contributor rights to user accounts; those are activated through PIM.
- **Role assignments are traceable.** A pull request that adds or changes role assignments references the work item that requested them.

## Network

- **Network boundaries follow the workload's segmentation strategy**.
- **Public network access is a deliberate choice.** Based on the workload's risk assessment, services are made private (private endpoints, VNet integration) or restricted with network rules. Public access with no restriction needs a documented reason.
- **The default stack has a network default per data classification**. The workload's highest classification (`compliance`) picks the row; a different choice is a recorded decision.

  | Highest classification | App Service | Azure SQL, Cosmos DB, Storage, Key Vault |
  |---|---|---|
  | `Public`, `Internal` | Public inbound over HTTPS | Public endpoint, firewall default-deny, open only to the app's outbound addresses |
  | `Confidential` | Public inbound over HTTPS; outbound through VNet integration | Private endpoint, public network access disabled |
  | `Restricted` | As `Confidential`, with a web application firewall in front of every public endpoint | Private endpoint, public network access disabled |

- **Service Bus and the container registry keep public endpoints** at every level, with local authentication disabled, because events carry identifiers only (`inter-service-communication`) and images hold no data. A namespace that carries data in its messages follows the data services column.
- **No public IP addresses on virtual machines.** Public access goes through a gateway or load balancer.
- **Insecure protocols are disabled** where a resource allows it, such as FTP on App Services and Function Apps.

## Data protection

- **TLS 1.2 or higher and HTTPS only** on every resource that supports the setting.
- **Encryption at rest is never disabled**.
- **Certificates follow the company process.** Self-signed certificates only for internal use, wildcard certificates only for internal-facing services.
- **Key Vaults are hardened.** RBAC authorisation, soft delete and purge protection on, diagnostic logging on, and one vault per application.

## Logging and threat detection

- **Microsoft Defender for Cloud covers Azure resources** for threat detection.
- **Every resource sends its resource logs** to the workload's Log Analytics workspace through a diagnostic setting.
- **Activity logs are kept beyond the default 90 days** by routing them to Log Analytics.
- **Retention follows the ISO's requirements**; within those, the workload team sets the retention per workspace.

## Backup and recovery

- **The workload defines backup per resource** and enables it in code where the resource supports it.
- **Storage holding business data has blob versioning and soft delete** on.

## Cost management

- **Every subscription has a monthly budget** in code, based on the approved yearly budget, with alerts on actual and forecasted cost (for example at 90% and 100%). The actual budget will be configured on the relevant subscription.

## DevOps security

- **Infrastructure repositories pass the security gate** of the pipeline quality gates, with its secret, dependency and code scanning.
- **No secrets in code or pipeline definitions.** Keys and credentials come from Key Vault or the service connection.
- **Audit logging is enabled on the DevOps platform**: the Azure DevOps organisation and its pipelines, or the GitHub organisation and its workflows.

## Not covered here

These are organisational processes, not infrastructure code, and are handled outside this standard: privileged access management and access reviews (`security`), incident response, endpoint security, vulnerability assessment and red teaming, threat modelling (`compliance`) and cost reporting.
