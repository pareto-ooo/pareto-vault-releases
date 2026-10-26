---
id: source-control
scope: global
severity: warning
applies: all
schemaVersion: 1
---

# Source control

How we commit, branch, review and version. Commits follow Conventional Commits 1.0.0 and versions follow Semantic Versioning 2.0.0. Both specifications say nothing about branching or pull requests, so those two chapters are our own convention.

## Commits

- **Every commit message follows Conventional Commits.** The shape is `<type>[(scope)][!]: <description>`, then an optional body, then optional footers. Body and footers are each separated by one blank line.
- **The description is a short summary, in the imperative** (`add retry to hub client`), without a trailing period.
- **The scope is a noun in parentheses** naming the part of the codebase: `fix(billing): round vat per line`.
- **The body explains why**, in free-form text. The diff already shows what.
- **Footers use git-trailer form**: `Refs: #12345`, `Reviewed-by: Name`. Tokens use hyphens instead of spaces.
- **One commit holds one type of change.** If it would need two types, split it into two commits.
- **A commit is small and complete.** Every commit on `main` builds and its tests pass. On a branch, a test-first commit with a deliberately failing test is allowed when its description says so (`test(billing): add failing vat rounding case`).
- **Rewriting history is never allowed.** A wrong type or message is fixed by a correcting commit. This is enforced by branch policies. As a result there is no ability to rebase.

### Types

| Type | Use for | Version effect |
|---|---|---|
| `feat` | A new feature | MINOR |
| `fix` | A bug fix | PATCH |
| `docs` | Documentation only | none |
| `style` | Formatting, no meaning change | none |
| `refactor` | Restructuring without behavior change | none |
| `perf` | A performance improvement | none |
| `test` | Adding or correcting tests | none |
| `build` | Build system or dependencies | none |
| `ci` | Pipeline configuration | none |
| `chore` | Anything else that does not touch the product | none |
| `revert` | Undoing an earlier commit | none |

### Breaking changes

- **A breaking change is marked with `!` before the colon** (`feat(api)!: remove v1 customers endpoint`), a `BREAKING CHANGE: <description>` footer, or both.
- **It can accompany any type**, and it always means a MAJOR version bump.
- **The description (or the footer) says what breaks and what to do about it.**

### Reverts

A revert is a `revert` type, with a `Refs:` footer listing the reverted commit hashes.

## Branching and Merging

- **`main` or legacy `master` is always releasable.** Nothing is committed to it directly. Everything arrives through a pull request.
- **Branch from `main`, and keep branches short-lived.** Merge small and often, so the branch does not drift. Usually this means once per feature delivered, so the whole feature is self-contained as a PR.
- **Branch names are `<type>/<short-kebab-description>`**, using the same types as commits: `feat/vault-assurance-alignment`, `fix/db-migrations-glob`.
- **One PR and branch holds one concern.** Unrelated work gets its own branch, so it can be reviewed and reverted alone. For example documentation, or refactoring changes.
- **The PR title follows the commit format** (`fix(practices): repair dead globs`). With squash merging, the title becomes the commit message on `main`, so it carries the type and the version effect.
- **The description says what changed, why, and how it was verified.** It references the work item (`Refs: #12345`) and calls out any breaking change.
- **(ISO27001) A PR needs a passing pipeline and a review before it merges.** The author does not merge their own unreviewed changes, and cannot approve their own PR. A team of one person cannot meet this. It works under a recorded and signed deviation whose compensating controls are a recorded assistant review and vault review on every PR, and required checks that pass before the author merges.
- **Reviewers check the commit type against the change.** A `fix` that adds behavior, or a missing `!` on a breaking change, is a review finding.
- **Merge by squash**, keeping `main` one commit per PR, unless the individual commits are each worth keeping.
- **Delete a branch after it is merged.** By default only your own branches can be deleted, usually this is automatically done through the Dev-Ops PR system with auto-complete enabled.

## Versioning

Semantic Versioning 2.0.0, in the form `MAJOR.MINOR.PATCH`.

- **MAJOR** for incompatible changes to the public API. **MINOR** for backward-compatible new functionality. **PATCH** for backward-compatible bug fixes.
- **Raising a number resets the lower ones to zero.** `1.4.2` becomes `1.5.0` for a feature and `2.0.0` for a breaking change.
- **The public API is declared.** For a service it is the documented HTTP contract, events and published packages. Anything outside that is internal and does not drive the version.
- **Commit types drive the bump**: any breaking change means MAJOR, otherwise any `feat` means MINOR, otherwise any `fix` means PATCH. Other types alone do not release.
- **`0.y.z` is initial development.** Anything may change at any time. Start at `0.1.0` and raise the minor number per release. Move to `1.0.0` once the software is in production or others depend on its API.
- **A released version never changes.** A correction is a new version.
- **Deprecate before removing.** A deprecation ships in a MINOR release, stays for at least one more minor, and is removed only in a MAJOR.
- **Bumping Version** A release is the point the new version number is established. The version is tagged on `main` and deployed through the pipeline by that specific tagged commit.
- **A release is an annotated tag on `main`**, named `vMAJOR.MINOR.PATCH`.
