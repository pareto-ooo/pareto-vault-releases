---
id: coding-standards-all-languages
scope: global
severity: warning
audience: all
schemaVersion: 1
---

# Coding standards

The principles behind how we write code. This should be taken as the ground truth rules for writing all general-purpose language code. Specific purpose languages can follow the spirit of these rules, but have their own details elsewhere. 

## Code Philosophy

All the written code is written with these ideals in mind. This helps with "in the spirit of the rules" decisions.

- **Readability first** Code is read far more than written. Easy to read, consistent in style, cleaned up continuously. The work needs to read like a book with minimal attention jumping. (Clean Code by Robert C. Martin)
- **Pragmatism over purity.** Architecture serves the problem. 
  - When a pattern overcomplicates the code, let the service do its job internally. Adding extra abstractions within a layer for the sake of purity should be avoided.
  - The clean boundary between code layers allows for this, as the complexity must not leak across a boundary.
  - Avoid extreme pattern/architecture adherence. We do not want a dogmatic approach to the rules to hinder readability and practicality of the codebase.
- **Boy Scout Rule** Always leave code cleaner than you found it, but only within the files already part of the work.
- **Avoid over-engineering.** Do not implement for a future "what if". No abstraction without a current need. Well written code allows for extensions easily enough in the future.
- **Prefer early failure.** Code that fails on compile time is preferred. Code that fails on tests is in the middle. Code that fails at runtime is the worst.

## Change discipline

- **Incremental and non-breaking.** Keep existing mechanisms until their replacement exists.
- **Small changes stay small.** If a proper refactor touches many files, defer it to its own task or commit. Don't bundle bulk refactors into unrelated work.
- **Treat high-risk logic with extra care.** Complicated processes in the Domain and especially Legacy code that does not follow practices yet, need deep analysis of use, and pre-existing tests before any changes to ensure not accidentally breaking the code.
- **Bug fixing Root cause** When debugging and resolving bugs, always aim for a full root cause analysis. Either fix the root cause, or apply a minimal mitigation and record a TODO with a work item for the real fix. Never a silent workaround.

## Code Architecture

- **Target:** Clean Architecture, Domain-Driven Design and Hexagonal Design are the ideal target.
- **Separation of concerns.** *Domain*, *Application* and *Infrastructure* stay in their own layers and project folders. This allows for clean separation of concerns. This is a minimal level of layering to always apply. Dependencies always point inward towards the domain. Infrastructure -> Application -> Domain -> Nothing.
  - **Domain:** The actual entities and their operations to facilitate the use-cases and control invariants.
  - **Application:** Orchestrates to serve specific use-cases through Services.
  - **Infrastructure:** All external dependencies of the code-base with an anti-corruption layer where needed to protect the codebase against external influences.
- **Invariants live with their owner.** The entity or aggregate enforces its own invariants and state-transition rules.
- **Idempotent state transitions.** Where possible, a double call of a domain function should not cause adverse effects.
- **Resilience.** Code should be resilient to failures by outside influences or missing runtime dependency. A request for data that fails for example, should gracefully retry and time-out appropriately.
- **No Domain Logic in UI** UI/frontend code should avoid implementing logic of any kind that belongs in Application or Domain layer.
- **Avoid Long Running Processes** Asynchronous non-blocking code is the preference. Long running processes beyond 3 seconds should be off-loaded to background workers of some type and provide intermediate updates to the callee.
- **Normalize at the boundary** Especially at infrastructure, you want to normalize towards domain concepts in the adapter layers to ensure internal quality.
- **Dependencies** Dependencies are handed in, not reached for, in line with the separation of concerns and architecture layout.
- **External Dependencies** Try to minimize use of external dependencies. Prefer reputable sources first (Microsoft for C# for example). Anything else needs end user approval first.

## Naming

- **One word per concept.** No puns (one word, two meanings gets confusing) and no synonyms (two words, one meaning). This applies to the entirety of the project even beyond code, like database tables, infrastructure, etc.
- **Consistent verbs.** Similar to the above One Word Concept. Pick one and keep it consistent through the codebase: `Get`, `Set`, `Add`, `Remove`, `OnEventName`, `IsActive`, `HasThing`. Do not mix `Fetch` with `Get`, or `Remove` with `Delete`.
- **Classes: `<Entity><Responsibility>`.** The prefix is the entity or aggregate, the suffix the role (`OfferRepository`, `BatchService`).
- **Prevent inline qualifiers.** Try to resolve conflicting qualifiers with a unique qualifier name definition at the top of the file.
- **Poor generic names** Meaningless suffixes like `Manager`, `Helper`, `Processor`. Should generally be avoided.
- **Avoid Name encoding** No Hungarian notation or type encoding like `strName`, `AccountList`, `IntValue`, and no repeating the owner, like `UserName` inside the object `User`. A language's own exceptions are listed in its standard.
- **Exception types take the language's suffix** (`Exception` in C#, `Error` in TypeScript) and name the situation (`CustomerNotFoundException`, `ApiRequestError`).
- **DTOs name their boundary:** This allows naming of the files to explain their origin and purpose clearly, this list is not exhaustive.

  | Suffix | Purpose |
  |--------|----------|
  | `Request` | Infrastructure Request object |
  | `Response` | Infrastructure Response object |
  | `Command` | Application Write/Edit object |
  | `Query` | Application Get object |
  | `Result` | Application Result object |
  | `ReadModel` | Repository Read Model object |

 Domain Results and Input ValueObjects are just entities and aggregate objects, they have no special suffix requirement.

## Comments

- **No comments by default.** Names and types of methods and variables should clearly communicate instead of a comment. A comment required to explain what code does, requires refactoring to clear it up, not a comment. Comments that describe function, cause rot and maintenance. This is considered a bad practice.
- **TODO:** A TODO comment is warranted, but only for future planned work, that can not immediately be resolved reasonably right now. It requires a suffixed reference to a future work item. E.g.: TODO: #12345 - Old API version, planned for refactor in attached PBI number.
- **Documentation** Signatures and Summary comments on public API items to facilitate documentation generation for API/Swagger documentation can be used. This should be rare outside the presentation Infrastructure layer.

## Design and Code smells

Code smells as described in Clean Code (Robert C. Martin) are a good guideline to keep in mind.

- **Temporal coupling.** Avoid and refactor Temporal Coupling, where function execution order causes misbehavior of a service or entity.
- **Anemic Domain.** Entities/Aggregates should enforce their own invariants. Invariants enforced in a Domain Service instead of their source is to be avoided.
- **Prevent God Classes** Aggregates should be 'what must be transactionally consistent together to protect an invariant'. One transaction changes one aggregate. Prevent nested aggregate objects by reference, they should be by Unique ID.
- **Indentation** Indentation makes code hard to read.
  - Early Out over If/else blocks
- **Cyclomatic Complexity <= 10** A high cyclomatic complexity is hard to figure out, strategically split code blocks to reduce nested logic blocks to reduce this to a readable bit of code.
- **Method Length** Methods should not exceed 30 lines, except when it makes no sense to split it (like config files or object creation, mapping, those will just take many lines if there are many parameters)
- **Boolean Parameters as flags** Boolean parameters as flags indicate there should be two named methods instead.
- **Line Length** Lines over 120 characters do not fit on screen and should be split with a new line, or cause an intermediate value to reduce length.
- **Complex Conditions** An if statement that compares more than two things at once, should instead have named intermediate value(s) to clarify what the individual checks are to improve readability.
- **Ordering by Flow** Public before Private Methods, Entry -> Mutation -> Exit. Constants -> Fields -> Properties -> Constructors -> Methods. The class needs to read like a chronological book.
- **Positive Conditions** Prefer IsValid over IsInvalid. Positive defaults makes it consistent to reason about condition checks.
- **Magic Values** Avoid magic values. Any value used more than once should be a named private value.
- **No output or in/out parameters** unless the language idiom requires them. Return values instead.
- **Empty collection, not absent.** A function that finds no results returns an empty collection, never `null` or `undefined`.
- **Get and Find.** A `Get` function returns a value or throws when it cannot. A `Find` function returns a value or an explicitly typed absent value (`Customer?` in C#, `Customer | undefined` in TypeScript).

## Error handling

- **No exceptions for flow control.** Exceptions are for error handling.
- **Expected Failures** Invalid user input and similar expected Domain exceptions should be structured as user presentable failure codes.
- **Validate first, then act.** Guard clauses and early-outs at the top of the method.
- **User-facing errors** Built and localized at the presentation layer.
- **Logging** Errors and Exceptions must always be logged and traceable properly to the standard outputs defined by the project. Do not log identifiable information (names or numbers related to the business), secrets, or tokens. Internal relational IDs for debugging purposes however are a must (what item did we fail on).
- **Rejected input yields a client error with field-level detail** and nothing about internals. Expected bad input is not an exception to log with a stack trace.

## Testing

- **When to add Tests.** New code in the Domain or Application layer always comes with full test suites. Adapters on the Infrastructure layer also need tests to ensure our anti-corruption layer. Infrastructure outside of our codebase should not be under test. UI Tests should validate UI to Application contracts, any other tests in UI would be too brittle to maintain.
- **Isolated and repeatable.** No reliance on shared external state or side effects like reading the system clock. A test that needs a database or other backing service gets its own throwaway instance (a container or the provider's emulator), never a shared one. Tests use data they control or that is seeded for them.
- **Run Tests in their layer.** Domain tests cover domain classes; Application layer tests should not count toward Domain coverage.
- **Prefer focused unit tests over broad integration tests.** Small accurate unit tests for most code.
- **Arrange-Act-Assert format.** Add a comment between each step in a test. This is for readability, and the one exception to "No comments by default".
- **Naming:** `MethodName_StateUnderTest_ExpectedBehavior`.
- **Don't write brittle tests.** Tests based on string related checks, or tests based on reflection are discouraged.
- **Exception testing**, exceptions are validated through type and included data values, not the exception message.
- **Layout mirrors source.** One test class per source project class under test, in the matching locations in the Test project folders. Don't split tests into multiple smaller files per source file. The test file mimics the original file name with a `Tests` suffix.
- **Coverage.** Coverage aims for >90% coverage both on lines and branches. Data objects and Generated code are excluded from coverage.
- **Coverage quality.** Coverage of code should assert the expected behavior of the code.
- **Bugs.** Bugs that get resolved in Domain or Application layer, need a matching test to ensure the bug does not regress.
- **Do not Assert Non Requirements.** For example, if the returned order is chronological purely by accident, not by specific requirement, it should not be asserted.
- **Speed.** Testing suites should be fast. Long timeout tests, full integration tests, and delays should be avoided unless no other options exist.
