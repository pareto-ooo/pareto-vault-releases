---
id: architecture-principles
scope: global
severity: warning
schemaVersion: 1
---

# Architecture standards: Enterprise architecture principles

The enterprise architecture context every solution is designed in. These are Pareto's IT architecture principles. **TODO:** confirm them as Pareto's own; they were adopted from an earlier client's architecture principles. They give direction to the design and realisation of processes, information systems, technical infrastructure and organisation, and every other standard in this vault works within them.

## Why architecture

- **Aligned IT for growth.** Without an overarching direction, departments look for solutions at different moments, which leads to scattered solutions, a higher Total Cost of Ownership and a longer time to market.
- **Architecture supports decisions.** It gives insight into the information systems, keeps IT costs (CAPEX and OPEX) in control, and prevents scattered solutions. It is supported by the managing board, IT and the business.
- **Dynamic Architecture (DYA).** Pareto follows Sogeti's DYnamic Architecture model, because its flexibility fits a rapidly changing environment. **TODO:** confirm.

## How principles work

- **Principles guide, they are not laws.** A deviation is possible, but it is recorded and signed off by an architect.
- **Every principle has a motivation and implications,** has been tested by decision makers, and is measurable or verifiable.
- **Designs are just enough, just in time.** Architecture documents give detail to the principles when a solution needs them, not ahead of need.
- **Three domains.** Business architecture: does the technology fit the organisation? Information architecture: is the information useful and reliable? Technical architecture: is the technology safe and available?

## Application layers (PACE)

Every application is assigned to one layer. The layer sets its expected pace of change, integration approach and governance intensity.

- **Every project records its layer in the project definition,** next to its business criticality, before design starts.
- **Layer and business criticality together set the effort.** The layer decides how much architecture, documentation and governance a project carries; the business criticality decides how much availability, testing and protection it needs. An innovating, non-critical project carries the least of both; a foundational, mission-critical one the most.
- **A layer change is a decision.** An innovating application that is scaled moves to the differentiating or foundational layer, and takes on that layer's stance before it does.

| Layer | Pace of change | Architecture stance | Integration stance | Governance stance |
|-------|----------------|---------------------|--------------------|-------------------|
| Foundational (systems of record) | Low, controlled | Standardise, configuration first, minimise customisation, be ready for continuous vendor releases | Stable contracts, clear ownership of data and interfaces | Strict, controlled change due to business criticality |
| Differentiating (systems of differentiation) | Medium, regular | Extend rather than customise the core; platforms and reusable components; keep the core upgradeable | Standardised APIs and events, reusable integration patterns | Clear product ownership and a managed backlog |
| Innovating (systems of innovation) | High, rapid | Fast delivery on approved platforms, lightweight documentation, security by design | Limited and controlled; decoupled patterns that protect foundational systems | Decide upfront whether the outcome scales to another layer or is retired |

## Business architecture

- **We act in the interests of the external customer.** Every change considers the customer's perspective; exceptions need support within the organisation.
- **We think big, but start simple.** Set the end goal at a high level, deliver in small steps, and prioritise requirements with MoSCoW, critical on everything below must-have.
- **We implement uniform processes.** Processes are designed for growth and uniformity across countries, never for one-off exceptions; they are documented and reuse existing process components.
- **We ensure trust in our information security.** Changes are checked against the Information Security Policy and the security standards for hardware, software and data; access rights are managed centrally and role-based.
- **We do not change to benefit one department and harm others.** The impact on other departments and on dependencies earlier and later in the chain is assessed with the stakeholders.

## Information architecture

- **We reuse before buy before build.** Every change starts with a scan of the existing application landscape. Buying or building only follows when existing functions are not sufficient.
- **We design to enable scalable international growth.** Platforms are configurable for local laws, currencies, languages and business rules without core modifications; vendors show proven scalability and multi-tenant readiness; fragmented solutions are decommissioned in favour of standardised platforms.
- **We use a single source of truth.** Data can be traced back to its source, and a change reuses existing sources and fields before adding new ones.
- **We standardise where possible, extend where needed.** Core applications are not customised; differentiation is built in extensions, integrations and adjacent solutions. A deviation from standard goes through the approval process, which assesses functionality, scalability, security, cost and upgradeability, and is documented.

## Technical architecture

- **We ensure availability of critical data and systems.** Critical processes are in the IT Disaster Recovery plan, critical data and systems are known and backed up, and the people involved know the recovery procedure.
- **We use proven and available technology, resources and knowledge.** Technology is proven in relevant environments and supported by reliable partners; experimental solutions are only considered when no proven alternative exists.
- **We use cloud and service-based solutions.** Cloud first for new solutions and major changes; service-based offerings (SaaS, managed platform services) are preferred over self-managed solutions. SaaS is governed as a business service focused on outcomes; IaaS and PaaS are governed as technical platforms with more internal responsibility.

## Architecture models

These models are maintained to keep insight into the architecture. A solution that changes what a model describes updates that model.

| Domain | Models |
|--------|--------|
| Business architecture | IT policy, products and services overview IT |
| Information architecture | Application landscape, interface overview, user access matrix, domain model |
| Technical architecture | Technology platform landscape |
