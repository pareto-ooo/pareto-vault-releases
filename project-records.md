---
id: project-records
scope: global
severity: critical
audience: all
schemaVersion: 1
---

# Project records

The records every project keeps about itself: what each one holds, where it lives in the repository and when it must exist. Other documents in this vault ask a project to "record" a fact; this document says where that fact goes. The content rules of each record stay with the document that asks for it.

## Record philosophy

- **One place per fact.** Each fact a project records has exactly one home, listed below. A fact that is needed elsewhere is referred to, not copied.
- **Records live in the repository.** They sit under `docs/`, are changed by pull request and are versioned with the code they describe. A record in a wiki, a chat or a slide deck does not count.
- **Roles, not people, in the vault; people in the project.** The vault names roles. The project records who holds each one.
- **A missing record blocks only what depends on it.** Each record below says when it must exist; until then the work it gates waits, and other work continues.

## The project definition

The project definition is the single document that states what the project is and the facts the vault asks a project to settle. It lives at `docs/project-definition.md`.

- **A signed charter is the first source.** When a Discovery Sprint produced a signed charter (`docs/discovery/PROJECT-CHARTER.md`), the facts the charter holds are owned by the charter. The project definition holds the remaining facts and names the charter for the others; it never restates them.
- **It is kept current.** A fact that changes is changed here, in the same pull request as the work that changes it.

| Fact | Asked for by | Must exist |
|---|---|---|
| Purpose, users and scope (in and out) | this standard | before design |
| PACE layer and business criticality | `principles` | before design |
| Impacted capabilities (L2 or L3; innovating layer exempt) | `business-capabilities` | before design |
| Organisation substitutions, client projects only (see below) | this standard | before design |
| Compliance frameworks that apply, and the controller or processor role per app | `compliance` | before design |
| Highest data classification | `compliance` | before design |
| Role holders and how to reach them: privacy owner, security owner, incident contact, architect who signs deviations, information security officer, IT director, enterprise architect. One person may hold several roles, and the record says so | `compliance`, `deviations` | before design, and before the first deviation |
| Tenancy: single-tenant, or multi-tenant with how users belong to tenants | `security` | before design |
| Error domain for problem `type` URIs | `api` | before the first error code appears in a response |
| Languages the product ships and its default locale | `coding-standards-vuejs` | before design |
| Accessibility law that applies (European Accessibility Act, public-sector rules) or none | `coding-standards-vuejs` | before design |
| Changes to the standard test levels, and why | `testing-strategy` | before the first pipeline |
| Support hours, for `BusinessCritical` workloads | `observability` | before go-live |
| Workload profile | `sizing` | before any service tier is chosen |
| Cost estimate per environment, and the performance test result for critical workloads | `sizing` | before the first deployment; the test result before go-live |
| Promotion pattern (environment branches or tag promotion) | `release` | before the first pipeline |
| Quality-gate substitutes and the reason | `quality-gates` | when a substitute is used |

### Organisation substitutions

The vault is written for Pareto's own organisation. On a client project, wherever a document names Pareto's own setup, the client's takes its place, and the project definition lists each substitution:

- the Azure tenant and subscriptions;
- the `<company>` token in resource names;
- the identity provider for the client's employees;
- the business capability model, or that the client has none;
- the compliance frameworks, and whether Pareto or the client is the controller;
- the central mail delivery service;
- the package feed, where the client has one;
- the integration platform, where the client has one.

A document whose `audience` field is `internal` is skipped on a client project, not substituted.

## Readiness before build

Build starts when the project's requirement set answers each item below, or names it as an open item with an owner. The requirement set is the Discovery Sprint's registers when the project ran one, otherwise `docs/requirements/`.

1. Goal and context: the problem, who has it and what success looks like.
2. Glossary of the domain terms the requirements use.
3. User stories, each with acceptance criteria in Given/When/Then form.
4. Data model: entities, attributes, relationships with their cardinality, and status lifecycles.
5. Roles and rights: which role may do what on which entity and screen.
6. Flows and screens: the main processes and the screens each step uses.
7. Integrations: each system, the direction, the data, and what happens when it fails.
8. Non-functional requirements: performance, availability, security, privacy and data classification.
9. Edge cases and business rules.
10. Scope: what is in the first release and what is explicitly out.
11. Deviations from `default-stack` and any other standard, each recorded as a decision.
12. Data migration: what moves from which system, and how it is reconciled.

## Other records

| Record | Location | Holds | Must exist |
|---|---|---|---|
| Architecture overview | `docs/architecture.md` | a context and a container diagram (C4 levels 1 and 2), and the main flows between the containers | before the first deployment |
| Decision records | `docs/decisions/NNNN-<slug>.md`, one file per decision, numbered in order | a decision that changes architecture, a contract, a data store, a security or compliance boundary, or the cost; every deviation, with the content `deviations` requires | when the decision is made, before the work that depends on it is merged |
| Data register | `docs/data-register.md` | every data store and flow with its classification, the personal data in it, purpose, retention and location, including copies such as caches and report tables | before the first store of personal data is deployed |
| Error catalog | `docs/error-catalog.md` | what `api` requires of each error code | before the first error code appears in a response |
| Dependency record | `docs/dependencies.md` | each direct package with what `compliance` requires of it | before the package is merged |
| Threat models | `docs/threat-models/<feature>.md` | what `compliance` requires of a threat model | during planning of a feature that needs one |
| Known weaknesses | `docs/known-weaknesses.md` | each finding not fixed immediately, with what `compliance` requires of it | when the finding is not fixed immediately |
| Runbooks | `docs/runbooks/<procedure>.md` | the trigger, the steps, how to roll back and whom to escalate to | before the first production deployment that needs the procedure |

- **A decision record has a fixed shape:** title, status (`proposed`, `accepted`, or `superseded by NNNN`), date, context, decision and consequences, plus what `deviations` requires for a deviation. An accepted record is never edited except for its status; a changed decision is a new record that supersedes it.
- **Diagrams are text.** Architecture, threat model and flow diagrams are Mermaid in the Markdown file, so they are reviewed in pull requests and cannot drift from an image nobody can edit. The architecture overview uses the C4 notation, and a threat model diagram (`compliance`) builds on its container diagram.
- **The repository README is the entry point.** It says what the application is, how to run it locally (`local-development`), and links to the records in `docs/`.
- **"Decision record" is the one term.** A decision log, an ADR or an architecture decision in any document of this vault means a decision record.
- **Decisions taken in a Discovery Sprint stay where the sprint recorded them.** Their deviations are listed in the charter. Decision records start with the first decision after the charter is signed.

## Review and ownership

- **Owners:** the vault owners maintain this document.
- **A new kind of record is added here first,** before another document asks a project to keep it.
