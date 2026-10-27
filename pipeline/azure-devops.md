---
id: azure-devops
scope: scoped
severity: warning
audience: all
applies-to:
  - "**/*.yml"
  - "**/*.yaml"
schemaVersion: 1
---

# Pipeline standards: Azure DevOps

How we build Azure DevOps pipelines. These are the ground truth rules for all pipeline definitions, templates, scripts and variables in a repository. It covers infrastructure, application build and deploy, and database migrations.

## Pipeline philosophy

- **Readability first.** A pipeline is read far more than it is written. Consistent in style, easy to follow, cleaned up continuously.
- **Avoid over-engineering.** Do not implement for a future "what if". A script or template does what the pipeline needs today, nothing more.
- **Built-in over engineered.** A built-in Azure DevOps task is preferred over a custom script or template that does the same thing.
- **Composition over repetition.** A pipeline is built from small reusable building blocks: entry points use stages or jobs, stages use jobs, jobs use steps, steps call scripts. Business logic lives in exactly one place.
- **Logic in scripts, not in YAML.** YAML describes *what* runs and *when*. Operational logic (loops, Azure CLI calls, conditions on resources) belongs in a script that can be read, tested and run on its own.
- **Environment-agnostic by default.** The same templates deploy every environment. Differences between environments are variables, never copies of a template.
- **Build once, deploy the artifact.** An application is built once per run. Deploy jobs only consume the published artifact or image, they never rebuild.
- **No secrets in source control.** Ever.

## Folder structure

All Azure DevOps pipeline assets live in the `pipeline/` folder at the root of the repository. Nothing pipeline-related lives anywhere else, with the exception of tool configuration files (see Tool configuration).

```
pipeline/
├── stages/                 (optional)
│   └── <purpose>.yaml
├── jobs/
│   └── <tool>-<ci|cd>-<action>.yaml
├── steps/
│   └── <tool>-<action>[-<object>].yaml
├── scripts/
│   └── <tool>-<action>[-<object>].sh
├── vars/
│   ├── environment.yaml
│   └── global.yaml
├── README.md
├── cd.yaml
├── [<deployable>-]pull-request.yaml
└── dependabot.yaml
```

| Folder | Contains | Used by |
|--------|----------|---------|
| `pipeline/` (root) | Entry-point pipelines and `README.md` | Azure DevOps |
| `stages/` | Optional reusable stage templates | Entry points |
| `jobs/` | Reusable job templates | Entry points and stages |
| `steps/` | Reusable step templates | Jobs |
| `scripts/` | Shell scripts and helper utilities | Steps |
| `vars/` | Shared variable templates | Entry points, stages and jobs |

## Entry points

- **The root of `pipeline/` holds only entry points and the `README.md`.** Entry points are the files an Azure DevOps pipeline is registered against. This is where a pipeline starts.
- **One entry point per purpose.** The table shows the common set. A repository may have more entry points, for example a separate CD pipeline per deployable with its own release cadence.
  | Entry point | Purpose |
  |-------------|---------|
  | `cd.yaml` | Build and deploy on a push to an environment branch, or on a push to `main` with tag promotion |
  | `release.yaml` | With tag promotion: build a release tag once and promote that artifact through the later environments |
  | `pull-request.yaml` | Validation of a pull request before merge, or one `<deployable>-pull-request.yaml` per part when validation is split by path |
  | `dependabot.yaml` | Scheduled run that opens pull requests for dependency and tool version updates |
- **Entry points orchestrate, they do not implement.** They define triggers, stages, variables and which job templates run. Steps and scripts are never inlined in an entry point.
- **Pull requests build and test at minimum.** Every pull request that changes deployable code builds that code and runs all its tests, and that pipeline is a required check on the target branch.
- **Pull request validation may be split by path.** When a repository holds parts that are validated separately (for example application code and infrastructure code), each part has its own pull request entry point, `<deployable>-pull-request.yaml`, registered as its own build validation policy with a path filter. The path filter lives on the branch policy, because Azure Repos ignores the `pr:` trigger in YAML. Every filter includes `/pipeline/*`, so a change to the pipelines is validated by all of them.
- **Dependency updates go through the normal pull request flow.** Pull requests opened by `dependabot.yaml` are validated by the pull request pipelines like any other change.
- **Deployments only trigger on code changes.** CD entry points use path filters so that changes to folders that do not hold deployable code do not trigger a deployment.

## Stages

`pipeline/stages/` contains optional reusable stage templates. A stage template holds the stages that more than one entry point runs.

- **Optional.** The folder only exists when there is duplication to remove. A repository without shared stages has no `stages/` folder.
- **Only for real duplication.** When several entry points run the same stages, those stages are extracted into a stage template to keep the entry points DRY. A stage used by one entry point stays in that entry point.
- **Stages orchestrate jobs.** A stage template only calls job templates. Steps and scripts are never inlined in a stage.
- **Differences through parameters.** What differs between the calling entry points (for example whether an artifact is published) is a parameter, not a copy of the stage.

## Jobs

`pipeline/jobs/` contains reusable job templates, each one complete workflow.

- **One workflow per job.** A job covers one complete workflow, such as validating, planning or applying infrastructure, or building or deploying an application.
- **Environment-agnostic where possible.** The environment comes in through variables, not through a hardcoded name or a copied job.
- **All inputs are parameterised.** A job does not reach for values it was not handed, other than the shared variables from `vars/`.
- **Shared logic becomes a step template.**
- **Built-in tasks may be used directly in a job.** A built-in task (checkout, download, publish artifact, extract files) does not need its own step template. A step template is for reusable combinations or for calling a script.
- **Deployments use `deployment` jobs** bound to an Azure DevOps environment, so approvals and checks are enforced by the environment.
- **Build and deploy are separate stages.** The build stage runs a normal job that produces and publishes the artifact: an application image (`containers`), or a plan for infrastructure. The deploy stage runs a deployment job. Environment approvals are checked when a stage starts, so the approval sits between build and deploy and the reviewer can inspect the result before approving.
- **Independent build jobs run in parallel.** Build and test jobs that do not need each other's output have no `dependsOn` between them, also in a run that publishes. When a deployable combines the output of several build jobs (for example a host package that bundles the web app), a separate packaging job depends on them, builds the deployables and publishes them.
- **A deploy job deploys only the published artifact.** It checks out the repository explicitly, but only to use the pipeline scripts.
- **(ISO27001) Infrastructure is applied from the pipeline, never from a workstation.**

## Steps

`pipeline/steps/` contains reusable step templates. Steps are the smaller building blocks that are combined into jobs.

- **Single responsibility.** A step template does one thing, such as installing a tool or opening network access.
- **Reusable across jobs.** A step that only makes sense in one job is a sign the responsibility is cut wrong.
- **Steps call scripts** for anything beyond a single command or a built-in Azure DevOps task.
- **A temporary change and its undo share one template.** The template takes an action parameter (`add`/`remove`) and sets `always()` on the undo action itself, so a caller cannot forget it.
- **Scripts are referenced through built-in variables** (`$(Build.SourcesDirectory)`). A path or value that is needed more than once and requires logic is defined once in `vars/global.yaml`.

## Scripts

`pipeline/scripts/` contains reusable shell scripts and helper utilities invoked by steps.

- **Bash on Linux.** Scripts are Bash and run on Linux agents. PowerShell (`pwsh`) is only used when the project itself must be built on Windows.
- **Idempotent through the tool.** Running a script twice has the same result as once. Use commands that are already safe to repeat, with no existence check before every action.
- **No unrequested extras.** No waits, retries or input validation unless a real failure required them.
- **Self-contained.** A script does not depend on state left behind by another script, and can be run locally given the same inputs.
- **Inputs are environment variables.**
- **Fixed script start.** Line 1 is the shebang (`#!/usr/bin/env bash`), line 2 is `set -euo pipefail` (PowerShell: `$ErrorActionPreference = 'Stop'`), then the header comment.

## Variables

`pipeline/vars/` contains shared variable templates.

- **Organised by scope.** `global.yaml` holds variables shared across all environments, `environment.yaml` selects the environment-specific values.
- **`environment.yaml` selects a variable group per environment,** by branch with environment branches, or by the stage's environment parameter with tag promotion. Environment-specific values live in the variable group, not in the YAML file.
- **Plain variables in `environment.yaml` only for compile-time values.** Values that must be known before the run starts, such as the service connection and the Azure DevOps environment name, may be plain variables, because variable group values are only resolved at runtime.
- **Secrets are stored in Azure Key Vault.** A pipeline reads them from Key Vault, directly or through a variable group linked to Key Vault, never from a plain variable.
- **Variable groups hold non-secret configuration.** Environment-specific settings that are not secret may live as plain values in a variable group.
- **Secrets go to tools through environment variables, never as command-line arguments.** Arguments are visible in process lists and logs.

## Environments and branches

- **A project uses one of the two promotion patterns of `release`:** environment branches or tag promotion. The branch names and the number of environments are decided per project.
- **With tag promotion, `release.yaml` builds once** on a release tag and deploys the artifact to each later environment in its own deploy stage.
- **With environment branches, a later environment's deploy stage builds nothing.** It reads the image running in the previous environment, checks it as `release` requires, and downloads the migration artifact from the build run named in the image's labels (the Download Pipeline Artifact task with that run's id).
- **The trigger-to-environment mapping lives in one place**, in `vars/environment.yaml`, not scattered over jobs and steps. Branch and tag names may otherwise only appear in entry point triggers and tool configuration, which cannot use variables.
- **Environment branches and release tags are protected.** An environment branch only changes through a pull request, never by a direct push. A release tag is created only by the people allowed to release, and repository permissions stop anyone from moving or deleting it.
- **Production is gated.** The production environment has a manual approval check, and both the environment and its service connection have a branch control check that admits only the production branch, or with tag promotion only release tags (`refs/tags/v*`).
- **Deployments to an environment run one at a time.** The environment has an exclusive lock, so two runs never deploy to it at the same time.

## Tool configuration

- **Tool configuration lives where the tool requires it.** Configuration files for tools, scanners and extensions are the one exception to the folder structure, but only when the tool cannot be pointed at `pipeline/`.

## Tasks and extensions

- **Marketplace extensions only from trusted publishers.** Microsoft, Microsoft DevLabs and other established companies. Extensions from individual hobbyist publishers are not allowed.
- **Extensions must be widely used.** Extensions with a low install count or a low review rating are not allowed.
- **Extensions must be maintained.** An extension that has not been updated in the last year is not allowed, and an extension already in use that passes that mark is replaced.
- **Extensions follow the Software Governance rules** (`compliance`). An extension is never installed or worked around on our own initiative.
  1. **Check the organisation.** Verify whether the extension is already installed in the Azure DevOps organisation. If it is, it may be used.
  2. **Flag it.** If it is not installed, flag it to the developer. The pipeline does not use the extension until it is installed.
  3. **Request it.** When the developer needs the extension, they request its installation from the IT director, as the Software Governance rules require (`compliance`).
  4. **Review.** The IT director, or whom they delegate, and where needed an architect, review the request against the rules above and install or reject the extension.
  5. **Follow up.** After the decision, the solution is discussed further: with an approved extension the work continues, with a rejected one an alternative is chosen.

## Versions

- **Tool versions and the agent image are pinned** to an explicit version in `vars/global.yaml`. Never `latest`.
- **Task versions are pinned to a major version.** A major version upgrade is a deliberate change.
- **Version bumps come from `dependabot.yaml`** or a deliberate pull request, never as a side effect of other work.

## Caching

- **Cache downloaded dependencies, never build output.** Everything a run downloads to build or test, such as packages, browsers for tests and infrastructure providers, is cached with the built-in cache task. Compiled output, test results and deployables are never cached; what another job needs is published as an artifact.
- **The cache key is the lockfile.** The key is the tool, the agent OS and the tool's committed lockfile, or its pinned version from `vars/global.yaml`. A changed lockfile or version starts a new cache.
- **A cache never replaces verification.** Restores still run in the tool's locked mode, and downloads are still checked against the lockfile hashes, so a cache hit installs exactly what a miss would. Tool binaries that a script downloads and checks against a checksum are not cached; the check is the point.
- **Scripts do not know about the cache.** The cache path is handed to the tool through the tool's own environment variable, defined once in `vars/global.yaml`. The cache step sits in the job, before the step that restores. A script runs the same locally, without a cache.

## Agents and access

- **Microsoft-hosted Linux agents are the default.** A project may choose self-hosted agents when its network or tooling requires it.
- **Azure service connections use Workload Identity Federation.** No client secrets or certificates on a service connection.
- **(ISO27001) A service connection is authorised per pipeline**, never for all pipelines.
- **Workarounds are always undone.** A temporary change such as a whitelisted agent IP must not outlive the run. The run can fail between adding and removing it, so the removing step runs `always()`; the adding step doesn't need it.

## Database migrations

- **Language-specific.** How database migrations are produced and applied differs per project and is described in the coding standards of that language. The rules below hold for all of them.
- **Migrations are built once, with the application.** The migration artifact (a bundle or script) is built in the build stage from the same commit as the application and published with it, so the schema the code expects and the migrations that produce it cannot drift apart. The deploy stage only consumes it.
- **The pipeline applies migrations, never the application.** A dedicated step in the deploy job applies them to the target database before the application is released, and a non-zero exit fails the stage and halts the deploy. An app that migrates on startup races its own instances and couples schema changes to traffic.
- **Migrations are backward compatible, so the release is not tightly coupled to the migration.** The previous release keeps serving while the migration runs and until the new one takes over, as the database standards (`database`) require. A failed migration leaves the old release running.
- **Migrations authenticate with the pipeline's workload identity** (Entra ID). No SQL passwords in the pipeline, ever.
- **Migrations run as part of the environment's deployment**, under the environment's exclusive lock, so two runs never migrate and deploy the same environment at the same time.
- **A pipeline never drops or resets a database.**

## Generated documents

- **Documents that can be generated from the code are generated by the build.** This covers API contracts (OpenAPI), schema exports and similar derived documents. They are produced from the built code on every build and published with the build artifact, so consumers use them without running the service. How a language or tool generates them is described in the standards of that language or domain.
- **A generated document is never committed.** The pipeline always regenerates it, so a hand-written or stale copy in the repository cannot become the source of truth.

## Naming and conventions

- **One word per concept.** No puns and no synonyms. Pick one verb (`add`/`remove`, `install`, `plan`/`apply`) and keep it consistent across file names, parameters and display names.
- **Files are `kebab-case`** and the extension is `.yaml` (not `.yml`), except where a tool requires otherwise (see Tool configuration).
- **Additional entry points are named `<deployable>-<trigger>.yaml`.**
- **Jobs are named `<tool>-<ci|cd>-<action>`.** `ci` for jobs that validate or build, `cd` for jobs that plan or deploy. A job outside these flows, such as a scheduled job, is named `<tool>-<action>`.
- **Stage templates are named `<purpose>.yaml`** after the part of the flow they cover.
- **Steps and scripts are named `<tool>-<action>[-<object>]`.** `<tool>` is the tool or platform acted on, `<action>` the verb, `<object>` what it acts on when the action alone is not clear. A script has the same name as the step template that calls it.
- **Azure DevOps names are `<purpose>-<environment>`.** This applies to variable groups, Azure DevOps environments and service connections.
- **Parameters are `PascalCase`** and always declare a `type`.
- **Variables are `camelCase`.**
- **Stage and job identifiers are `PascalCase`.** Azure DevOps does not allow hyphens in identifiers; the `displayName` carries the readable text.
- **Script inputs are `UPPER_SNAKE_CASE`** environment variables.
- **Template references are relative** to the referencing file.
- **Every step has a `displayName`** that says what it does, so the pipeline run reads like a log of actions.

## README

Every `pipeline/` folder has a `README.md`. It gives a brief overview for developers working on the pipelines. It holds notes and considerations, not an explanation of every file.

- **File structure.** A tree of the `pipeline/` folder.
- **Pipelines.** A table per trigger type (branch, pull request, schedule) with pipeline, branches or schedule, and whether it deploys.
- **Considerations.** Short notes on design decisions a developer needs to know, such as why templates take parameters or how environment-specific variables are selected.
- **Security.** A table of the variable groups: name, description, and which pipelines use them.
- **Keep it brief.** The README is the place for the "why" that would otherwise end up as comments in YAML. It is not a design document.

## Comments

- **No comments by default.** File names, parameter names and display names should communicate what happens. A comment needed to explain what YAML or a script does calls for a clearer name or a split, not a comment.
- **Script header is the exception.** Every script has a header comment, directly after the fixed script start, describing what it does and its input environment variables.
- **TODO:** A TODO comment is only for planned future work and requires a reference to a work item.

## Security

- **(ISO27001) Audit logging is on for the DevOps platform**, including non-production environments, and pipeline events are monitored for deviations.
