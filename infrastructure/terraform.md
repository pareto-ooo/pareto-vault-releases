---
id: terraform
scope: scoped
severity: warning
applies-to:
  - "**/*.tf"
  - "**/*.hcl"
schemaVersion: 1
---

# Infrastructure standards: Terraform and Terragrunt

How we build infrastructure as code. These are the ground truth rules for all Terraform modules, Terragrunt configuration and remote state in a repository. How the infrastructure is validated and deployed is described in the pipeline standards and not repeated here.

## Infrastructure philosophy

- **The Terragrunt documentation is the reference.** Where this standard is silent, the current Terragrunt documentation and its recommendations apply.
- **Derive what can be derived.** State keys come from the unit path, resource names from the environment tokens, location from the resource group unit. A derived value cannot drift.
- **Commit what must be stable.** Values that must survive every apply unchanged (resource names, provider lock files) are committed under review, never computed at apply time (no `random_string` in names).
- **Generate boilerplate once.** Backend, provider and version blocks are generated into every unit from one root configuration, never copied per environment.
- **One state per unit, per environment.** Blast radius, plan time and lock contention follow the state file.
- **Terragrunt owns the order.** Units declare what they depend on, and Terragrunt builds the dependency tree and runs every unit in the correct order. Values flow between units as outputs and inputs, not as pipeline variables per environment.
- **Bootstrap lives outside Terraform.** Whatever has to exist before Terraform can run (the state storage) is created by a reviewed script, never by the configuration that depends on it.
- **Fail loudly at the edge.** A missing backend or a changed lock file stops the run at the point of cause, not as a confusing error further on.
- **Runnable without a cloud.** Formatting and validation work offline, so a repository gets real feedback before any environment exists.

## Folder structure

All infrastructure code lives in the `infrastructure/` folder at the root of the repository, split in two tiers. Every repository with infrastructure uses Terragrunt with this layout, also with a single environment or component.

```
infrastructure/
├── live/
│   ├── root.hcl
│   ├── _envcommon/
│   │   └── <component>.hcl
│   └── <env>/
│       ├── env.hcl
│       └── <component>/
│           └── terragrunt.hcl
├── modules/
│   └── <module>/
├── create-backend.sh
└── README.md
```

| Folder | Contains | Language |
|--------|----------|----------|
| `live/` | Units: what is deployed where | Terragrunt (`.hcl`) only |
| `live/_envcommon/` | Module source, dependencies and inputs shared by a component in every environment | Terragrunt |
| `live/<env>/` | Environment identity and that environment's units | Terragrunt |
| `modules/` | Reusable Terraform modules | Terraform (`.tf`) only |

- **No `.tf` file lives under `live/`, no `.hcl` configuration under `modules/`.**

## Units

A unit is a leaf directory under `live/<env>/`: one component, one `terragrunt.hcl`, one state file.

- **The root configuration is `root.hcl`, never `terragrunt.hcl`.** As the Terragrunt documentation recommends: a root `terragrunt.hcl` is hard to tell apart from a unit and is picked up by `run --all` as one.
- **One unit runs one module.** The unit points at its module in `modules/`; there is no per-environment wrapper module.
- **A unit only holds what differs per environment** (SKU, instance count, retention, zone redundancy). Everything else comes from the levels above.
- **A unit with nothing to size keeps an empty `inputs` block.** The unit still owns its state; an empty unit is not tidied away.

## Configuration levels

Configuration has three levels, and only three.

| Level | File | Holds |
|-------|------|-------|
| Environment identity | `live/<env>/env.hcl` | Environment name and short name, subscription and tenant, region, tags, pipeline principal, environment-wide network values, management resource group and backend coordinates |
| Component-wide | `live/_envcommon/<component>.hcl` | Module source, dependencies, and the inputs identical in every environment |
| Per-environment sizing | `live/<env>/<component>/terragrunt.hcl` | Only what genuinely differs per environment |

- **A value lives at the highest level where it is the same.** It is never repeated at a lower level.
- **One `env.hcl` per environment.** The environment identity is not split over several files.
- **`root.hcl` passes the environment identity as inputs** to every unit, so units do not read `env.hcl` themselves for these values.
- **Adding an environment adds files, never edits.** A new `env.hcl` and one `terragrunt.hcl` per component, plus bootstrapping its backend. The pipeline path filters are the only change outside `infrastructure/`.

## Dependencies

- **Units depend on each other through `dependency` blocks.** A unit that needs a value from another unit reads that unit's outputs. Terragrunt builds the dependency tree from these blocks and runs the units in order.
- **Dependencies are declared in `_envcommon`.** The dependency structure is the same in every environment, so it is declared once per component.
- **No `terraform_remote_state` data sources and no hand-copied values.** A value owned by another unit always comes through its outputs.
- **Mock outputs only where the real ones cannot exist yet.** `mock_outputs` make a unit validatable and plannable before its dependency has been applied. They are limited to `validate` and `plan` through `mock_outputs_allowed_terraform_commands`, never `apply`.
- **Mocks never replace real state.** Merging mocks with existing state uses `mock_outputs_merge_strategy_with_state`, not the deprecated `mock_outputs_merge_with_state`.
- **`dependencies` only for order without values.** When a unit must run after another but needs none of its outputs, it uses a `dependencies` block instead of a `dependency` block.
- **Run the whole environment through Terragrunt.** Plan and apply run with `run --all`, so the dependency order is always respected.

## Modules

`infrastructure/modules/` contains reusable Terraform modules.

- **One module per component.** A module wraps one component (a Key Vault, a Function App) and its close companions, and exposes what other units need as outputs.
- **No `terraform{}` or `provider{}` block in any module.** Both are generated by `root.hcl`. A module that declares its own fights the generator.
- **Resource groups are a unit.** A `resource-groups` unit creates the workload resource groups of an environment. Other units depend on it and take the name and location from its outputs.
- **The management resource group is read, not created.** The resource group holding the state account exists before Terraform runs (see Remote state) and is only read with a data lookup.
- **Names are built in one place.** A module derives its resource names from the environment tokens in its locals, not from free-text inputs.

## Remote state

- **One state storage account per environment.** Its coordinates live in that environment's `env.hcl`, so one environment's state cannot be reached with another environment's permissions.
- **State keys are derived from the unit path,** never hand-written. A unit's location and the name of its state file cannot drift apart.
- **State keys always use forward slashes.** The derived path is normalised to `/`, because on Windows it contains backslashes and would point to a different state file than the pipeline uses.
- **The state account is bootstrapped out of band** by `create-backend.sh`: once per environment, idempotent, with a dry-run mode that prints every command first.
- **The bootstrap script is not interactive.** It takes the environment as an argument and reads its values from that environment's `env.hcl`.
- **The bootstrap script checks before it creates.** It checks whether the state account already exists and whether the name is still available, and stops with a clear message when the name is taken by someone else.
- **The pipeline requires the backend to exist** (`--backend-require-bootstrap`). It never lets Terragrunt create one (`--backend-bootstrap`).
- **State is protected.** Blob versioning, soft delete and a delete lock on every state account, because state has no backup anywhere else.
- **State accounts are locked down.** Public blob access disabled, HTTPS only and TLS 1.2 minimum on every state account.
- **State accounts are named `st<app>tf<company><env>`.** This follows the storage account format of the naming convention, with `tf` marking the account as Terraform state, so every app repo derives the same kind of name.
- **State access uses Entra ID.** OIDC and Azure AD auth for the backend; shared-key access is disabled on state accounts.
- **Role assignments are requested, not applied.** The bootstrap script prints the role assignments the pipeline identity needs; a delegated admin applies them.

## Deletion protection

Data that cannot be recovered once deleted (databases, state accounts) has two guards: one in Terraform and one in Azure.

- **Both layers are required.** `prevent_destroy` in the resource's `lifecycle` block fails the plan on a destroy or a replacement. A `CanNotDelete` management lock blocks every other delete path (portal, CLI, ARM, a Terraform run where the lifecycle guard was edited away). Either one alone leaves a gap.
- **The lifecycle guard is unconditional.** `prevent_destroy` must be a literal, so it cannot be wired to a variable, and removing it is a code change visible in a pull request.
- **The lock may be variable-driven** (for example `deletion_protection`) for non-production, so dropping it is an explicit, reviewable act. The lock at server scope is inherited by the databases under it.
- **The guard lives inside the module**, so every consumer inherits it without remembering to add it. A geo-secondary database carries the same two layers.
- **Forcing a replace in a non-production environment** means disabling the lock variable and making a deliberate, pull-request-visible edit to the `lifecycle` block for the duration, and restoring both right after. Production deviations need an ADR update.
- **Both layers are never dropped together on a production database** unless the intent is permanent deletion, with the data exported and a final backup restore tested.
- **The state account guard is never disabled.** If the account must be replaced, the state is migrated to a new account first and the old one deleted manually, never through `terraform destroy`.

## Naming and stable values

- **Names follow the Azure naming convention.**
- **Name length limits are validated in the module.** A module that builds a name for a type with a short maximum length validates the length, so a too-long name fails at plan instead of at apply.

## Terraform names

Names inside the code follow the Terraform best practices referenced by the Cloud Governance Policy (RM-05).

- **Underscores, not hyphens,** in resource names, data source names, variables, locals and outputs.
- **Lowercase letters and digits only.**
- **Do not repeat the resource type** in the resource name, not partially and not completely.
- **Singular nouns** for names.
- **Hyphens only in values a human sees,** such as the Azure resource name or a DNS name.
- **`count` or `for_each` is the first argument** of a resource or data source block, followed by an empty line.
- **`tags` is the last real argument** of a resource that supports tags.

## Versions and lock files

- **Terraform and provider versions are constrained** in the generated version block: a minimum Terraform version and a pessimistic constraint per provider (`~>`).
- **Lock files belong to the unit.** Each unit commits its `.terraform.lock.hcl`. Lock files under `modules/` are git-ignored.
- **Lock files hold hashes for every platform that runs Terraform,** at least `linux_amd64` for the pipeline and `windows_amd64` and `darwin_arm64` for developers, created with `terraform providers lock`.
- **The pipeline initialises with a read-only lock file** (`-lockfile=readonly`), passed through `TF_CLI_ARGS_init` because Terragrunt runs `init` itself.

## Authentication

- **Terraform authenticates through the pipeline's Workload Identity Federation service connection** with OIDC, using the `ARM_*` environment variables. No client secrets.
- **Long runs refresh their own token.** The service connection id is passed to the provider, so an apply that outlives the short-lived OIDC token requests a new one instead of failing halfway.

## Validation

- **Two formatters.** `terraform fmt` for `modules/`, `terragrunt hcl fmt` for `live/`. Terraform does not parse Terragrunt HCL.
- **Offline validation runs at module level.** Every module in `modules/` is validated on its own with `terraform init -backend=false` and `terraform validate`. `live/` is checked with `terragrunt hcl validate`. Both run on every change, without Azure access, before any environment exists.
- **Unit validation runs against Azure.** Validating or planning units through Terragrunt reads dependency outputs from the backend, so it only runs where the environment's backend is reachable. No workarounds in `dependency` blocks to make units validate offline.

## Applying

- **Apply re-plans.** Infrastructure is the exception to "deploy the published artifact" in the pipeline standards. Apply runs `run --all apply` against the current state instead of a saved plan, because a saved plan goes stale whenever mocks were used, resources were destroyed, or a new dependency was introduced.
- **The plan stage is for review.** Its output is published so the reviewer can read it before approving the apply stage.
- **Validate and plan run locally, apply only from the pipeline.** A developer may validate and plan against an environment from a workstation; changes are only applied by the pipeline.

## Comments

- **No comments by default.** Module, variable and local names should communicate what happens.
- **Variables and outputs have a `description` field.** It is the module's documentation, not comment smell.
- **A deliberate exception is commented,** such as an empty unit that must not be removed or a lifecycle rule that ignores changes, with the reason in one line. When a security scanner flags the exception, that comment is the scanner's suppression (`#trivy:ignore:<ID>:exp:<yyyy-mm-dd>` with the reason and the decision), directly above the resource, so the reason and the acceptance are one line and cannot drift apart.
- **TODO:** A TODO comment is only for planned future work and requires a reference to a work item.
