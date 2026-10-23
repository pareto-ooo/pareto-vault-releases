---
id: security
scope: global
severity: critical
schemaVersion: 1
---

# Security standards

How we keep what we build secure. These are the ground truth rules for security in every language, service, pipeline and piece of infrastructure we write.

## Marked rules

A rule that exists because the company works to a regulation, a standard or an internal governance rule starts with its source(s) in brackets. These rules should be strictly enforced.

## Security philosophy

These principles are the catch-all rules. Every specific rule in this document, and every security finding in a review, is an application of one of them. A violation that no specific rule covers is still a violation of a principle, and is reported as one.

- **Never trust clients.** Everything that crosses a trust boundary is untrusted until the receiver has validated it, in the receiver's own runtime.
- **The server is the authority.** Anything running on a device we do not control (browser, mobile app, script) is a convenience. Every check that matters happens again on the server.
- **Deny by default, fail closed.** Access, features and network paths are off until explicitly granted. When a check cannot be completed (a missing setting, an unreachable identity provider, a failed validation), the answer is no, never a silent fallback to open.
- **Least privilege.** Every identity, person, service, token and network rule gets the minimum scope and duration that does the job, and no more. Broad grants need a recorded decision.
- **Check every time.** Authentication and authorization are verified on every request and for every resource. Nothing is trusted because an earlier step passed, because the caller is "internal", or because the request came from our own code.
- **Secure by construction.** Make the insecure path impossible or loud, instead of relying on people remembering. Prefer types, framework defaults, validated configuration and build failures over conventions and comments.
- **Minimise what exists.** Data we do not collect, keep, log or expose cannot leak. Features, endpoints, ports, dependencies, permissions and credentials we do not have cannot be attacked.
- **Secrets should be secret.** any form of leaking them is cause for rotation.
- **Use proven components.** No self-made cryptography, authentication, session handling or input sanitizing. Use the platform and reputable libraries, configured as intended.
- **No security off in lower environments.** The code path that runs in production is the one that runs everywhere. Security is never disabled "for development" or "for the migration step".
- **Keep it current.** Dependencies, base images and tools are attack surface. Known vulnerabilities are fixed on a schedule, not when convenient.
- **Think before building.** New trust boundaries, data flows or exposure are threat-modelled before the code is written.
- **No unapproved deviation.** Adopting any additions or deviations from the outlined rules needs consultation with the information security officer and the approval of the IT director, and additionally the enterprise architect if architectural change is involved. These are roles, not people: each project records who holds each role in its project definition before its first deviation. One person may hold several roles when the organisation has no separate holder, and the record says so.

## Authentication

- **SSO is the model, the Microsoft provider is the mechanism.** Single sign-on means an employee signs in once and reaches every internal application. The Microsoft provider is the platform's built-in Entra ID authentication in front of the application.
- **(ISO27001) Internal applications always use SSO** with the company identity provider. They have no separate login.
- **Internal web apps authenticate at the edge** with the Microsoft provider, so no application code runs before the user is authenticated. Signing in within the code (PKCE through a library) is only used where the platform provider is not available.
- **The application still authorises** what the authenticated user may do.
- **External applications do not use SSO.** Customers and other external users never sign in through the company identity provider. **TODO:** decide how external users sign in.
- **(ISO27001) Every request carries a token and the backend validates it**: signature, issuer, audience and expiry. The backend keeps no session state.
- **Tokens are short-lived.** Access tokens live about one hour at most. A long-lived token needs a recorded reason.
- **Refresh tokens rotate on every use and end with the session.** On logout the client discards every token it holds and ends the session at the identity provider. Where the identity provider can revoke the refresh token of one application, logout revokes it. Where it can only revoke every session the user has, as for a single-page app on Entra ID, the refresh token is left to expire, and its lifetime is never extended beyond the provider's default for that kind of client (24 hours for a single-page app on Entra ID).
- **(ISO27001) Every Pareto login is multi-factor, or machine to machine.** Employee logins go through the company identity provider, which enforces two-factor authentication wherever possible. No Pareto account is exempt, and no exclusion or "legacy" sign-in path is left open. Passwordless sign-in is preferred. This does not apply to external users of external applications.
- **Every account that has or can get access to business-critical or mission-critical workloads uses a hardware security token (a YubiKey).** The accounts that carry one are the dedicated ADC-#### accounts. Any other login mechanism needs two-factor authentication as well, and a recorded decision for why the company identity provider is not used. Non-human callers use machine-to-machine credentials (managed identity, federated or signed tokens), never a person's login.
- **(ISO27001) Public API keys are a machine-to-machine exception.** An API key is allowed only for a non-human caller that cannot use token-based authentication. It is issued per client, scoped to the minimum, stored hashed, revocable, rotated, rate limited and never used to sign in a person.
- **Anonymous endpoints are a documented exception.** Each one is explicitly marked as anonymous in code, declared with empty security in the API description, rate limited, strictly validated, and never reads or writes user-specific data. Health checks, marketing pages and sign-up are the usual cases.
- **Health checks are anonymous by design.** `/health/live` (the process runs) and `/health/ready` (it can serve requests) are marked anonymous so the platform can probe them. They return only a status, never version, configuration or dependency detail, and they are not rate limited, because the platform's probes would trip the limit.
- **(ISO27001) Authentication is never switched off or made conditional** on environment or startup mode. Its configuration is validated at startup so a missing value fails the boot instead of the first request.
- **(ISO27001) Service-to-service calls use managed-identity tokens.** Never an end user's token and never a shared API key. The receiver checks the audience and the caller's identity against an explicit allowlist.

## Authorization

- **(ISO27001) Deny by default.** Every endpoint requires an authenticated caller unless it is explicitly marked otherwise. Forgetting an annotation must fail closed.
- **(ISO27001) Authorize the resource, not just the endpoint.** A role answers "may you read customers", not "may you read this customer". The check names the specific resource and happens in the use case, on every read and every write.
- **(ISO27001)(GDPR) Tenant and ownership isolation is enforced in the data access path**, not left to each caller. In multi-tenant data, every query is filtered by tenant, backed by a second layer (a global query filter, or database row-level security, chosen by sensitivity).
- **Authorization failures are reported as forbidden** through the standard error handling, without revealing whether the resource exists to a caller who may not know.
- **Tests cover the forbidden path** as well as the allowed path, for every new authorization check.
- **A scoped role always has a recovery path.** When the last holder of a role scoped to a team, tenant or other group (a team lead, an owner) is gone, an operator moves the role to another existing member through an audited internal operation, guarded by its own app role. That app role grants no access to the scope's data.
- **A leaver loses access at the identity provider.** Disabling or removing the account in the company identity provider blocks every sign-in, so security never depends on the application's own clean-up. The application defines, per scoped role, what happens to what the leaver held: their open work is reassigned by whoever manages the scope, and a role that would be left without a holder goes through the recovery path.

## Transport security (TLS)

- **(ISO27001)(GDPR) All traffic is encrypted in transit**, including internal and service-to-service traffic. There is no plain HTTP, except a redirect to HTTPS.
- **(ISO27001) TLS 1.3 is the target, TLS 1.2 is the floor.** Our infrastructure requires TLS 1.3 wherever it can. Some third-party integrations cannot use it (for example callbacks from an external service), and for those the specific endpoint falls back to TLS 1.2, with the reason recorded and re-checked periodically. Nothing ever accepts anything lower. Other insecure services and protocols (SSH version 1, SMB version 1, NTLM version 1, WDigest) and weak ciphers are detected and disabled as well.
- **HTTPS-only is on for every web app**, and HTTP is answered with a permanent redirect. It is switched on as a follow-up change once the certificate is active, never in the same change that creates the domain binding.
- **Strict transport security headers are sent** on every web response.
- **Client-side certificate pinning never pins the leaf certificate**, because a leaf certificate rotates at every renewal. Pin the intermediate or root, or do not pin.
- **(Cloud Governance) Self-signed certificates are for internal use only, and wildcard certificates only for internal-facing websites and services.** Anything public-facing has a certificate for its exact hostname from a trusted authority.
- **(ISO27001) TLS certificate validation is never disabled** in code, configuration or tests, for any environment.
- **Database and storage connections require encryption** and reject unencrypted connections.

## Secrets and keys

- **(ISO27001) All secrets live in a managed secret store**, one per app per environment. Nothing is shared across apps or environments: two apps that need the same credential each hold their own copy.
- **Secrets reach the app as settings injected by the platform.** Code reads the setting and does not call the secret store in steady state nor are they held in memory unencrypted.
- **Rotation is a deploy, not code.** The owner adds a new secret version and the app is restarted. No in-process refresh timers exist.
- **A missing or unresolved secret fails the boot.** Every settings class is validated at startup, so a secret that did not resolve is caught before traffic. An unpopulated secret is never referenced.
- **There are no hardcoded fallbacks** for secrets or security settings. Only clear placeholder values if definition or placeholder is required to exist. A placeholder should also fail the boot.
- **A secret is anything whose leak would expose us.** An environment URL or feature flag is configuration. A staging API key is a secret.
- **(ISO27001) Secrets never appear in**: source code, configuration files, build variables that ship to a client, container build arguments or layers, command-line arguments, commit messages, test fixtures, logs at any level, or pipeline variables.
- **Test secrets are obviously fake** (`test-key-do-not-use`), never a redacted real one.
- **(ISO27001) A leaked secret is rotated.** The key has left the secure environment, this is irrevocable. Assume someone already stole it, or can access it in the future because of the leak.
- **Encryption keys are kept as keys in the key store, not as secrets**, and key material never leaves it.
- **(ISO27001) Local development never has production secrets.** It uses test values and third-party sandbox keys.
- **Credentials that do not need to exist do not exist.** Credentials that are not used need to be removed. Authentication paths we do not want to use are disabled when possible.

## Cryptography

- **(ISO27001) No custom cryptography.** Use the platform's libraries and managed services, with their recommended and proven algorithms. For infrequent requests a Key Vault Encryption Key has preference through Azure CryptographyClient.
- **Passwords are never stored.** Where a credential must be verified, we use an established library or the platform's built-in password hasher with the latest recommended encryption standards. The hashing settings are revisited when the standard moves, and stored hashes are upgraded when a user next signs in.
- **Random values for security purposes** None cryptographically secure hashes are never used as secrets or capability tokens.
- **Weak algorithms are not used**: Always use the latest recommended algorithms and up-to-date security practices preferably the ones presented by the platform library tools.
- **(ISO27001) Data at rest is encrypted by default** with platform-managed encryption on every database and storage account. Customer-managed keys are used only where a requirement calls for deeper control of the keys.

### Persistent Storage

- **(ISO27001)(GDPR)(Cloud Governance) Backups are automated, protected, monitored and tested for restore.** Business-critical resources are backed up regularly and automatically. Backups are encrypted and access-controlled, with deletion and retention changes limited and audited, so ransomware or a malicious insider cannot remove them. Backup health is monitored, and restores are tested against the recovery time and recovery point set by the business criticality of the workload. A backup that has never been restored is not a backup.
- **(ISO27001)(GDPR) Production data is never copied into lower environments.** Test data is synthetic or properly anonymised.

## Azure

Rules that only hold on our Microsoft Azure platform. The general rules in the other chapters apply here as well.

### Identity (Entra ID)

- **(ISO27001) The company identity provider is Microsoft Entra ID (OIDC).** It is used for Pareto employees and internal applications only; external users are covered under Authentication.
- **(Cloud Governance) App registrations are created by developers with the built-in Application Developer role.** The Application Administrator role is granted to no one, users cannot consent to applications on their own, and tenant-wide admin consent (for example when deploying to acceptance or production) is requested through the identity team.
- **(ISO27001) No standing privileged access.** Subscription owners, and contributors on subscriptions that hold production, activate their role just in time through privileged identity management. Highly privileged tenant roles use separate accounts with maximum protection.
- **(ISO27001) Access is reviewed every three months.** Each team reviews who has access to its resources, removes what is no longer needed, and records the review in the ticketing system. Access changes are recorded as requests there as well.

### Roles and access (RBAC)

- **(ISO27001) One runtime identity per app per environment**, a user-assigned managed identity, distinct from the single pipeline identity and from any background-work identity. Distinct identities give distinct audit trails, revocation and blast radius.
- **(ISO27001) Role assignments are scoped to the specific resource**, never the resource group or subscription, and use built-in roles. Broad scopes are for organisation-level roles and the bootstrap identity only.
- **(ISO27001) Role assignments are declared in Terraform** and executed by the delegated admins through the batched change request. No ad-hoc grants in the Portal.
- **(ISO27001) `Owner` and subscription-level `Contributor` are never granted** to workload identities. A developer account never gets rights to act as a production identity, outside break-glass.
- **(ISO27001) Authentication to Azure from pipelines is keyless** through workload identity federation. No service principal secrets and no personal access tokens. Pipeline identities cannot create role assignments.
- **A service principal is the exception.** It is used only where a service supports neither managed identity nor federation. It is scoped to the one resource, and uses a certificate credential.
- **Roles are assignable to groups.** Roles scoped to the specific resource, can be assigned to a group for easy management of multiple individuals that need the same role.

### Secrets (Key Vault)

- **(ISO27001) Secrets and encryption keys live in Azure Key Vault**, one vault per app per environment, in RBAC authorization mode.
- **Secrets reach the app through Key Vault references** in the app settings. An unresolved reference arrives as a literal placeholder string, so startup validation must turn it into a failed boot.

### Certificates

- **Certificates are platform-managed.** Custom domains use App Service managed certificates and never an uploaded PFX. Managed certificates cover exactly one hostname, so each hostname gets its own binding and certificate.
- **DNS records stay in place** for the lifetime of the app, because certificate renewal re-verifies them and fails silently when they drift. An availability test with a certificate-lifetime check raises an alert before expiry.
- **Bring-your-own certificates are an exception.** They live in Key Vault with an integrated certificate authority so renewal stays automated, and a manual rotation cadence is documented where it is visible.

### Data services

- **Database access is a contained database user for the app's identity**, never `db_owner`.
- **The database firewall is default-deny.** Each app opens only its own outbound addresses, and operator access is a personal, temporary rule that is removed after use.
- **Storage containers are never publicly accessible.** Public content is served by an app with explicit configuration.
- **Irreplaceable data is protected from accidental destruction** by the two-layer guard (Terraform lifecycle protection and a resource lock).

### Logging and monitoring

- **(Cloud Governance)(ISO27001) Platform logs are routed with diagnostic settings.** The Azure Activity Log is deleted by the platform after 90 days, so it is routed to a Log Analytics workspace for up to a year, or to storage for longer.

## Input handling

- **(ISO27001) Validate at every boundary, in the receiver.** HTTP requests, queue and event payloads, timer-triggered calls, uploaded files and database columns filled by external systems each validate their own input. The same data model reused by another entry point is validated again there.
- **Allow-list, never block-list.** Define what is valid and reject everything else.
- **Type first.** Bind to the narrowest type that fits (a GUID, a date, an enum, a number) so malformed input is rejected before any logic runs. Route and query parameters are user input.
- **Every string and every number has bounds.** An unbounded field is a denial-of-service and storage problem. Maximum lengths and ranges are always set, and align with the database column limits.
- **(ISO27001) Output is encoded for its destination.** Data placed in HTML, SQL, a shell, a URL, a header, a file path or a log is encoded or parameterized for that context, never concatenated.
- **No dynamic execution of input.** No `eval`, no shell commands built from input, no deserialization of untrusted data into arbitrary types, no loading of code or templates named by input.
- **Paths and URLs from input are constrained.** A file path is resolved and checked to stay inside its allowed root, and an outbound URL fetched on a user's behalf is restricted to an allow-list, so it cannot reach internal addresses (SSRF).
- **Payload size limits are always set** at the server and the gateway. Request bodies, uploads, collection sizes and recursion depth are all capped.

### File uploads

- **Size cap, and an allow-list of types.** The expected types are finite, everything else is rejected.
- **Verify content, not just the declared type.** Check magic bytes.
- **Scan for malware** any file that leaves the trust boundary (forwarded to a third party or downloaded by another user). //TODO is this viable?
- **Store in blob storage with generated names.** Never use the client-supplied file name for storage, and never write uploads to local disk beyond the request lifetime.

### Events and messages

- **Subscribers validate the payload against the published schema.** A non-conforming message goes to the dead-letter queue for review and does not crash the subscriber. It is a data quality issue, not a transport failure.
- **Internal endpoints validate both the caller's token and the body.** Coming from our own function is not a reason to skip either.
- **Handlers are idempotent**, because messages are delivered at least once.

## Logging and audit

- **Log identifiers, not values.** A customer GUID is fine, a customer email is not. When in doubt, log the identifier and look the value up elsewhere.
- **(ISO27001)(GDPR) Never log**: personal data (names, emails, phone numbers, addresses, dates of birth, government IDs), authentication material (passwords, any token, cookies, signed URLs), cryptographic material, payment data, health data and other special-category data, business-sensitive figures, and full request bodies.
- **Structured logging with named fields.** No string interpolation into messages. It keeps fields queryable and removes the temptation to add personal data "for context".
- **Free text is dangerous.** User-provided text, search terms and error messages that echo input are logged as a length, a fact or a hash, never as content.
- **Redact at the boundary** when a value must appear (a third-party error that embeds input). Redaction is the second defence. The first is not logging it.
- **(ISO27001)(GDPR) Security events go to a dedicated audit stream** kept at least two years and then purged: authentication success and failure, authorization failures, privilege and role changes, and access to sensitive data (exports, bulk reads). The stream is append-only, and nothing in the application changes or deletes audit lines.
- **Production log verbosity is Information at most.** Debug is enabled only for an incident under controlled access, and then switched back.
- **No temporary debug logging of objects.** It ships, and a personal-data tap stays open for the hour it takes to notice.
- **(ISO27001) Log access is itself controlled and recorded**, scoped per app, and used only for an incident or a documented reason.
- **(Cloud Governance)(ISO27001) Security logs go where the company monitors them.** Platform logs are routed to a store we control, because the platform deletes some of them itself. The log categories that the information security officer makes mandatory are delivered to the central security monitoring service.
- **(Cloud Governance)(ISO27001) Time comes from approved sources.** Timestamps carry the date, time and time zone (UTC), and compute resources use the platform's default time server unless there is a specific requirement.
- **(GDPR) Client-side logs are not shipped automatically.** A diagnostics report is explicitly user-initiated, shows the user what is sent, and excludes form input and page contents.

## Dependencies and supply chain

- **(Software Governance) Packages come only from official registries** (NuGet, npm, PyPI, or the company's approved feed). Never from a third-party site, a downloaded binary, a link, a raw git address or a tarball. Package sources are configured explicitly in the repository, so a restore cannot silently pull from anywhere else, and unauthorised ways of retrieving software are blocked. The same holds for container base images: official publishers only.
- **(Software Governance) Integrity is verified.** Signed packages are verified where the ecosystem supports signatures, and the lockfile pins a content hash (SHA-256 or stronger) that is checked on every restore. A mismatch fails the build. MD5 is not an integrity check.
- **(Software Governance) Versions are controlled.** Versions are pinned or tightly bounded by semantic versioning, never `latest` or an open range, and lockfiles are committed, with lockfile changes matching the dependency change in scope.
- **(Software Governance) Patch and minor updates are applied promptly**, as a change that contains only the dependency update with the tests run, so it is easy to roll back. **Major upgrades are planned**: follow the maintainers' upgrade instructions, keep the change isolated from the regular flow, and validate it in a dedicated acceptance environment before production.
- **(ISO27001)(Software Governance) Dependencies are kept up to date.** Outdated packages are detected by automated update pull requests and scheduled scans for outdated versions and best-practice findings, which run regularly. Their results are reviewed, and a package is not left behind because nobody looked.
- **(ISO27001) Runtimes, frameworks and base images stay on a supported version.** An end-of-life runtime, framework or base image is treated as a vulnerable dependency, because it no longer receives security fixes. Its upgrade is planned before support ends.
- **(ISO27001) A known vulnerability in a deployed dependency is fixed within a set time**, by severity, counted from when it is published or found. The time limit applies to code already in production, where the merge gate does not reach. Remediation is risk-based and automated where possible: a more severe vulnerability in a more critical workload is fixed first.
- **(ISO27001) Findings from penetration tests and audits about outdated or vulnerable components are handled like scan findings**: recorded with an owner, triaged on reachability, fixed or accepted within the deadline, and the fix is checked on the retest.
- **(ISO27001)(Cloud Governance) Vulnerability scanning runs on every pull request** and daily on the main branch, since a newly published vulnerability affects code that has not changed.
- **(ISO27001) Critical and High findings block the merge.** Medium surfaces in the pull request for the reviewer, and Low and informational are reviewed periodically. There is no merge-anyway option, and the gate is never bypassed (`continueOnError`).
- **Triage is analysis, not assumption**: read the vulnerability, check whether the vulnerable path is reachable in this app, then upgrade, work around, or replace. Risk acceptance is a last resort and is recorded with the identifier, the reachability analysis and a re-check date. Allowlist the specific vulnerability, never the whole library, and review the list during new phase planning.
- **Test-only dependencies are at most Medium**, unless the test host runs production code.
- **Do not switch a base image to dodge a finding.** Bump to the patched tag of the same base.