---
id: quality-gates
scope: scoped
severity: warning
applies-to:
  - "**/*.yml"
  - "**/*.yaml"
schemaVersion: 1
---

# Pipeline standards: Quality gates

The checks a change must pass before it is merged.

## Gate philosophy

- **Gates run on the pull request.** A gate decides whether a change merges, so it runs in the pull request pipeline. A scheduled run is an extra signal, never the only place a required gate runs.
- **A gate may be scoped to the paths it judges.** When pull request validation is split by path, every required check of that part, including the SonarCloud status check, has the same path filter. A required status check without that filter blocks every pull request whose pipeline did not run. On GitHub the reverse applies: a required check must report on every pull request, as the GitHub Actions standards describe.
- **Block on what matters, report the rest.** A gate blocks the merge on a failed build or test, a failed quality gate, a secret, and critical or high security findings. Medium findings are reported as a warning on the pull request; low findings are only reported.
- **Gates judge new code.** A pull request is judged on what it changes, not on the existing backlog. Existing findings are picked up by scheduled scans that create work items.
- **A hotfix uses an exception, never a weaker gate.** The gate stays in place for every change; an urgent fix goes through the exception process below.

## Quality gate

- **The code builds and all tests pass.**
- **SonarQube Cloud (SonarCloud) analyses every pull request.** The analysis runs in the pull request pipeline of the code it analyses, through the SonarCloud tasks (prepare, analyse, publish) around the build and test steps, and its result is published to the pull request.
- **The SonarCloud quality gate must pass.** A failed quality gate blocks the merge. The quality gate covers new code: coverage, duplication, maintainability, reliability and security issues.
- **Coverage is reported to SonarCloud.** Test coverage is collected during the test run and imported by the analysis, so the coverage condition in the quality gate is measured on the pull request.
- **The SonarCloud quality gate matches the coding standards.** The coverage threshold in the quality gate is the threshold from the coding standards; a project does not lower it on its own.
- **Exclusions are deliberate.** Generated code and data objects are excluded from coverage, in line with the coding standards; every other exclusion is a reviewed choice in the SonarCloud project settings.

## Security gate

- **Each scan has one tool.** SonarCloud scans the code; GitHub Advanced Security (for GitHub, or for Azure DevOps) scans for secrets and vulnerable dependencies (Cloud Governance GS-10, DS-02); Trivy scans infrastructure code.
- **A secret always blocks.** GitHub Advanced Security secret scanning is on with push protection: a secret is blocked at push, and a secret detected in a change blocks the merge.
- **Critical and high findings in new code block.** Code vulnerabilities from SonarCloud, vulnerable dependencies introduced by the change from GitHub Advanced Security, and infrastructure misconfigurations from Trivy block the merge at critical or high severity.
- **Medium findings warn, low findings report.** They are visible on the pull request but do not block it.
- **Security hotspots are reviewed, not ignored.** A hotspot is resolved in SonarCloud as fixed or as safe with a reason, never left open.
- **Trivy runs on every pull request that changes infrastructure code,** using the Trivy extension installed in the Azure DevOps organisation or the Trivy action on GitHub.
- **The existing backlog is scanned on a schedule.** Dependency and infrastructure scans also run on a schedule against the main branch, because new vulnerabilities are published for code that did not change; findings become work items.
- **Web applications are scanned periodically** with ZAP against the OWASP rule sets (Cloud Governance GS-10). This runs on a schedule against a deployed environment, not on the pull request.
- **An application without a deployed environment is scanned against itself in the pipeline.** Until it has one, the scheduled run starts the application stack in the pipeline and runs a ZAP baseline scan against it. Findings are handled like other scheduled findings.

## Without SonarCloud or GitHub Advanced Security

A repository where SonarCloud or GitHub Advanced Security is not available (for example a private repository on a free GitHub plan) uses these substitutes. The substitutes and the reason are recorded in the project definition, and the project moves to the standard tools when they become available.

| Gate | Substitute |
|------|------------|
| Coverage | The test run enforces the coding standards' coverage threshold and fails the pipeline below it |
| Code quality and code scanning | The language's own analyzers and linters at their strictest setting, with warnings as errors (the .NET analyzers, ESLint, `vue-tsc`) |
| Secrets | gitleaks on every pull request; a finding blocks |
| Vulnerable dependencies | Trivy filesystem scan of the committed lock files on every pull request, blocking at critical and high, plus Dependabot alerts |

- **The same thresholds apply.** A substitute blocks on what the standard tool would block on, and reports the rest.
- **What no substitute covers is named.** The record lists what the standard tool would have checked that the substitutes do not (for example SonarCloud's security hotspots and duplication), so the gap is known.

## Exceptions

- **A single finding is accepted, not ignored.** A finding that is a false positive or an accepted risk is resolved in its tool (for example accepted in SonarCloud) with a written justification, so it no longer counts in the gate.
- **A blocked hotfix merges through the bypass permission.** The permission to bypass the branch rules when completing a pull request (the Azure DevOps bypass-policies permission, or a GitHub ruleset's bypass list) is held by a small, named group. The person who bypasses records the reason on the pull request.
- **Every exception has a follow-up.** An accepted finding or a bypass gets a work item to fix the cause, so the exception is temporary.
- **Exceptions are audited.** Bypasses are visible in the platform's audit log and are reviewed periodically; a gate that is bypassed routinely is reviewed instead of bypassed again.
