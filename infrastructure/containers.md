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

# Container standards (TODO)

> **TODO: placeholder.** These rules were moved out of the general security standards so that container rules live with container. The rest of the container standards still need to be written. Nothing in this file is final.

## Security

- **Multi-stage builds.** Build in one stage, ship only the artefact in the runtime stage. No source, build tools, `.git`, `.env*` files, logs or data dumps in the image.
- **Minimal base images (chiseled or Alpine)**, never a default Ubuntu or Debian tag. Base images are pinned to the chosen major or LTS version, never to `latest`.
- **Run as a non-root user**, with no shell and no package manager in the runtime stage. No privileged or host-network flags.
- **No tools added to the runtime image for debugging** (`curl`, `wget`, an SSH client or keys). The risk outlives the one-time need.
- **No secrets in build arguments, environment variables baked into the image, or layers.**
- **The image is scanned before every deploy**, and the exact image that passed in Acceptance is the one promoted to Production. Images are tagged with the commit identifier, never `latest`.
- **`.dockerignore` is maintained and reviewed.**
- **Web containers set strict security headers** and listen on an unprivileged port.
