---
id: testing-strategy
scope: global
severity: warning
audience: all
schemaVersion: 1
---

# Code standards: Testing strategy

Which kinds of tests a project has, what each covers, where it runs and what it blocks. These are the ground truth rules for testing beyond the single test. How unit tests are written is owned by `coding-standards-all-languages` and the language standards, the merge gates by `quality-gates`, and performance tests by `release` and `sizing`; they are not repeated here.

## Testing philosophy

- **Most tests are fast and small.** Each higher level only covers what the levels below cannot, because higher levels are slower, more brittle and more expensive.
- **A test that does not run automatically is not a control.** Every level runs in a pipeline or on a schedule, except acceptance by people.
- **Tests own their data.** A test creates or seeds what it needs, and never depends on what another test or a person left behind.

## Test levels

| Level | Covers | Runs against | Blocks |
|-------|--------|--------------|--------|
| Unit | Domain and application logic | Memory only | The merge |
| Integration | Adapters: repositories, HTTP clients, message handlers, and the API's endpoints in process | A throwaway container or emulator per run | The merge |
| Contract | The API and event contracts against their published versions | The pipeline | The merge |
| End-to-end | The critical user journeys | A deployed environment | Promotion to the next environment |
| Performance | The targets of the workload profile | Acceptance, with production tiers | Go-live, as `release` describes |
| Acceptance | The change meets its acceptance criteria, judged by the business | Acceptance | Production |

- **Integration tests use the real provider** (Testcontainers or the provider's emulator), never an in-memory stand-in (`coding-standards-csharp`).
- **The API contract is checked on every pull request.** The OpenAPI document the build exports (`api`) is compared with the one of the last release, with a tool such as `oasdiff`. A breaking change fails unless the major version changes with it.
- **Event contracts are checked the same way.** A changed JSON Schema is compared with its published version, and a breaking change fails unless it goes to a new topic version (`inter-service-communication`).
- **End-to-end tests cover the critical user journeys only** (`coding-standards-vuejs`) and run after each deployment to the environment they test, before that change is promoted further.
- **Acceptance is recorded on the work item:** who accepted, when, and against which build.

## Test environments and data

- **Tests below end-to-end need no shared environment.** They start their own backing services and throw them away.
- **Deployed test and acceptance environments hold synthetic data only** (`compliance`). It is loaded by a repeatable seed job from the repository, built from the same generators as the local seed data (`local-development`).
- **End-to-end tests create the data they use** through the application's own API, with unique values, so they can run in parallel and in any order.
- **Production is never a test environment.** It holds no test accounts or test data. After a deployment only read-only checks run there: the health checks and the availability tests (`observability`).

## The project's test approach

- **The project definition records any change to these levels** (`project-records`), such as a level that does not apply or an extra one a client requires. Without such a record, every level applies.

## Review and ownership

- **Owners:** the vault owners maintain this document.
