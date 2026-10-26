---
id: saas
scope: global
severity: warning
applies: all
schemaVersion: 1
---

# Architecture standards: SaaS applications

How we select and implement bought business applications delivered as Software as a Service. These are the ground truth rules for the "buy" step of the principle "reuse before buy before build". Tools and add-ons that do not directly support a business process follow the software governance rules instead.

## SaaS philosophy

- **SaaS before PaaS before IaaS.** A fully managed application lets the vendor carry infrastructure, maintenance and support, so we focus on the business need.
- **Reuse before buy.** A documented business need is first checked against the applications we already own; buying only follows when they cannot meet it.
- **Buy what the business needs.** Only the features and licences needed now are bought, with the future in mind, not every available add-on.
- **Proven technology and vendors.** We are not the first customer; the product and vendor are proven in comparable organisations.
- **Configuration over customisation.** The application is used out of the box and configured; a missing feature is first requested from the vendor. Custom development needs a positive business case approved by ICT management.

## Selection

- **The purchase follows a structured path:** documented business need, stakeholders and budget, longlist, evaluation (information requests, trials), shortlist (proposals, demonstrations), decision and negotiation.
- **Public (shared) SaaS over private.** A private, dedicated offering is only chosen when security, compliance or customisation requires it.
- **Monthly billing over yearly** for flexibility; prepaying is considered only for flat, predictable usage.
- **English is available** for the application, documentation and training, at no extra cost.
- **Industry-standard integrations and reports are available out of the box,** and are demonstrated during the shortlist phase.

## Identity and security

- **Single sign-on with Entra ID.** The application supports federated identity with Entra ID, so multi-factor authentication and conditional access apply and on- and offboarding are managed centrally.
- **Role-based authorisation through groups.** Access is granted through Entra ID security groups; automatic user provisioning uses SCIM where the vendor supports it.
- **Encrypted in transit and at rest,** with robust vendor security practices (for example ISO 27001) and openness to independent security assessments.

## Data

- **We own our data.** The data can be exported in a usable format at any time and at the end of the agreement, followed by its deletion.
- **Single source of truth.** For every record there is one source application; the SaaS application is either that source or integrates with it.
- **Business intelligence access.** The data can be delivered to the business intelligence platform, preferably incrementally (changes since a given moment).
- **Data is hosted in the EU** when compliance or regulation requires it.
- **Backups and disaster recovery** are provided by the vendor and described in the agreement.

## Integration and mail

- **Integrations with our own applications** follow the same rules as in-house integrations; integration that needs custom development is arranged with ICT Development.
- **Mail to customers goes through our central mail delivery service.** A SaaS application that sends mail from our domain connects to it as its SMTP server or through its API; it never sends customer mail through its own mail service in our name.
- **Mail delivery is centralised on purpose.** One provider gives every application the same mail templating and throttling and one trusted, allow-listed mail server, and it keeps the number of parties that can send mail in our name small.
- **Internal mail uses Microsoft Exchange.** An application that only mails employees sends through Exchange with a service account of its own.
- **The vendor's mail setup is checked during selection.** An application that must send customer mail from our domain but cannot relay through the central mail delivery service is a deviation, signed off by an architect.

## Operation

- **Every application has a business owner** known to ICT, responsible for billing and administrative questions.
- **Business-critical and mission-critical applications** also have key users and a product owner.
- **Support is available in our time zone (CET/CEST) and in English.**
- **Updates are planned and announced** by the vendor with a roadmap, take place outside business hours, and a test environment is available for larger upgrades.
- **The agreement (SLA) records** contacts and escalation, uptime and penalties, performance, scalability, backup and recovery with RTO and RPO, data extraction and deletion, security and incident response, support levels, service reviews and the update process. When a vendor offers no negotiable SLA, ICT still records these points from the vendor's published terms.
- **Compliance is part of the agreement:** GDPR (including a data protection impact assessment where personal data is processed), NIS2, data breach notification without undue delay, retention, and the right to audit.
