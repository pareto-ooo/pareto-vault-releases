---
id: governance
scope: scoped
severity: critical
applies-to:
  - "**/*.tf"
  - "**/*.hcl"
schemaVersion: 1
---

# Infrastructure standards: Cloud governance

How the Pareto Cloud Governance Policy translates into infrastructure code. These are the ground truth rules for every Azure resource built in infrastructure code. Each rule names the policy rule it comes from (for example RM-04), so it can be easily referenced in planning documents.

## Governance philosophy

- **Compliant by default.** Governance is built into the modules, so a resource is compliant when it is created, not after a review.
- **The workload team owns its workload.** Within the boundaries of the policy, the workload team decides and is accountable for security and cost (GS-01, CM-01).
- **Policy wins.** Where this standard and the Cloud Governance Policy differ, the policy applies and this standard is corrected.

## Tenants and subscriptions

- **Everything is deployed in the main Pareto tenant.** A workload that needs full isolation is deployed in a separate tenant, and nothing is shared between the two (RM-01). **TODO:** confirm which workloads, if any, run in a separate tenant.
- **Production and non-production never share a subscription.** The production environment has its own subscription; non-production environments use dev/test subscriptions (RM-03).
- **Management groups are not managed by workload code.** The management group structure is set by the policy, and a workload never creates or moves management groups (RM-02).

## Tags

- **Every resource that supports tags carries the mandatory tags** (RM-04):
  | Tag | Value |
  |-----|-------|
  | `WorkloadName` | The workload the resource belongs to |
  | `DataClassification` | The classification of the data in the resource |
  | `BusinessCriticality` | `MissionCritical`, `BusinessCritical`, `BusinessSupport` or `NonCritical` |
  | `BusinessOwner` | The owning department |
  | `WorkloadTeam` | The team responsible for the workload |
- **Temporary project resources carry `RoadmapItemUID`.** Resources created only for a roadmap item and removed after the project are tagged with its financial code (CM-03).
- **Tags are set once per environment** in `env.hcl` and applied by every module. Modules do not add or override the mandatory tags.
- **Tag values are validated.** A module rejects a `BusinessCriticality` value outside the allowed list, so a wrong tag fails at plan.

## Identity and access

- **Entra ID is the only identity system.** Resources use Entra ID authentication; local authentication (shared keys, local SQL logins, access keys) is disabled wherever the resource supports Entra ID (IM-01).
- **Workloads authenticate with managed identities.** Service-to-service access uses managed identities; pipelines use Workload Identity Federation. A service principal with a secret is only used when neither is possible (IM-03).
- **Secrets live in Azure Key Vault** and are read through managed identities, never stored in code, configuration or state outputs (IM-08).
- **Built-in roles only.** A custom role is an exception that needs a documented reason (PA-07).
- **Least privilege, at the narrowest scope.** A role is assigned on the resource that needs it, not on the resource group or subscription, unless the access genuinely spans them (PA-07).
- **Roles go to groups and identities, not to people.** Code never assigns standing Owner or production Contributor rights to user accounts; those are activated through PIM (PA-02).
- **Role assignments are traceable.** A pull request that adds or changes role assignments references the ticket in the operational ticketing system (PA-03).

## Network

- **Network boundaries follow the workload's segmentation strategy** (NS-01).
- **Public network access is a deliberate choice.** Based on the workload's risk assessment, services are made private (private endpoints, VNet integration) or restricted with network rules. Public access with no restriction needs a documented reason (NS-02).
- **No public IP addresses on virtual machines.** Public access goes through a gateway or load balancer (NS-02).
- **Insecure protocols are disabled** where a resource allows it, such as FTP on App Services and Function Apps (NS-08).

## Data protection

- **TLS 1.2 or higher and HTTPS only** on every resource that supports the setting (DP-03).
- **Encryption at rest is never disabled** (DP-04).
- **Certificates follow the company process.** Self-signed certificates only for internal use, wildcard certificates only for internal-facing services (DP-07).
- **Key Vaults are hardened.** RBAC authorisation, soft delete and purge protection on, diagnostic logging on, and one vault per application (DP-08).

## Logging and threat detection

- **Microsoft Defender for Cloud covers Azure resources** for threat detection (LT-01).
- **Every resource sends its resource logs** to the workload's Log Analytics workspace through a diagnostic setting (LT-03).
- **Activity logs are kept beyond the default 90 days** by routing them to Log Analytics (LT-05, LT-06).
- **Retention follows the ISO's requirements**; within those, the workload team sets the retention per workspace (LT-06).

## Backup and recovery

- **The workload defines backup per resource** and enables it in code where the resource supports it (BR-01).
- **Storage holding business data has blob versioning and soft delete** on (BR-01).

## Cost management

- **Every subscription has a monthly budget** in code, based on the approved yearly budget, with alerts on actual and forecasted cost (for example at 90% and 100%). The actual budget will be configured on the relevant subscription. (CM-02)

## DevOps security

- **Infrastructure repositories pass the security gate** of the pipeline quality gates, with its secret, dependency and code scanning (GS-10, DS-02).
- **No secrets in code or pipeline definitions.** Keys and credentials come from Key Vault or the service connection (DS-03).
- **Audit logging is enabled on the DevOps platform**: the Azure DevOps organisation and its pipelines, or the GitHub organisation and its workflows (DS-07).

## Not covered here

These policy rules are organisational processes, not infrastructure code, and are handled outside this standard: privileged access management and access reviews (PA-01 to PA-06, PA-08), incident response (IR-01 to IR-07), endpoint security and Northwave onboarding (GS-09, ES-01 to ES-03), vulnerability assessment and red teaming (PV-05 to PV-07), threat modelling (DS-01) and cost reporting (CM-04).
