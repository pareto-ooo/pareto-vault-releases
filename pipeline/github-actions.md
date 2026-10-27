---
id: github-actions
scope: scoped
severity: warning
audience: all
applies-to:
  - "**/*.yml"
  - "**/*.yaml"
schemaVersion: 1
---

# Pipeline standards: GitHub Actions

How we build GitHub Actions workflows. These are the ground truth rules for all workflows, actions, scripts and variables in a repository hosted on GitHub. It covers infrastructure, application build and deploy, and database migrations. A repository on Azure DevOps follows the Azure DevOps pipeline standards instead; the quality gates and release standards hold for both.

## Pipeline philosophy

- **Readability first.** A workflow is read far more than it is written. Consistent in style, easy to follow, cleaned up continuously.
- **Avoid over-engineering.** Do not implement for a future "what if". A script or action does what the workflow needs today, nothing more.
- **Built-in over engineered.** A GitHub-owned action (`actions/*`) or a verified action from the tool's own publisher is preferred over a custom script or action that does the same thing.
- **Composition over repetition.** A workflow is built from small reusable building blocks: entry points call reusable workflows, reusable workflows run jobs, jobs use composite actions, composite actions call scripts. Business logic lives in exactly one place.
- **Logic in scripts, not in YAML.** YAML describes *what* runs and *when*. Operational logic (loops, Azure CLI calls, conditions on resources) belongs in a script that can be read, tested and run on its own.
- **Environment-agnostic by default.** The same workflows deploy every environment. Differences between environments are variables, never copies of a workflow.
- **Build once, deploy the artifact.** An application is built once per run. Deploy jobs only consume the published artifact or image, they never rebuild.
- **No secrets in source control.** Ever.

## Folder structure

GitHub only runs workflows from `.github/workflows/`, and that folder cannot have subfolders. Workflows and composite actions therefore live under `.github/`; everything else pipeline-related lives in the `pipeline/` folder at the root of the repository, with the exception of tool configuration files (see Tool configuration).

```
.github/
├── workflows/
│   ├── cd.yaml
│   ├── [<deployable>-]pull-request.yaml
│   └── <tool>-<ci|cd>-<action>.yaml        (reusable workflows)
├── actions/
│   └── <tool>-<action>[-<object>]/
│       └── action.yaml
└── dependabot.yml
pipeline/
├── scripts/
│   └── <tool>-<action>[-<object>].sh
└── README.md
```

| Folder | Contains | Used by |
|--------|----------|---------|
| `.github/workflows/` | Entry-point workflows and reusable workflows | GitHub Actions |
| `.github/actions/` | Composite actions, one folder each | Jobs |
| `pipeline/scripts/` | Shell scripts and helper utilities | Composite actions |
| `pipeline/` (root) | `README.md` | Developers |

## Entry points

- **An entry point is a workflow with a trigger** (`push`, `pull_request`, `schedule`, `workflow_dispatch`). This is where a run starts.
- **One entry point per purpose.** The table shows the common set. A repository may have more entry points, for example a separate CD workflow per deployable with its own release cadence.
  | Entry point | Purpose |
  |-------------|---------|
  | `cd.yaml` | Build and deploy on a push to an environment branch, or on a push to `main` with tag promotion |
  | `release.yaml` | With tag promotion: build a release tag once and promote that artifact through the later environments |
  | `pull-request.yaml` | Validation of a pull request before merge, or one `<deployable>-pull-request.yaml` per part when validation is split by path |
- **Entry points orchestrate, they do not implement.** They define triggers, permissions, concurrency and which reusable workflows or jobs run. Multi-line scripts are never inlined in an entry point.
- **Pull requests build and test at minimum.** Every pull request that changes deployable code builds that code and runs all its tests, and that workflow's checks are required on the target branch.
- **Pull request validation may be split by path.** When a repository holds parts that are validated separately (for example application code and infrastructure code), each part has its own pull request entry point, `<deployable>-pull-request.yaml`, that validates only the paths of that part. Every part's paths include `.github/**` and `pipeline/**`, so a change to the workflows is validated by all of them.
- **A required check reports on every pull request.** A workflow that a `paths` filter skips reports nothing, and a required check that never reports blocks the merge. A split workflow whose checks are required therefore runs on every pull request, and a first job decides from the changed files whether the other jobs run. A job skipped by its `if` condition counts as passing.
- **Dependency updates go through the normal pull request flow.** Dependabot is configured in `.github/dependabot.yml`; the pull requests it opens are validated by the pull request workflows like any other change.
- **Deployments only trigger on code changes.** CD entry points use `paths` filters so that changes to folders that do not hold deployable code do not trigger a deployment.

## Reusable workflows

Reusable workflows (`on: workflow_call`) take the place of stage and job templates. They live in `.github/workflows/` next to the entry points.

- **Only for real duplication.** When several entry points run the same jobs, those jobs are extracted into a reusable workflow. A job used by one entry point stays in that entry point.
- **A reusable workflow has no other trigger.** It declares only `workflow_call`, so it never runs on its own.
- **All inputs are declared.** A reusable workflow declares its `inputs` with a `type` and its `secrets` by name; it never uses `secrets: inherit`.
- **Differences through inputs.** What differs between the calling entry points (for example whether an artifact is published) is an input, not a copy of the workflow.

## Jobs

- **One workflow per job.** A job covers one complete workflow, such as validating, planning or applying infrastructure, or building or deploying an application.
- **Environment-agnostic where possible.** The environment comes in through the job's `environment` and its variables, not through a hardcoded name or a copied job.
- **Shared logic becomes a composite action.**
- **Actions may be used directly in a job.** A GitHub-owned action (checkout, upload or download artifact, setup of a tool) does not need its own composite action. A composite action is for reusable combinations or for calling a script.
- **Deployments run in a job bound to a GitHub environment**, so protection rules (required reviewers, deployment branches) are enforced by the environment.
- **Build and deploy are separate jobs.** The build job produces and publishes the artifact: an application image (`containers`), or a plan for infrastructure. The deploy job declares `needs` on it and runs in the environment. Protection rules are checked when the deploy job starts, so the approval sits between build and deploy and the reviewer can inspect the result before approving.
- **Independent build jobs run in parallel.** Build and test jobs that do not need each other's output have no `needs` between them, also in a run that publishes. When a deployable combines the output of several build jobs, a separate packaging job needs them, builds the deployables and publishes them.
- **A deploy job deploys only the published artifact.** It checks out the repository explicitly, but only to use the pipeline scripts.
- **(ISO27001) Infrastructure is applied from the pipeline, never from a workstation.**

## Composite actions

`.github/actions/` contains composite actions. They are the smaller building blocks that are combined into jobs.

- **Single responsibility.** A composite action does one thing, such as installing a tool or opening network access.
- **Reusable across jobs.** An action that only makes sense in one job is a sign the responsibility is cut wrong.
- **Actions call scripts** for anything beyond a single command or a published action.
- **A temporary change and its undo are two steps of the same job.** The undo step runs `if: always()`, so a failure between the two cannot leave the change behind.
- **Scripts are referenced through the workspace** (`${{ github.workspace }}/pipeline/scripts/...`).

## Scripts

`pipeline/scripts/` contains reusable shell scripts and helper utilities invoked by composite actions.

- **Bash on Linux.** Scripts are Bash and run on Linux runners. PowerShell (`pwsh`) is only used when the project itself must be built on Windows.
- **Idempotent through the tool.** Running a script twice has the same result as once. Use commands that are already safe to repeat, with no existence check before every action.
- **No unrequested extras.** No waits, retries or input validation unless a real failure required them.
- **Self-contained.** A script does not depend on state left behind by another script, and can be run locally given the same inputs.
- **Inputs are environment variables.**
- **Fixed script start.** Line 1 is the shebang (`#!/usr/bin/env bash`), line 2 is `set -euo pipefail` (PowerShell: `$ErrorActionPreference = 'Stop'`), then the header comment.

## Variables and secrets

- **Shared values are workflow `env` or repository variables.** A value used by one workflow is set in its `env`; a value shared by several workflows is a repository (or organisation) variable, read as `vars.<NAME>`.
- **Environment-specific values are environment variables** of the GitHub environment, never a branch condition inside a job.
- **Azure access uses OpenID Connect.** Jobs sign in to Azure with a federated credential on an Entra ID app registration; no client secrets or certificates are stored in GitHub. The federated credential's subject names the environment (or `pull_request` for plan-only jobs), so a job outside that environment cannot get a token.
- **Secrets are stored in Azure Key Vault.** A workflow reads them from Key Vault after signing in. A GitHub secret is only for what cannot live in Key Vault, such as a token for another GitHub repository, and is scoped to the environment that needs it.
- **Secrets go to tools through environment variables, never as command-line arguments.** Arguments are visible in process lists and logs.

## Permissions

- **Least privilege by default.** Every workflow sets `permissions: contents: read` at the top. A job that needs more (`id-token: write` for OIDC, `pull-requests: write` to comment) declares it on that job only.
- **No `pull_request_target` with a checkout of the pull request's code.** It runs untrusted code with the base repository's permissions and secrets.

## Environments and branches

- **A project uses one of the two promotion patterns of `release`:** environment branches or tag promotion. The branch names and the number of environments are decided per project.
- **With tag promotion, `release.yaml` builds once** and deploys the artifact to each later environment in its own environment-bound job.
- **With environment branches, a later environment's deploy job builds nothing.** It reads the image running in the previous environment, checks it as `release` requires, and downloads the migration artifact from the build run named in the image's labels (`actions/download-artifact` with `run-id`).
- **The trigger-to-environment mapping lives in one place**, in the entry point's trigger and its `environment` selection, not scattered over jobs and steps.
- **Environment branches and release tags are protected by rulesets.** An environment branch only changes through a pull request, never by a direct push. A release tag is created only by the people allowed to release, and is never moved or deleted.
- **Production is gated.** The production environment has required reviewers and a deployment policy that only admits the production branch, or with tag promotion only release tags.
- **Deployments to an environment run one at a time.** Every deploy job sets `concurrency` to a group named after the environment, with `cancel-in-progress: false`, so two runs never deploy to it at the same time.

## Tool configuration

- **Tool configuration lives where the tool requires it.** Configuration files for tools, scanners and Dependabot are the one exception to the folder structure, but only when the tool cannot be pointed at `pipeline/`.

## Actions from the marketplace

- **Actions only from trusted publishers.** GitHub (`actions/*`, `github/*`), the tool's own vendor (for example `azure/*`, `hashicorp/*`) and other verified creators. Actions from individual hobbyist publishers are not allowed.
- **Actions must be widely used and maintained.** An action with low usage, or that has not been updated in the last year, is not allowed, and an action already in use that passes that mark is replaced.
- **The organisation's allowed-actions policy is the enforcement.** An action outside the policy is never worked around on our own initiative.
  1. **Check the policy.** Verify whether the action is already allowed in the organisation. If it is, it may be used.
  2. **Flag it.** If it is not allowed, flag it to the developer. The workflow does not use the action until it is allowed.
  3. **Request it.** When the developer needs the action, they request it through the Workspace Software Governance process.
  4. **Review.** IT Ops, and where needed an architect, review the request against the rules above and allow or reject the action.
  5. **Follow up.** After the decision, the solution is discussed further: with an approved action the work continues, with a rejected one an alternative is chosen.

## Versions

- **Actions are pinned to a full commit SHA**, with the version in a comment (`uses: actions/checkout@<sha> # v5.0.0`). A tag can be moved; a SHA cannot.
- **Tool versions and the runner image are pinned** to an explicit version. Never `latest`, and never `ubuntu-latest` where the image matters to the result.
- **Version bumps come from Dependabot** (with the `github-actions` ecosystem enabled) or a deliberate pull request, never as a side effect of other work.

## Caching

- **Cache downloaded dependencies, never build output.** Everything a run downloads to build or test, such as packages, browsers for tests and infrastructure providers, is cached with `actions/cache` or the setup action's built-in cache. Compiled output, test results and deployables are never cached; what another job needs is uploaded as an artifact.
- **The cache key is the lockfile.** The key is the tool, the runner OS and the tool's committed lockfile, or its pinned version. A changed lockfile or version starts a new cache.
- **A cache never replaces verification.** Restores still run in the tool's locked mode, and downloads are still checked against the lockfile hashes, so a cache hit installs exactly what a miss would. Tool binaries that a script downloads and checks against a checksum are not cached; the check is the point.
- **Scripts do not know about the cache.** The cache path is handed to the tool through the tool's own environment variable. The cache step sits in the job, before the step that restores. A script runs the same locally, without a cache.

## Runners and access

- **GitHub-hosted Linux runners are the default.** A project may choose self-hosted runners when its network or tooling requires it. A self-hosted runner never serves a public repository.
- **Workarounds are always undone.** A temporary change such as a whitelisted runner IP must not outlive the run. The run can fail between adding and removing it, so the removing step runs `if: always()`; the adding step doesn't need it.

## Database migrations

- **Language-specific.** How database migrations are produced and applied differs per project and is described in the coding standards of that language. The rules below hold for all of them.
- **Migrations are built once, with the application.** The migration artifact (a bundle or script) is built in the build job from the same commit as the application and published with it, so the schema the code expects and the migrations that produce it cannot drift apart. The deploy job only consumes it.
- **The pipeline applies migrations, never the application.** A dedicated step in the deploy job applies them to the target database before the application is released, and a non-zero exit fails the job and halts the deploy. An app that migrates on startup races its own instances and couples schema changes to traffic.
- **Migrations are backward compatible, so the release is not tightly coupled to the migration.** The previous release keeps serving while the migration runs and until the new one takes over, as the database standards (`database`) require. A failed migration leaves the old release running.
- **Migrations authenticate with the workflow's federated identity** (Entra ID). No SQL passwords in the pipeline, ever.
- **Migrations run as part of the environment's deployment**, in the environment's concurrency group, so two runs never migrate and deploy the same environment at the same time.
- **A pipeline never drops or resets a database.**

## Generated documents

- **Documents that can be generated from the code are generated by the build.** This covers API contracts (OpenAPI), schema exports and similar derived documents. They are produced from the built code on every build and published with the build artifact, so consumers use them without running the service. How a language or tool generates them is described in the standards of that language or domain.
- **A generated document is never committed.** The pipeline always regenerates it, so a hand-written or stale copy in the repository cannot become the source of truth.

## Naming and conventions

- **One word per concept.** No puns and no synonyms. Pick one verb (`add`/`remove`, `install`, `plan`/`apply`) and keep it consistent across file names, inputs and step names.
- **Files are `kebab-case`** and the extension is `.yaml` (not `.yml`), except where a tool requires otherwise (`.github/dependabot.yml`).
- **Additional entry points are named `<deployable>-<trigger>.yaml`.**
- **Reusable workflows are named `<tool>-<ci|cd>-<action>.yaml`.** `ci` for workflows that validate or build, `cd` for workflows that plan or deploy.
- **Composite actions and scripts are named `<tool>-<action>[-<object>]`.** `<tool>` is the tool or platform acted on, `<action>` the verb, `<object>` what it acts on when the action alone is not clear. A script has the same name as the action that calls it.
- **GitHub environments are named after the environment** (`acc`, `prd`), the same names the infrastructure uses.
- **Inputs are `kebab-case`** and always declare a `type`.
- **Job identifiers are `kebab-case`;** the `name` carries the readable text.
- **Script inputs and environment variables are `UPPER_SNAKE_CASE`.**
- **Every step has a `name`** that says what it does, so the run reads like a log of actions.

## README

Every `pipeline/` folder has a `README.md`. It gives a brief overview for developers working on the workflows. It holds notes and considerations, not an explanation of every file.

- **File structure.** A tree of `.github/` and `pipeline/`.
- **Workflows.** A table per trigger type (branch, pull request, schedule) with workflow, branches or schedule, and whether it deploys.
- **Considerations.** Short notes on design decisions a developer needs to know, such as why a workflow takes inputs or how environment-specific values are selected.
- **Security.** A table of the GitHub environments and the federated credentials: name, subject, and which workflows use them.
- **Keep it brief.** The README is the place for the "why" that would otherwise end up as comments in YAML. It is not a design document.

## Comments

- **No comments by default.** File names, input names and step names should communicate what happens. A comment needed to explain what YAML or a script does calls for a clearer name or a split, not a comment.
- **Script header is the exception.** Every script has a header comment, directly after the fixed script start, describing what it does and its input environment variables.
- **TODO:** A TODO comment is only for planned future work and requires a reference to an issue.

## Security

- **(ISO27001)(Cloud Governance) The organisation audit log is retained and reviewed**, including events for non-production environments, and workflow events are monitored for deviations.
