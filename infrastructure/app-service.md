---
id: app-service
scope: scoped
severity: warning
audience: all
applies-to:
  - "**/*.tf"
  - "**/*.hcl"
schemaVersion: 1
---

# Infrastructure standards: App Service

How applications on the default stack run on Azure App Service: plans, packaging, slots, health checks, configuration and scaling. These are the ground truth rules for every App Service and App Service plan in infrastructure code. Naming, tags, sizing and network rules are owned by `naming`, `governance` and `sizing` and not repeated here.

## App Service philosophy

- **The platform runs the app, the app proves it is healthy.** App Service restarts, swaps and scales; the application tells it when it is ready, through its health endpoints.
- **What runs in production is what was tested.** The same container image moves from environment to environment; only configuration differs.
- **Every setting is code.** Plans, apps, slots and settings are declared in infrastructure code. A change in the portal is drift and is reverted.

## Plans and packaging

- **Applications run on App Service for Linux as container images** (`default-stack`). The image is built once by the pipeline and follows `containers`.
- **One App Service plan per application per environment.** The frontend and the backend of one application share it; another application never does, so one application's load or restart never affects another.
- **The tier follows `sizing`, with two floors:** an app that runs hosted services needs a tier with Always On, and production needs a tier with deployment slots.
- **The app pulls its image with its managed identity,** which holds `AcrPull` on the registry. The registry's admin user is disabled.

## Deployment slots and release

- **Production deploys to a `staging` slot and swaps.** The new image starts in the slot, is warmed up on `/health/ready`, and is swapped into production. A failed release is undone by swapping back.
- **Other environments deploy directly,** without a slot, unless the environment is used to test the swap itself.
- **Settings that belong to an environment, not a release, are slot settings,** so they stay with the slot when it is swapped.
- **The release pipeline checks the slot state before a swap,** as `release` requires, so a re-run never swaps back.

## Health and availability

- **The App Service health check probes `/health/ready`,** the anonymous endpoint `security` defines. An instance that fails it is taken out of rotation and replaced.
- **Always On is on for every app that runs hosted services,** in every environment. Without it the app is unloaded when idle and its background work stops.
- **Scheduled and singleton background work runs on one instance only.** When the app scales out, a hosted service that must not run twice takes a lease (a blob lease or a database lock) before it acts.
- **Scale-out rules follow the workload profile** (`sizing`). Production runs at least two instances when the workload's business criticality is `BusinessCritical` or `MissionCritical`.

## Configuration

- **Non-secret configuration is app settings,** set from infrastructure code per environment. Secrets are Key Vault references, as `security` requires, resolved with the app's managed identity.
- **The app validates its configuration at startup.** A missing setting or an unresolved Key Vault reference fails the boot, so the health check keeps the broken instance out of rotation.
- **`WEBSITES_PORT` is set to the container's unprivileged port.**

## Hardening

- **HTTPS only, the minimum TLS version from `security`, FTP and basic publishing credentials disabled, remote debugging off,** on every app and slot.
- **The app runs with its user-assigned managed identity** (`security`). A system-assigned identity is not used.
- **Inbound and outbound network access follow the network defaults** of `governance`, by the highest data classification of the workload.
- **Diagnostic settings send the platform logs to the Log Analytics workspace** (`governance`).

## Review and ownership

- **Owners:** the vault owners maintain this document.
- **Another compute service is a deviation** as `default-stack` describes.
