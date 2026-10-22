---
id: business-capabilities
scope: global
severity: warning
schemaVersion: 1
---

# Architecture standards: Business capabilities

How projects relate to the Business Capability Model (BCM). The BCM is the single, enterprise-wide map of what Pareto must be able to do, independent of how it is organised, which processes it uses or which applications support it. These rules describe how a project uses the BCM; maintaining the BCM itself is the responsibility of the enterprise architect and the capability owners.

## Capability philosophy

- **What, not how.** A capability is an observable business ability. Processes, applications and teams change; capabilities stay stable.
- **One map.** There is one enterprise-wide BCM. No project or department keeps its own capability map.
- **Capabilities connect strategy to change.** Initiatives, applications and investments are traced to the capabilities they improve, create or retire.

## Projects and capabilities

- **Every project records the capabilities it impacts** in the project definition: the L2 or L3 capabilities it improves, creates or decommissions. Projects in the innovating PACE layer are exempt.
- **Capabilities are referenced by their BCM name or identifier,** never by a local label, so mappings stay unambiguous across projects.
- **Scope follows capabilities.** Solution scope and architecture decisions are derived from the capabilities in scope; applications are mapped to capabilities, not the other way around.
- **A project does not redefine capabilities.** A project may add local mappings (its applications, its KPIs), but never renames, moves or redefines a capability.
- **A missing or disputed capability is a change request.** When a project needs a capability that is not in the BCM, or the scope of one is unclear, it raises a change request to the enterprise architect instead of creating its own.

## Capability language

When a project proposes a new or changed capability, it follows the BCM standard:

- **Nouns, not verbs.** A capability is named as a noun phrase (Invoice Generation), never as an activity or process step; verbs are for processes and value streams.
- **No organisation labels.** A capability name does not mirror a department or team.
- **Every capability has a description** in business language, with explicit in-scope and out-of-scope boundaries.
- **Three levels, once only.** The BCM has three levels (L1 domains, L2 groups, L3 capabilities); every capability appears once, with one parent, and siblings do not overlap.
