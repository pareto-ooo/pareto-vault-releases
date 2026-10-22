---
id: default-stack
scope: global
severity: warning
schemaVersion: 1
---

# Architecture standards: Default stack

The technologies every custom-built application starts from: its frontend, backend, background work, data, messaging, mail and telemetry. These are the ground truth defaults for a new application and for analysing an existing one. They name no versions: every runtime and framework uses its latest long-term support release, or otherwise its latest supported major version. Data platform and integration workloads are out of scope and choose the technologies that fit them.

## Default stack philosophy

- **One stack, unless there is a reason.** Every application starts from these defaults, so knowledge, standards, pipelines and infrastructure modules are reused across projects instead of reinvented.
- **A choice off the default is a deviation.** It is recorded with its reason and signed off by an architect. Whether it blocks is set per element below, not by the general easy-or-hard test.
- **The defaults are a company decision.** Where a default is not a Microsoft product, as with the frontend, it is chosen deliberately and takes precedence over the company's Microsoft first policy.

## Frontend

- **Every web frontend is a Vue single-page app on the `@pareto4/ui-vue` design system,** written in TypeScript. There is no other frontend framework.
- **The frontend runs in its own App Service,** separate from the backend.
- **Another frontend framework is a blocking deviation.** It breaks the shared design system, styling and frontend standards.

## Backend

- **Backends are C# on .NET,** with ASP.NET Core for APIs.
- **Another backend language is a non-blocking deviation.** It is recorded with its reason, and the work continues.

## Hosting

- **Applications run on App Service.** Single sign-on at the edge through App Service authentication and the platform-managed certificates assume it.
- **Another Azure compute service is a non-blocking deviation.** Container Apps, for example, is recorded with its reason.
- **Deploying to compute outside Azure is a blocking deviation.**

## Background work

- **Background, scheduled and queue-driven work runs as hosted services inside the application,** in the same App Service.
- **Another Azure service for background work is a non-blocking deviation,** such as Azure Functions for work that must scale apart from the application.

## Data

- **Azure SQL Database is the default database.**
- **Cosmos DB is an equal choice for document-shaped data,** without a deviation.
- **Another database in a deployed environment is a blocking deviation** from the Microsoft first policy.
- **Files are stored in Blob Storage,** never in the database.

## Messaging

- **Applications communicate asynchronously through Service Bus** topics and queues.
- **Another Azure messaging service is a non-blocking deviation,** such as Event Grid for events raised by Azure resources.
- **A messaging service outside Azure in a deployed environment is a blocking deviation.**

## Mail

- **Mail to customers goes through the central mail delivery service,** over SMTP or its API. It is the one trusted, allow-listed sender for our domain. **TODO:** name the mail delivery service.
- **Internal mail goes through Microsoft Exchange,** with a service account per application.
- **Another mail provider in a deployed environment is a blocking deviation.** It changes who can send mail in our name.

## Telemetry

- **Applications send telemetry to a workspace-based Application Insights resource,** instrumented through the Azure Monitor OpenTelemetry distribution. The same OpenTelemetry instrumentation feeds the Aspire dashboard during local development.
