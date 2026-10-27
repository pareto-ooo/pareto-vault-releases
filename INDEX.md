## architecture
- `architecture/business-capabilities.md` — how projects use the Business Capability Model: record impacted capabilities, reference BCM names, raise change requests, capability naming; used when defining a project's scope or proposing a capability
- `architecture/default-stack.md` — default stack for custom-built apps: Vue on @pareto4/ui-vue, C#/.NET on App Service, hosted services, Azure SQL or Cosmos DB, Service Bus, Application Insights; used when starting or analysing an application
- `architecture/deviations.md` — [cross-cutting] how to deviate from a principle or standard: record it, who signs off, security approvers, which deviations block the affected work; used when any choice goes against the vault
- `architecture/integration.md` — in-house integration rules: source of truth, API exposure, integration platform, sync or async, decoupling, business keys, failures, monitoring, documentation; used when planning any integration between systems
- `architecture/principles.md` — generic enterprise architecture principles: PACE application layers, business, information and technical principles, architecture models; used when starting a project or making any design or technology decision
- `architecture/saas.md` — buying SaaS applications: SaaS before PaaS before IaaS, selection, Entra ID sign-on, data ownership, customer mail, SLA and compliance; used when the reuse-buy-build choice leads to buying an application
- `architecture/technology-choices.md` — choosing services and technologies: reuse before buy before build, Microsoft first, managed over self-hosted, selection criteria, LTS and versions; used when planning which services a workload is built on

## business
- `business/company.md` — Pareto company profile, mission, vision, values, strategic goals and objectives; only for internal Pareto projects, ignored for client projects; used when judging where an internal project proposal fits

## code-standards
- `code-standards/coding-standards-all-languages.md` — ground-truth principles for all code: philosophy, change discipline, layered architecture, naming, comments, code smells, errors and testing; used when planning or writing code in any language
- `code-standards/coding-standards-csharp.md` — C#/.NET specifics: stack, file layout, async, DI, logging, mappers, time and money, EF Core entities, repositories and migrations; used when planning or writing C# code or EF Core data access
- `code-standards/coding-standards-sql.md` — SQL style: uppercase keywords, PascalCase aliases, explicit columns, leading commas, joins, no DISTINCT or UNION for grouping, CTEs over subqueries; used when writing or reviewing any SQL query or view
- `code-standards/coding-standards-vuejs.md` — Vue 3 + TypeScript: script setup, @pareto4/ui-vue, Pinia, routing, screens, API access, i18n, accessibility (WCAG 2.2 AA where law requires), done; used when planning or writing Vue frontend code
- `code-standards/source-control.md` — Conventional Commits, branch naming, pull request and squash-merge rules, SemVer releases as tags on main; used when planning commits, branches, pull requests or a version bump
- `code-standards/testing-strategy.md` — test levels beyond the unit test: integration, contract (OpenAPI, event schemas), end-to-end, performance, acceptance, test environments and data; used when planning tests or a QA approach

## infrastructure
- `infrastructure/api.md` — HTTP API rules: auth per operation, DTOs, routes and versioning, enum names, problem+json errors, paging, idempotency, rate limits, operator endpoints, OpenAPI; used when planning or reviewing HTTP endpoints
- `infrastructure/app-service.md` — apps on App Service for Linux: plan per app per environment, image pull, staging slot and swap, health check, Always On, singleton work, settings, hardening; used when planning App Service resources or releases
- `infrastructure/containers.md` — container images: one per deployable, multi-stage, minimal pinned base, non-root, commit tags, scanned, registries per subscription, promotion by digest; used when planning Dockerfiles, images or registries
- `infrastructure/database.md` — language-neutral database rules: one database per app, GUIDv4 ids (v7 for large tables), keyset paging, audit columns, concurrency, outbox, migrations, Cosmos DB; used when planning schema or persistence work
- `infrastructure/governance.md` — cloud governance as infrastructure rules: tenants and subscriptions, tags, identity, network, data protection, logging, backup, cost; used when planning any Azure resource or landing-zone change
- `infrastructure/inter-service-communication.md` — choosing HTTP vs Service Bus between apps: no call by default, queries vs facts, timeouts and fallback, one event envelope, IDs only, RBAC on topics; used when planning a cross-app call or event
- `infrastructure/naming.md` — [cross-cutting] Azure resource naming convention: components, formats per resource type, security groups, restricted names, stable names; used when planning new Azure resources to decide how each is named
- `infrastructure/observability.md` — telemetry, W3C trace propagation from SPA to subscribers, sampling and ingestion caps, alerts by criticality, availability tests, dashboards; used when planning monitoring or alerts
- `infrastructure/sizing.md` — sizing and cost rules: workload profile, environment profiles, availability and redundancy by criticality, tiers, cold starts, cost estimate, performance test; used when choosing service tiers or estimating costs
- `infrastructure/terraform.md` — Terraform and Terragrunt rules: folder layout, units, configuration levels, access unit for role assignments, modules, remote state, deletion protection, applying; used when planning or writing infrastructure code

## pipeline
- `pipeline/azure-devops.md` — Azure DevOps pipeline rules: folder layout, stages, jobs, steps, scripts, variables, environments, migrations and security; used when planning or writing pipeline YAML, templates or scripts
- `pipeline/github-actions.md` — GitHub Actions rules: layout, entry points, reusable workflows, scripts, OIDC and permissions, environments and promotion mechanics, migrations, security; used when planning or writing GitHub Actions workflows
- `pipeline/quality-gates.md` — merge-blocking PR gates: build and tests, SonarCloud, GHAS and Trivy, substitutes without them, ZAP, thresholds, hotfix exceptions; used when deciding which pipeline checks must pass before merge
- `pipeline/release.md` — promotion (tag or environment branches, never a rebuild), one stage per approval, state checks before slot swaps, performance test before critical go-lives; used when planning deploy, promotion or release changes

## security
- `security/compliance.md` — GDPR, ISO 27001, Microsoft first, regions, backups, classification levels, one-person teams, audit trail, threat models, evidence; used when planning personal data, dependencies, trust boundaries, backups or audits
- `security/authentication.md` — sign-in on the default stack: app registrations per environment, MSAL with PKCE, token validation in the API, app roles, managed-identity callers, External ID; used when planning sign-in, roles or service calls
- `security/security.md` — security principles and rules for authentication, authorization, tenancy, role recovery, TLS, secrets, cryptography, Azure, input handling, logging and supply chain; used when planning security-sensitive work or a review

## root
- `local-development.md` — local development in containers: Docker Desktop, a dev container per repo, emulated services, Aspire or Compose, one start command, own Entra login, seed data; used when setting up how a project runs on a laptop
- `project-folder-layout.md` — [cross-cutting] top-level repository layout: code/, infrastructure/, pipeline/, docs/, what may live at the root, tool configuration and migrations; used when creating a repository or deciding where a new file goes
- `project-records.md` — [cross-cutting] where a project records what the vault asks: project definition, client substitutions, build readiness, decision records, architecture overview, diagrams; used when any rule asks a project to record something
- `prototype.md` — throwaway discovery prototypes on @pareto4/ui-vue: output, language, styling, layout, interaction level, sample data; used when planning discovery-phase mockups, never for the real build
