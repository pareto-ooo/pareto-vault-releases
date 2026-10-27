---
id: containers
scope: scoped
severity: warning
audience: all
applies-to:
  - "**/Dockerfile"
  - "**/*.Dockerfile"
  - "**/Containerfile"
  - "**/.dockerignore"
schemaVersion: 1
---

# Infrastructure standards: Containers

How deployable applications are packaged as container images, stored and promoted. These are the ground truth rules for every Dockerfile, image and container registry. Applications on the default stack run these images on App Service for Linux (`default-stack`, `app-service`); local development runs the same definitions (`local-development`).

## Container philosophy

- **One image per deployable, built once.** The pipeline builds the image once per change, and that exact image, identified by its digest, moves through every environment.
- **Small and boring.** An image holds the runtime and the application, nothing else. Less in the image is less to patch and less to attack.

## Images

- **Each deployable has its own `Dockerfile`** next to its project in `code/` (`project-folder-layout`).
- **Multi-stage builds.** Build in one stage, ship only the artefact in the runtime stage. No source, build tools, `.git`, `.env*` files, logs or data dumps in the image.
- **Minimal base images**, never a default Ubuntu or Debian tag. A .NET backend uses the Microsoft chiseled ASP.NET runtime image; a single-page app is served by an unprivileged static file server image on Alpine. Base images are pinned to the chosen major or LTS version, never to `latest`.
- **Run as a non-root user**, with no shell and no package manager in the runtime stage. No privileged or host-network flags.
- **No tools added to the runtime image for debugging** (`curl`, `wget`, an SSH client or keys). The risk outlives the one-time need.
- **No secrets in build arguments, environment variables baked into the image, or layers.** Configuration arrives at runtime, as app settings.
- **Web containers listen on an unprivileged port** (`8080`) and set strict security headers.
- **`.dockerignore` is maintained and reviewed.**

## Registry and promotion

- **Images are stored in Azure Container Registry:** one registry for non-production and one for production, each in its own subscription. Admin user and anonymous pull are disabled; pushes and pulls use Entra identities.
- **Images are tagged with the commit identifier,** never `latest`, and deployed by digest.
- **Every image is labelled with its commit, its source tree and its build run:** `org.opencontainers.image.revision` for the commit, and the tree hash and run identifier as labels of their own. A later deploy uses them to check what it promotes and to find the run's other artifacts (`release`).
- **The image is scanned before every deploy** (`quality-gates`), and the exact image that passed in acceptance is the one promoted to production.
- **Promotion to production imports the image by digest** from the non-production registry into the production registry. It is never rebuilt.
- **Old images are purged by a retention policy,** keeping at least every image that is deployed in any environment and the one before it, so a release can be rolled back.

## Review and ownership

- **Owners:** the vault owners maintain this document.
