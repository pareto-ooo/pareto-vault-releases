---
id: naming
scope: scoped
severity: warning
audience: all
applies-to:
  - "**/*.tf"
  - "**/*.hcl"
schemaVersion: 1
---

# Infrastructure standards: Naming convention

How we name Azure resources. These are the ground truth rules for every Azure resource name. They follow the naming convention of the Cloud Governance Policy (RM-05), which is based on the Microsoft Cloud Adoption Framework.

## Naming philosophy

- **A name says what, for what, and where.** Resource type, application and environment can be read from every name without opening the portal.
- **Derived, never typed.** Names are built from the same components in every environment, so they cannot drift between environments.
- **Stable for life.** A resource name is never changed after the first apply, because most Azure resources cannot be renamed without being recreated.

## Naming scope

The scope of a resource decides how unique its name must be, and which components it needs.

| Scope | Unique within | Examples |
|-------|---------------|----------|
| Global | All of Azure (the name is part of a DNS name) | Storage accounts, Key Vaults, SQL servers, Data Factories, App Services, Function Apps, container registries |
| Resource group | The resource group | Virtual networks, public IPs, network interfaces, virtual machines |
| Resource | The parent resource | Subnets, SQL databases, storage containers |

## Components

| Component | Meaning | Rule |
|-----------|---------|------|
| `<type>` | Resource type | The abbreviation from the Microsoft Cloud Adoption Framework |
| `<app>` | Project, application or service the resource is for | One short token per application, the same in every environment |
| `<scope>` | Use of a shared or multi-purpose resource | Only when needed, such as `landing`, `private`, `shared` or `generic` |
| `<company>` | Company short name | `pareto`; required in globally scoped names, allowed in others |
| `<region>` | Azure region | `northeurope` by default; only in network resource names |
| `<env>` | Environment | The environment short name: `dev`, `tst`, `stg`, `acc` or `prd`, or `shared` for a resource that serves several environments |
| `<###>` | Instance number | Three digits, starting at `001`, only for the types that use it |

- **Lowercase only.** Azure treats many names case-insensitively, and mixed case leads to drift between code and portal.
- **Hyphens separate the components**, except where the resource type does not allow them (see Restricted names).
- **Components keep their order.** Type first, environment and instance number last, as in the formats below.
- **Every name carries an environment.** A resource that genuinely serves several environments uses `shared` in place of the environment.
- **A role qualifier comes last.** When a workload has several resources of one type that differ in role (an API and a web front end, or the private and public subnets Databricks requires), the role is added after the environment, in place of the instance number. The instance number is for identical copies of one role.

## Formats

| Resource | Scope | Format |
|----------|-------|--------|
| Resource group | Subscription | `rg-<app>-<env>` |
| Virtual network | Resource group | `vnet-<scope>-<region>-<env>-<###>` |
| Subnet | Virtual network | `snet-<scope>-<region>-<env>-<###>` |
| Public IP address | Resource group | `pip-<vm or app>-<region>-<env>-<###>` |
| Network security group | Subnet or NIC | `nsg-<policy or app>-<###>` |
| Network interface | Resource group | `nic-<##>-<vm>-<env>-<###>` |
| Virtual machine | Resource group | `vm<role><env><###>` |
| Web app | Global | `app-<app>-<company>-<env>-<###>` |
| Function app | Global | `func-<app>-<company>-<env>-<###>` |
| SQL server | Global | `sql-<app>-<company>-<env>` |
| SQL database | SQL server | `sqldb-<app>-<env>` |
| Storage account | Global | `st<app><company><env><###>` |
| Data Lake storage account | Global | `dls<##><app><company><env>` |
| Container registry | Global | `cr<app><company><env><###>` |
| Machine Learning workspace | Resource group | `mlw-<app>-<env>` |
| Data Factory | Global | `adf-<app>-<company>-<env>` |
| Diagnostic setting | Target resource | `diag-<app>-<env>` |
| Management lock | Locked resource | `lock-<resource name>` |
| Terraform state resource group | Subscription | `rg-<app>-tfstate-<env>` |
| Terraform state storage account | Global | `st<app>tf<company><env>` (no instance number; `tf` marks it as Terraform state) |

- **Types not in the table follow the same shape.** Globally scoped: `<type>-<app>-<company>-<env>[-<###>]`. Otherwise: `<type>-<app>-<env>[-<###>]`. The type abbreviation comes from the Cloud Adoption Framework.
- **The table is a guideline, not a catalogue.** It shows the common types as examples of the convention; it is not meant to cover every resource type. Any type not listed is derived from the general shape and the Cloud Adoption Framework abbreviations.

## Security groups

Entra ID security groups that grant roles follow their own convention, because they are not Azure resources.

- **Security groups are named `SG_<Purpose>_<Role>[_<Specific>]`.** `SG` marks a security group; `<Purpose>` is the abbreviation of the application or area the group applies to (`ALL` for everybody); `<Role>` is the full role name; `<Specific>` is only added when the same role exists in several contexts.
- **The group description documents the group:** why it exists, who owns it, and whether its membership is managed by ICT or by its owner.

## Restricted names

- **Types without hyphens drop the separators.** Resource types that only allow lowercase letters and digits (storage accounts, container registries, virtual machine names) use the same components in the same order without hyphens.
- **Length limits are respected.** Storage accounts and Key Vaults allow at most 24 characters.
- **Shortening shortens the application token, never the type, environment or instance number.**

## Out of scope

- **Names inside a service are not covered,** such as Databricks catalogs and schemas, or containers and secrets inside a resource. They follow the conventions of that service, consistently within the workload.

## Tags

- **Names do not replace tags.** Workload, data classification, business criticality, owner and team are tags.
