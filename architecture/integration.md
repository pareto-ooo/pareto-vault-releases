---
id: integration
scope: global
severity: warning
audience: all
schemaVersion: 1
---

# Architecture standards: Integration

How we design integrations between systems. These are the ground truth rules for every integration built in-house; integrations built by third parties are out of scope. They work out the architecture principle "We use a single source of truth".

## Integration philosophy

- **Keep integrations simple.** An integration automates something that could be done by hand. The simplest design that does this reliably wins.
- **Data comes from its source of truth.** Every dataset has one authoritative source, and integrations read from it. When the same data is already replicated in a system the integration must connect to anyway, it may be read from there instead, for efficiency.
- **Explicit over implicit.** Matching, selection and transformation are stated exactly, never assumed or derived from patterns.
- **Changes go through the application.** Data in another system is only changed through its application layer (an API or the application itself), never directly in its database, so validation, security and business rules stay enforced.

## Topology

### API exposure

Azure API Management (APIM) is not used by default. An API is exposed directly by its app, and the app handles authentication and rate limiting itself (`api`).

- **Using API Management is an architecture decision,** taken by the project that needs it (for example for many external consumers, subscription keys, or a public contract that must stay stable while the internal API changes) and recorded as a decision record (`project-records`). The record states the instance, the products and how the project publishes its APIs.
- **The external contract is decoupled from the internal API.** API Management maps the public contract to the internal API, so the internal API can change without breaking external consumers, and versions are managed at the front door.
- **API Management handles the edge.** Edge authentication, subscription and key management, throttling and request and response translation are handled there, not also in the app.
- **The app still validates every token.** Authentication at the edge does not replace validation in the backend.

### Integration platform

- **There is no shared integration platform.** Each project builds its own integrations, as calls to the other system's API or as messages (`inter-service-communication`).
- **A client project uses the integration platform the client already has,** listed as a substitution (`project-records`).
- **Adopting a dedicated integration service** (such as Logic Apps or Data Factory) is an architecture decision, recorded as a decision record (`project-records`).

## Decoupling

- **The integration is decoupled from the data model.** It connects through an API, a database view or a message, not to internal tables, so a change in a system's data model has the smallest possible impact. The vendor of the system owns that interface.
- **Synchronous or asynchronous is a deliberate choice.** A synchronous integration (a direct call) is simpler and fits when the sender needs an immediate answer; when the receiver is down, the call fails and the sender handles that failure. An asynchronous integration (a queue or message bus) fits when the receiver may be unavailable or slow, so messages wait until it can process them. The choice and how failures are handled are part of the integration design.
- **Interfaces are versioned** so a system upgrade does not force an immediate change of every integration.
- **The source system is not overloaded.** Changes are read incrementally (change data capture, events or an API with throttling), and heavy integrations run outside operational hours.

## Data handling

- **Match on business keys, not technical keys.** A business key (a contract or customer number) is shared across systems; a technical key is only unique within one.
- **No fuzzy matching.** Data sets are never combined on non-exact criteria such as names.
- **No derived keys or wildcard attributes.** When a unique key or a needed attribute is missing, the source system is extended, unless it is external. Partial keys built from logic and selections on string patterns are not used.
- **Mapping tables for lists, expressions for simple conditions.** A long list of values to include or exclude lives in a mapping table that can change without a release; a short, stable condition stays in the code.
- **Soft failures for incomplete data.** Missing data from a non-critical source is handled by design (for example an "Unknown" value or an empty attribute) instead of failing the flow.
- **Data is validated and cleaned as part of the integration.** Accuracy, consistency and completeness are checked before data is passed on, and faulty data is reported to the data steward or application manager.

## Quality and operation

- **Cost is a design input.** Run frequency, required capacity, data volume, logging retention and reuse are weighed in the design.
- **Parallel over sequential** where the systems allow it, so the load on source and destination is short.
- **Hard failures alert, soft failures are recorded.** An unexpected failure (a system down, a changed schema) alerts the maintainers; only critical failures alert, to avoid alert fatigue. Soft failures are logged for follow-up.
- **Every integration is monitored.** A dashboard or report shows whether runs succeed, how many soft failures occur and how performance develops over time.
- **Secure by design.** Transport is encrypted, secrets live in Key Vault, and the integration and its developers get least-privilege access.
- **Version control and environments.** Integrations are in source control and are tested in an environment before production, with enough isolation that tests never affect production.

## Documentation and support

- **High-level documentation describes the integration:** its design and components, configuration, schedule and order of operations, and how to deploy, handle soft failures and monitor it.
- **Support is arranged before go-live.** Who manages the integration, what happens when it fails, whether a manual fallback exists, whether it matters for disaster recovery, and who the backup maintainer is.
