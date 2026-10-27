---
id: principles
scope: global
severity: warning
audience: all
schemaVersion: 1
---

# Architecture standards: Enterprise architecture principles

The enterprise architecture context every solution is designed in. These principles are generic: they hold for any organisation we build for, internal or client. A client's own architecture principles take precedence where they differ, and the difference is recorded. They give direction to the design and realisation of processes, information systems, technical infrastructure and organisation, and every other standard in this vault works within them.

## Why architecture

- **Aligned IT for growth.** Without an overarching direction, departments look for solutions at different moments, which leads to scattered solutions, a higher Total Cost of Ownership and a longer time to market.
- **Architecture supports decisions.** It gives insight into the information systems, keeps IT costs (CAPEX and OPEX) in control, and prevents scattered solutions. It needs the support of management, IT and the business.
- **Architecture is dynamic.** It follows the pace of the organisation: direction is set up front, detail is added when a solution needs it.

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
- **We think big, but start simple.** Set the end goal at a high level, deliver in small steps, and prioritise requirements with MoSCoW, and be critical of everything below must-have.
- **We implement uniform processes.** Processes are designed for growth and uniformity, never for one-off exceptions; they are documented and reuse existing process components.
- **We ensure trust in our information security.** Changes are checked against the organisation's information security policy and the security standards for hardware, software and data; access rights are managed centrally and role-based.
- **We do not change to benefit one department and harm others.** The impact on other departments and on dependencies earlier and later in the chain is assessed with the stakeholders.

## Information architecture

- **We reuse before buy before build.** Every change starts with a scan of the existing application landscape. Buying or building only follows when existing functions are not sufficient.
- **We design for growth.** Platforms are configurable for new markets, laws, currencies, languages and business rules without core modifications; vendors show proven scalability and multi-tenant readiness; fragmented solutions are decommissioned in favour of standardised platforms.
- **We use a single source of truth.** Data can be traced back to its source, and a change reuses existing sources and fields before adding new ones.
- **We standardise where possible, extend where needed.** Core applications are not customised; differentiation is built in extensions, integrations and adjacent solutions. A deviation from standard goes through the approval process, which assesses functionality, scalability, security, cost and upgradeability, and is documented.

## Technical architecture

- **We ensure availability of critical data and systems.** Critical processes are in the organisation's disaster recovery plan, critical data and systems are known and backed up, and the people involved know the recovery procedure.
- **We use proven and available technology, resources and knowledge.** Technology is proven in relevant environments and supported by reliable partners; experimental solutions are only considered when no proven alternative exists.
- **We use cloud and service-based solutions.** Cloud first for new solutions and major changes; service-based offerings (SaaS, managed platform services) are preferred over self-managed solutions. SaaS is governed as a business service focused on outcomes; IaaS and PaaS are governed as technical platforms with more internal responsibility.

## Architecture models

These models are maintained to keep insight into the architecture. A solution that changes what a model describes updates that model.

| Domain | Models |
|--------|--------|
| Business architecture | IT policy, products and services overview IT |
| Information architecture | Application landscape, interface overview, user access matrix, domain model |
| Technical architecture | Technology platform landscape |
