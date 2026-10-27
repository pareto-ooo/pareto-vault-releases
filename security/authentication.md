---
id: authentication
scope: global
severity: critical
audience: all
schemaVersion: 1
---

# Security standards: Authentication

How users and services prove who they are to an application on the default stack: a Vue single-page app and a C# API in separate App Services, on Entra ID. These are the ground truth rules for the app registrations, the token flow between the browser, the API and other services, roles, local development and external users. The security principles behind them (single sign-on, token validation, short-lived tokens, multi-factor) are owned by `security` and not repeated here.

## Authentication philosophy

- **The API is the boundary.** Every request to the API carries an access token that the API validates. The single-page app holds no data of its own, so its static files are not what is protected.
- **The identity provider signs people in, the application never does.** The application has no login form, no password store and no session of its own.
- **One mechanism, one library per side.** MSAL in the browser, Microsoft.Identity.Web in the API. No hand-written token handling on either side.

## App registrations

- **Each application has two app registrations per environment:** one for the single-page app and one for the API, named `<app>-spa-<env>` and `<app>-api-<env>`. Environments never share a registration, so a token from one environment is never accepted in another.
- **The single-page app registration** uses the single-page application platform, with only the redirect URIs of its own environment. The development registration also holds the `localhost` redirect URI for local development. It has no client secret and no certificate.
- **The API registration exposes one delegated scope,** `access_as_user`, and defines the app roles. Its application ID URI is `api://<app>-api-<env>`. Only the single-page app registration of the same environment is pre-authorised for the scope.
- **App registrations are created and changed as `security` describes.**

## Sign-in from the single-page app

- **The single-page app signs in with MSAL Browser** using the authorization code flow with PKCE, against the company tenant for internal applications and against the external tenant for external users.
- **Sign-in is a redirect, not a pop-up.** Pop-ups are blocked too often to be the default.
- **Tokens are cached in session storage,** never in local storage, so they end with the browser tab and are never shared between tabs of different users.
- **The HTTP core acquires the access token** for the API scope silently before each request and attaches it as a bearer token. When a silent acquisition fails, it starts an interactive redirect. No other code touches tokens (`coding-standards-vuejs`).
- **Sign-out signs out at the identity provider,** through MSAL's logout redirect, after clearing the token cache.
- **The frontend App Service has no App Service authentication.** Edge authentication cannot hand the single-page app a token for a different App Service, and the API protects the data. App Service authentication remains the mechanism for an internal app without an API of its own, such as a server-rendered tool.

## The API

- **The API validates every token with Microsoft.Identity.Web:** signature, issuer (the one tenant of its environment), audience (its own application ID URI or client id) and lifetime.
- **A user token must carry the `access_as_user` scope,** and a service token must carry an application role. A token with neither is rejected, even when it is otherwise valid.
- **Authentication configuration is validated at startup.** A missing tenant, client id or audience fails the boot.

## Roles

- **Roles are app roles on the API registration,** never Entra group claims. Group claims are cut off above a size limit and leak the user's group memberships to the app.
- **App roles are assigned to security groups, not to people.** The groups follow the security group convention of `naming`. Assigning a person to a role is adding them to the group.
- **The API authorises on the `roles` claim** and then on the resource itself, as `security` requires. A role name is a stable string in code (`Customers.Read`), never a display name.
- **Role groups exist per environment.** Membership of a production role group never follows from a development one.

## Service-to-service calls

- **A calling service uses its user-assigned managed identity** to request a token for the receiving API's application ID URI. It never forwards a user's token and never uses a shared secret.
- **The receiving API defines an application role for each kind of access it allows** (`Orders.Read.All`), and that role is assigned to each allowed caller's managed identity. That assignment is the explicit allow-list `security` requires: a caller without the role is rejected.
- **The role assignments to managed identities are declared in the environment's access unit** with its other role assignments (`terraform`).

## Local development

- **Locally, the application signs in with the developer's own Entra account** against the development registrations, the same way the deployed development environment does. There is no authentication bypass, mock identity provider or fake token in any configuration.
- **Developers get development roles through the development security groups,** so a developer tests the same role checks the deployed app enforces.
- **The API running locally validates real tokens** for the development audience.

## External users

- **External users sign in through Microsoft Entra External ID,** in an external tenant of the organisation, never as guests in the company tenant.
- **The external tenant provides every sign-in method:** email with password or one-time passcode, and social or federated providers where the project definition records them. Sign-up, password reset and multi-factor run there, so the API has no login, sign-up or password-reset endpoints.
- **Multi-factor authentication is required for external users** of an application holding `Confidential` or `Restricted` data.
- **An application serving both employees and external users** registers in both tenants, and the API accepts each tenant as a separate issuer with its own roles.

## Review and ownership

- **Owners:** the vault owners maintain this document. The security owner signs off exceptions.
- **Another authentication mechanism is a security deviation** (`deviations`): it blocks the affected work until it is approved.
