---
id: local-development
scope: global
severity: warning
schemaVersion: 1
---

# Local development

How a developer runs an application on their laptop. The application, its toolchain and its backing services run in containers, so a laptop needs almost nothing installed, and the local setup resembles the infrastructure in the cloud. These rules apply to every application project.

## Local development philosophy

- **Nothing installed but the essentials.** A laptop has a container engine, Git and an editor. SDKs, package managers, CLIs and backing services run in containers, so a new developer is productive without an installation guide.
- **Close to the cloud.** Locally the application uses the same kinds of services as in Azure, at the same major versions, so a difference between laptop and cloud shows up early instead of after a deployment.
- **Throwaway by design.** The local environment can be deleted and rebuilt at any time. Nothing on the laptop holds state that matters.
- **Editor agnostic.** Everything works from the command line, so the setup works the same from any editor, or from none. Editor integration is a convenience on top.

## Container engine

- **Docker Desktop is the container engine.**
- **On Windows, Docker Desktop runs on the WSL2 backend.**
- **Container definitions follow the open standards** (the OCI image format and the Compose specification), so they do not depend on one engine.

## Dev container

- **Every repository has a dev container** that holds the full toolchain to build, test and run the application: SDKs, package managers and CLIs.
- **The dev container definition lives where the tooling requires it,** which may be at the repository root.
- **Each repository defines its own dev container** on official public base images, extended with dev container features. There is no central Pareto dev container image.
- **Toolchain versions match the pipeline.** The SDK and runtime major versions in the dev container are the ones the pipeline builds with and the cloud runs.
- **The .NET SDK is the one exception.** A C# project may use the .NET SDK installed on the laptop, because the Aspire AppHost and Visual Studio both expect it there. All other tooling, such as Node.js, Python and CLIs, stays in the dev container, and the dev container still contains the .NET SDK for developers who do not install it.

## Backing services

- **Every backing service the application uses runs locally** as a container, or as the official emulator for Azure services that have one.
- **Local services use the same major version as in Azure.** A database, cache or broker runs locally at the major version the cloud environment runs.
- **A service without a local equivalent uses the development environment in Azure,** reached with the developer's own login. A laptop never connects to test, acceptance or production resources.

## Orchestration

- **One command starts everything.** The application and all its backing services start with a single command from the repository root, without an editor. The README documents that command.
- **A C# project uses an Aspire AppHost,** also when the application is a single service. The AppHost is a project in `code/` and starts the application, its backing services and the Aspire dashboard.
- **Other projects use a Compose file** run by Docker Compose, in `code/`.
- **The deployable image runs locally.** Besides running from source for debugging, the application can be started from the same container definition the pipeline builds, so the image is exercised before it is deployed.
- **Local orchestration never deploys.** The AppHost and the Compose file are for local development only. Deployment to Azure goes through Terraform and the Azure DevOps pipeline, never through the publish features of a local orchestrator.

## Configuration and secrets

- **The application authenticates with the developer's own Entra ID login.** Locally, the same managed identity code path uses the developer's Azure sign-in, so it reads the development Key Vault and other development resources without secrets in files.
- **Developers get access to development resources through security groups,** never through individual role assignments.
- **No real secrets in local configuration.** The well-known keys of local emulators may be in committed local configuration, because they grant access to nothing outside the laptop. Every other secret comes from the development Key Vault.

## Data

- **Seed data is part of the repository** and is loaded when the local environment starts. It is fake or generated data that covers the cases developers need.
- **Production data never goes to a laptop,** also not as a copy, an extract or a backup.
- **Local data can be thrown away.** Container volumes hold only data that the seed recreates, so deleting them never loses work.

## Adoption and documentation

- **New projects have a local environment from the start.** An existing project adds one when it gets substantial work.
- **The README describes the local environment:** the prerequisites, the command that starts it, which services run locally, and which use the development environment in Azure.
