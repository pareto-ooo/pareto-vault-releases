---
id: api
scope: scoped
severity: warning
audience: all
applies-to:
  - "**/*.Api/**/*.cs"
  - "**/*Controller.cs"
schemaVersion: 1
---

# API standards

The rules for the HTTP APIs our services expose, for app-to-app integration and for frontend consumption alike.

## Security

- **Every endpoint declares its authentication and authorization.** The API description states the security requirement per operation.
- **Errors use one uniform shape** (problem+json with a stable `code` and `traceId`) and never contain stack traces, SQL, hostnames, secrets or personal data.
- **Resource identifiers in public APIs are GUIDs.** Sequential or database-generated identifiers are never exposed, so resources cannot be enumerated.
- **Request bodies bind to dedicated input types**, never directly to entities, so a client cannot set fields it should not (mass assignment). Responses are dedicated output types, never entities, so nothing leaks by accident.
- **Rate limits apply per principal, and per address for anonymous endpoints.** The response is 429 with `Retry-After`. Expensive endpoints are limited tighter than cheap ones. Where the limit is enforced (API Management or the app) is described under Rate limits.
- **Lists are paginated with a maximum page size**, so a single call cannot dump a table.
- **State-changing operations that clients may retry are idempotent** (idempotency key), which also blocks replay.
- **CORS is an explicit allow-list of origins.** Never a wildcard with credentials.
- **Only the HTTP methods an endpoint needs are enabled.** Verbs are used for their meaning (no state change on GET).
- **Security headers are set on responses**: `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, a restrictive `Content-Security-Policy` for anything serving HTML, and a framing restriction. Server and framework version headers are removed.
- **Webhooks we receive are verified** by signature and timestamp before processing, and webhooks we send are signed.
- **Documentation and diagnostic endpoints are not exposed in production** unless they are authenticated. Health checks reveal liveness only, and never version, configuration or dependency detail to anonymous callers.
- **Cursors carry position, not permission.** Authorization and tenant filters are applied again on every page query.

## Controllers and DTOs

- **A controller action is validate, call service, return DTO.** Three lines or fewer in most cases. If an action needs more than five or six lines, the missing pieces belong in the service.
- **No business logic, no try/catch, no transaction boundaries and no adapter-type mapping in a controller.** An `if` in an action is a prompt to ask whether the rule belongs in the service.
- **Exceptions are mapped to HTTP in exactly one place**, the exception handler. A controller never catches.
- **One controller per resource.** `CustomerController` does not also serve `/orders/...`.
- **Naming.** The class is `<Resource>Controller` (singular noun), with a class-level `[Route("v<n>/<resource>")]`. Actions are named after the domain operation (`Get`, `Create`, `Archive`), not the verb (`GetCustomer`, `PostCustomer`).
- **DTO names.** `<Resource>Response`, `Create<Resource>Request`, `Update<Resource>Request`, and `<Resource>Summary` for list views. A version-specific shape gets its own type (`CustomerV2Response`).
- **Request and response DTOs are separate records**, never EF Core entities and never upstream adapter types. Serving an entity ships a navigation graph, leaks internal columns such as `created_by` and row versions, and turns every entity rename into a breaking change for consumers.
- **A response DTO may carry derived fields the client needs** (`lineCount`, `total`) instead of mirroring the entity's columns.
- **Validation attributes (DataAnnotations) sit on the request DTO** in the API project. `[ApiController]` triggers them at model binding, and manual `if (...) throw` validation in an action is banned.
- **Return the DTO type directly where the shape is fixed**, rather than wrapping everything in `IActionResult`. Actions that create a resource use `CreatedAtAction` / `AcceptedAtAction`, which set `Location`.
- **No `Dictionary<string, object>` or anonymous objects as a response.** A shape without a type has no schema, no OpenAPI and no client typing.
- **GUID parameters bind as `Guid`, without a route constraint,** so a malformed value fails model binding and is rejected with a 400 before the action runs. A `{id:guid}` constraint would turn it into a 404, because the route no longer matches.
- **Never read the request body twice.** Buffering is a middleware-level decision with a documented reason, never an in-action workaround.
- **Cross-cutting behaviour outside the framework's own is a marker attribute plus a globally registered action filter**, or a DI decorator for service-level concerns. Middleware applies to every request, an attribute and filter applies per action, and a decorator wraps specific service methods.
- **Every HTTP request is handled synchronously** within the request scope, until there is a concrete reason not to (work that fans out to consumers unknown to this app, outlives a reasonable timeout at the 95th percentile, depends on an untrusted third party, or is scheduled). Deferred work is answered with 202.

## Routes and versioning

- **Public routes are versioned in the URL path from day one**: `/v<n>/<resource>`. There are no unversioned public endpoints.
- **Major versions only.** No `/v1.2/`. Minor and patch changes are additive within the major. Major version bump is indicated by SemVer 2.0.0, meaning "breaking changes" are introduced.
- **Header-based and date-based versioning are not used.** The version stays visible in logs, traces, caches and the address bar.
- **Paths are plural `kebab-case` collections**: `/v1/customers`, `/v1/customers/{id}/orders`, `/v1/customer-tags`.
- **No verbs in paths.** An action endpoint (`POST /v1/customers/{id}/archive`) is allowed sparingly, when the action is genuinely orthogonal to the resource lifecycle. Otherwise model it as a state change through PATCH.
- **Internal endpoints live under `/internal/...`**, are unversioned, and are excluded from the public OpenAPI document. They serve operations that a person runs, such as an owner running a GDPR erasure, and the background workers of the same deployment boundary where a worker needs one.

### Breaking and additive changes

- **A breaking change bumps the major.** Breaking is: a removed or renamed response field, a changed field type, a removed endpoint, a required request field added without a default, a tightened validation (a stricter regex, a narrowed enum), and a changed behaviour (default sort, error shape, status code on an edge case).
- **Everything else is additive and stays on the same major**: a new endpoint, a new response field, a new optional query parameter or body field, a required body field with a default, a widened type, a new error `code`, a new status in a 4xx slot.
- **A breaking change is never shipped without a version bump.**
- **Feature flags are not versions.** A flag rolls out behaviour without changing the schema. A flagged change that alters the response schema is a major bump regardless of the flag.
- **Two majors coexist in one service.** Separate controllers (`CustomerV1Controller`, `CustomerV2Controller`) delegate to the same service layer, with the same persistence, and each major has its own OpenAPI document. Three majors in production is the threshold to start retiring the oldest.

### Deprecation

- **Announce first.** The timeline is 90 days minimum, and 180 days for APIs used across teams.
- **Deprecated paths respond with a `Deprecation` header holding the date of deprecation (`@1688169599`, RFC 9745), a `Sunset` date and a `Link: <...>; rel="successor-version"` header.**
- **Usage of a deprecated path is tracked** with a `deprecated_path` structured log field, so the migration is measurable.
- **Sunset is 410 Gone** pointing at the successor, followed by removal (404) after a 30 day grace period. A path is never removed without warning.

## Naming and JSON shape

- **JSON keys are `camelCase`.** Never `snake_case` or `PascalCase`, and never mixed (`customerID`, `Id`, `_id`).
- **Query parameters are `camelCase`**, URL path segments are `kebab-case`, and headers take the standard HTTP form. A custom header is prefixed `X-Pareto-` and is rarely needed.
- **The naming policy is configured once**, platform-wide, on System.Text.Json. There are no per-DTO naming attributes.
- **Field names have fixed shapes.**

  | Concept | Field name |
  |---------|------------|
  | Identifier | `id` |
  | Reference to another resource | `<entity>Id` (`customerId`) |
  | Timestamp | `<verb>At` (`createdAt`, `paidAt`) |
  | Boolean | `is<Adjective>` / `has<Noun>` (`isActive`, `hasOpenInvoices`) |
  | Counter | `<noun>Count` (`orderCount`) |
  | Currency | `currency`, an ISO 4217 string (`"EUR"`) |
  | Status | `status`, not `state`, `mode` or `phase` |
  | Collection | plural (`items`, `tags`, `addresses`) |

- **Booleans are positive.** `isActive` over `isInactive`, `hasMore` over `noMore`.
- **Be specific.** A field called `data`, `meta` or `info` is a smell. Name what is inside.
- **Timestamps are RFC 3339 in UTC with milliseconds** (`2026-04-01T15:23:48.123Z`), with no offsets. Dates are `2026-04-01`, times of day are `14:30:00`, and durations are ISO 8601 (`PT30M`), or integer `durationSeconds` where an ISO duration would surprise consumers.
- **Locale formatting is the client's job.** The API ships ISO values.
- **Money is a string decimal** (`"199.99"`) with a sibling `currency`. Other decimals where precision matters are strings as well. Counts, ages and quantities are JSON numbers.
- **IDs are always strings**, even numeric ones. The client treats them as opaque and never generates them, the server does.
- **Enum values are stable `snake_case` names** (`"active"`, `"partially_refunded"`), mapped from the stored number at the boundary, and query values use the same names. A name is a contract like the stored number: it is never renamed, removed or reused.
- **Flat by default, nested for genuine sub-objects.** An address or an amount with a varying currency stands alone as a value the client handles as a unit, so it is nested. Do not nest only to group flat fields (`audit: { createdAt, updatedAt }`).
- **Absent values are consistent per endpoint.** Either the field is always present with `null`, or it is omitted. The semantics are documented, never mixed.
- **A response never contains** internal non-GUID identifiers, framework handles or type-discriminator hints that are not part of the contract.
- **One set of envelope names**, never reinvented per endpoint:

  | Field | Where | Meaning |
  |-------|-------|---------|
  | `items` | Paginated list | The page's rows |
  | `page` | Paginated list | Pagination metadata |
  | `results` | Batch response | Per-item outcomes |
  | `code`, `traceId` | Error response | Stable identifier and request correlation |
  | `validationErrors` | 400 error response | Field-level errors |

## HTTP methods and status codes

Pick the status from what actually happened. Custom 4xx codes (450, 477) are not used, the `code` extension disambiguates.

| Status | Use for |
|--------|---------|
| **200 OK** | Reads, PUT and PATCH returning the updated resource, batch endpoints (including partial success) |
| **201 Created** | A POST (or PUT upsert) that synchronously creates a resource. Body is the new resource |
| **202 Accepted** | A request that enqueues deferred work. Body is a handle (`{ id, status: "queued" }`) |
| **204 No Content** | DELETE of a single resource. Never with a body |
| **400 Bad Request** | A malformed request: JSON that does not parse, DataAnnotations failures, type mismatches |
| **401 Unauthorized** | Not authenticated. Sent with a `WWW-Authenticate` header |
| **403 Forbidden** | Authenticated, but not allowed to do this |
| **404 Not Found** | The resource does not exist (a domain `*NotFoundException`), or the route is not mapped |
| **409 Conflict** | The current state precludes the request: a duplicate, an optimistic-lock failure, "order already paid" |
| **410 Gone** | An intentionally sunset endpoint or resource |
| **422 Unprocessable Entity** | A well-formed request rejected by a domain invariant |
| **429 Too Many Requests** | A rate limit fired, and also load shedding |
| **500 Internal Server Error** | An unhandled exception. This is a bug |
| **502 Bad Gateway** | An upstream dependency answered with a 5xx |
| **503 Service Unavailable** | An intentionally drained service (maintenance). Most deploys are zero-downtime, so not a reflex |
| **504 Gateway Timeout** | An upstream call exceeded this service's timeout |

- **`Location` is mandatory on 201 and 202.** It points at the new resource, or at the status resource to poll.
- **A create endpoint ships with the endpoint its `Location` points at**, in the same change. A `Location` that answers 404 is never released.
- **PUT upsert is 201 when the resource was created and 200 when it was updated.** DELETE that triggers deferred work is 202 with the handle.
- **400 versus 422.** If the client sent a syntactically valid request, it is 422. If not, it is 400.
- **409 versus 422.** 409 is for a request that would succeed but the current state precludes it. 422 is for a request that violates an invariant regardless of state ("ship date is before order date"). When both apply, prefer 409.
- **401 versus 403.** 401 means log in or present a token. 403 means asking for permission, and logging in again does not help. A 404 never stands in for a 403, except where a documented policy says the existence of the resource must not leak.
- **Empty collections are 200, never 404.** A list endpoint with zero items is `200` with `{ "items": [], "page": {...} }`. 404 is reserved for a missing parent (`GET /customers/<unknown>/invoices`). A known parent without children is a normal business state.
- **DELETE of an unknown resource is 404**, not a silent 204. A 200 with a `null` body is never used for "not found".
- **Partial success is not an error.** A batch endpoint that succeeds for some items returns 200 with a per-item `results` array (embedding a problem body for failed items). It does not use 207, and it does not use a 4xx. The batch is either accepted (200) or rejected (4xx).
- **5xx means a bug**, and is never the expected response of a user-facing operation. 500 is not "validation failed" and 503 is not "you may not see this". An unhandled error is never mapped to a friendly 400.
- **A body never carries an error under a 200** (`{ "error": "..." }`). A 204 never has a body.
- **Replaying an idempotent request returns the original status** (the original 201), never a third "duplicate" answer.

## Errors

- **Every error is `application/problem+json`** (RFC 9457) with `type`, `title` and `status`, plus the required extensions `code` and `traceId`, and `detail` and `instance` recommended. This includes errors generated by model validation and errors raised at the framework boundary.
- **Clients match on `code` (or the `type` slug)**, never on `detail` or on `status` alone. `type`, `title`, `status` and `code` are stable across versions. `detail` is free text that may be reworded or localised.
- **`code` is a `SCREAMING_SNAKE_CASE` identifier** (`CUSTOMER_NOT_FOUND`). Distinct error classes get distinct codes, and a code is never overloaded. A retired code is retired forever.
- **`traceId` is the W3C trace-id** of the request, which Application Insights records as the operation ID.
- **The `type` URI is `https://<error domain>/<slug>`,** on an error domain the project records in its project definition (`project-records`). The same domain is used in every environment, so a `type` never changes between them. Nothing user-facing ships before the domain is recorded.
- **Every `code` and `type` is registered in the service's error catalog** (`docs/error-catalog.md`) before it appears in a response. The catalog holds the code, the type slug, the status, the title and a paragraph on what it means and how to recover. New error classes are added by pull request.
- **No bespoke error shapes.** Not per endpoint, not per service, and no free-form `error: "..."` body.
- **The default 500 response carries a generic `detail`** (`An unexpected error occurred. Reference: <traceId>.`). The real cause is in the log, and the response never carries an internal path, a class name or the exception message verbatim.
- **Expected failures say only roughly what was wrong with the input**: the field and the rule it broke (`validationErrors`), or the stable `title`. They never explain internals, existence of resources the caller may not know about, or why the system decided so, because an error body is an attack surface.
- **Errors are never localised at the API.** `title` and `detail` are a stable contract, and localisation happens in the client, keyed by `code`.
- **Use `Microsoft.AspNetCore.Mvc.ProblemDetails`** with `code` and `traceId` as extensions. A custom error DTO is not used.
- **One exception handler chain per service**, registered in one place: a domain handler and a fallback handler (`IExceptionHandler`, `AddProblemDetails()`, `UseExceptionHandler()`). It maps directly. It does not catch and rethrow a wrapped exception.
- **`code` and `type` per exception class come from one resolver**, sourced from the error catalog.
- **401 and 403 from the authentication and authorization middleware never reach the exception handlers.** The middleware short-circuits, so its challenge and forbid events (`JwtBearerEvents`) emit the same problem shape. A resource-level refusal raised in a use case (`security`) is a domain exception that the domain handler maps to 403.
- **The handler logs 4xx at Information** without a stack (bad input is normal traffic) and **5xx at Error** with the full stack. The correlation scope gives every line the `traceId`, so a client-reported `traceId` finds the log line.

### Mapping of non-domain failures

| Failure | Status | `code` |
|---------|--------|--------|
| Model-state validation failure | 400 | `VALIDATION_FAILED` |
| `BadHttpRequestException` or `JsonException` during binding | 400 | `MALFORMED_REQUEST` |
| Route or binding type mismatch (a non-GUID id) | 400 | `MALFORMED_REQUEST` |
| Query parameter that does not parse | 400 | `MALFORMED_QUERY_PARAM` |
| Failed authentication | 401 | `UNAUTHENTICATED` |
| Failed authorization | 403 | `FORBIDDEN` |
| `DbUpdateConcurrencyException` | 409 | `CONCURRENT_UPDATE` |
| `DbUpdateException` of a recognized constraint | 409 | `CONFLICT` |
| Any other `DbUpdateException` and any unhandled `Exception` | 500 | `INTERNAL_ERROR` |

### Validation errors

- **Model-state failures carry a `validationErrors` array**, produced by one `InvalidModelStateResponseFactory` per service.
- **Each entry has `field`, `code` and `message`.** `code` mirrors the validation attribute's class name without the `Attribute` suffix, for built-in and custom attributes alike (`EmailAddress`, `Required`, `StringLength`, `[PostalCode]` becomes `PostalCode`), `message` is the attribute's English message, and clients localise off `code`. `field` uses dotted paths for nested DTOs (`address.postalCode`).

## Filtering and sorting

All list endpoints share one query-parameter style, with `camelCase` names.

| Concern | Style |
|---------|-------|
| Field equality | `?status=active` |
| Several values of one field (OR) | `?status=active&status=pending` (repeated parameter) |
| Several fields (AND) | `?status=active&tier=gold` |
| Range, exclusive | `?createdAfter=...&createdBefore=...` |
| Range, inclusive | `?createdFrom=...&createdTo=...` |
| Free-text search | `?q=pareto` |
| Sort | `?sort=-createdAt,name` (`-` prefix is descending) |
| Pagination | `?limit=50&cursor=...` |
| Field projection | `?fields=id,name,createdAt` |
| Total count | `?includeTotal=true` |

- **Repeated parameters, never comma-separated or bracketed values.** A comma collides with values that contain one, and `?filter[status]=` is not used.
- **`<field>After` and `<field>Before` are strict**, so adjacent ranges do not overlap on the boundary. An inclusive bound is named `<field>From` and `<field>To`.
- **Filters combine with AND.** There is no generic OR across fields and no `?filter=` JSON blob. A filter that needs an OR tree is a search endpoint (POST with a body).
- **`q` searches the fields the endpoint decides.** The client does not choose the searched columns, and the server normalises casing.
- **Sort is documented per endpoint.** The default is `-createdAt` (alphabetical for catalogs). The server appends the primary key as a tie-breaker when the sort is not unique. An unknown sort field is a 400 with `code: INVALID_SORT_FIELD`.
- **Sortable and filterable fields of user-facing endpoints are backed by an index.**
- **Field projection is optional per endpoint**, added only where a list genuinely benefits (admin tables, slow mobile clients). Always-required fields stay in the response.
- **Values are strict.** Booleans are `true` or `false` (never `1` or `yes`), dates are ISO 8601, and a value that does not coerce is a 400 with `code: MALFORMED_QUERY_PARAM`.
- **A malformed or unknown filter is never silently ignored.** A typo'd field returns a 400, not a 200 with unfiltered or empty results.

## Pagination

- **Cursor pagination is the default**, for public read APIs and any growing list (orders, events, log entries).
- **The response is always an envelope** `{ "items": [...], "page": {...} }`, never a bare array. `items` is always an array and `page` is always present.
- **The cursor shape is `{ "limit": 50, "nextCursor": "...", "hasMore": true }`.** `nextCursor: null` means the last page.
- **A cursor is opaque.** It is base64-URL encoded, clients never parse it and only echo it back. It holds the sort fields plus the primary key as tie-breaker, a schema version tag, and the active filter. Cursor decoding lives at the controller boundary, and a malformed cursor is a 400.
- **A cursor from one filter cannot be used with another.** The mismatch is a 400 with `code: CURSOR_FILTER_MISMATCH`.
- **Filters apply before pagination**, and the cursor is taken from the filtered result set.
- **Cursor sort order is explicit and stable**, `created_at desc, id desc` by default. Without the primary key as tie-breaker, rows are skipped or duplicated at page boundaries. A random order (`newid()`) is incompatible with cursors. Every paginated endpoint documents its sort in OpenAPI.
- **Limits are bounded.** Default 50, maximum 200, minimum 1, configurable per app. Larger bounds (1000) are allowed only for streaming exports, never for user-facing lists. A client asking for more than the maximum receives the maximum, not a 4xx.
- **Totals are opt-in** (`?includeTotal=true`), because computing one is `O(rows)`. The server may cap it (`10000+`).
- **Offset pagination (`?page=3&pageSize=50`) is allowed only** for bounded, internal lists that do not grow (admin views, reports, configuration tables) of under roughly 10,000 rows, where the user benefits from jump-to-page. Its metadata is `page`, `pageSize`, `totalPages` and `totalItems`.
- **One shape per endpoint.** `page`, `pageSize` and `cursor` are never mixed.

## Idempotency

- **Every state-changing POST accepts an `Idempotency-Key` header**, and so does every PATCH that is not naturally idempotent (incrementing a counter). GET, PUT and DELETE do not use one, because HTTP makes them idempotent.
- **The key is opaque and client-generated**, a UUID (v4 or v7) of at most 128 characters, scoped to one logical user action. Retries reuse the same key. The server never parses its structure.
- **The server stores one row per key**: `scope`, `idempotency_key`, `request_hash`, `status`, `locked_until`, `response_snapshot`, `expires_at`, and the audit columns that `database` requires.
- **The scope is the endpoint (`POST:/v1/customers`) and, on an authenticated endpoint, the caller's identity** (the token's `oid`), so endpoints and users never collide on a client key. A stored response is only ever replayed to the caller that stored it.
- **The claim commits before the work.** The server saves the key with status `in_progress` and a `locked_until` (default 60 seconds) in its own short transaction, so a concurrent request can see it. The use case then saves the state change and the response snapshot in its one unit of work. A claim past `locked_until` may be taken over, so a crash never blocks a key until it expires.
- **Replay behaviour is fixed.**

  | Situation | Response |
  |-----------|----------|
  | Same key, same body | The stored response: same status, same body, same `Location` |
  | Same key, different body | 409 with `code: IDEMPOTENCY_KEY_CONFLICT` |
  | Same key while the first request is still running | 409 with `code: IDEMPOTENCY_REQUEST_IN_FLIGHT` and `Retry-After` in integer seconds. The client retries with the same key after that delay |
  | New key | Processed normally, and the result is stored |

- **A reused key with a different body is surfaced, never ignored.** Silently ignoring a "duplicate" body is silent corruption.
- **Records expire after 24 hours by default**, configurable per app up to 7 days. A key without `expires_at` makes the table grow forever. After expiry the key is treated as new.
- **Records live in the app's own database**, never in instance memory, because instances recycle and scale out.
- **Keys are not forwarded downstream.** Each service boundary generates its own key. A queued job that ends in a public endpoint carries the original key in its payload, so the endpoint deduplicates as it would for a person.
- **No idempotency key on GET.**

## Rate limits

- **The response contract is the same wherever the limit is enforced.** A fired limit answers 429, never 503 (503 means the service itself is down, and load shedding also answers 429). `Retry-After` is always set, in integer seconds.
- **The body is the standard problem response** with `code: RATE_LIMITED` and a `retryAfter` extension that equals the header. Header and body never differ.
- **`RateLimit-Limit`, `RateLimit-Remaining` and `RateLimit-Reset` are optional but recommended**, so smarter clients can adapt.
- **What is limited**: public APIs, authenticated endpoints that do expensive work (search, exports, bulk operations), and any endpoint that verifies a credential such as an API key, which is limited strictest. Sign-in, sign-up and password reset belong to the identity provider (`authentication`), so the API has no endpoints for them.
- **What is not limited**: internal endpoints (a self-DOS), health checks, and service-to-service traffic between trusted apps.
- **Token bucket is the default algorithm.** Capacity is the burst allowance, the refill rate is the steady-state limit, and a request consumes one token. Fixed or sliding windows are allowed alternatives.
- **The limit is declared per operation** in the OpenAPI document with the `x-rate-limit` extension (`scope`, `limit`, `windowSeconds`) and a `429` response referencing the shared `RateLimited` response. The extension is informational and does not replace the actual configuration.

### Behind Azure API Management

Azure API Management is the intended front door of our APIs. Rate limiting is enforced there, outside the app, and the app does not implement its own limiter.

- **Limits are API Management policies**, configured per operation and per environment from the same values as the `x-rate-limit` declaration. They are not hardcoded.
- **The policy follows the scopes of this chapter**: per principal for authenticated traffic, per address for anonymous endpoints, and global per endpoint for hot endpoints.
- **The policy answers with the 429 contract above**, including the problem body.

> **TODO: not modeled yet.** We do not publish through API Management today. The rules for creating, updating and publishing an API in API Management (policy definitions, products, subscriptions, how the OpenAPI document is imported) still need to be written, and belong in this chapter once that is in place.

### Without API Management

Until an API is published through API Management, assume callers reach the app directly. The app then enforces the limit itself.

- **Limits apply per principal, and per address for anonymous endpoints**, enforced in the app.
- **The limiter uses shared state.** An in-memory limiter counts per instance, so the real limit becomes the limit times the instance count.
- **Addresses come from the platform's forwarded headers**, validated against the known front end, never from a client-supplied header. A limit has no bypass header.
- **Limits are configuration per app and environment**, never hardcoded.
- **Adopting API Management removes this app-level limiter**, so the limit is not counted twice.

## Internal endpoints

- **They live under `/internal/...`** and are excluded from the public OpenAPI document. A separate internal document (`/internal/openapi.json`, restricted to ops) is allowed but is not the default.
- **They are never versioned and never rate limited.**
- **They keep the same error shape and status rules** as public endpoints.
- **An operation a person runs is guarded by its own app role.** It requires a valid token plus an Entra ID app role for that one operation, assigned through a security group. The role grants nothing else, and no public endpoint accepts it.
- **The operator calls with their own token**, never a shared key or a service identity, so the action is traceable to the person.
- **Every operator operation writes an audit entry** with the operator, the operation and its target, by identifier.

## OpenAPI

- **OpenAPI is generated code-first from the controllers** with the built-in `Microsoft.AspNetCore.OpenApi`, at runtime. The controller is the source of truth, the document is the by-product, and there is no hand-written `.yaml` or `.json` document in the repository.
- **Metadata is complete.** `[ProducesResponseType]` is carried for every status a route can return, request DTOs carry DataAnnotations, and nullable reference types drive `required`. Missing metadata does not fail the build, it silently ships a worse contract.
- **The document is exported at build time** in the pipeline (`Microsoft.Extensions.ApiDescription.Server`), so consumers can generate clients without running the service.
- **There is one document per major version**, at `/v<n>/openapi.json`. It shares types where the contract is shared and splits them where it diverges. Each is registered with its own `AddOpenApi("v<n>")` and scoped to its path prefix with `ShouldInclude`.
- **Every document contains**: `info.title` (`<App> API`), `info.version` (the major, which bumps together with the URL), `info.description` (one paragraph and a link to the platform docs), `servers` per environment from bound configuration, a document-level `security` scheme (Bearer JWT) that endpoints leave only when truly public, and `components.schemas` for the DTOs, the problem shape, the pagination envelope and the error codes.
- **The shared schemas are defined once per service,** in its API project: `ProblemDetail`, `Page<T>`, the error envelope and the `traceId` and `code` extensions. Their shape is the one this standard defines, so the error and pagination shapes are identical in every service's document.
- **No anonymous inline schemas.** Every shape is a named DTO, so `$ref` resolves and generated clients get proper names.
- **Doc comments are added only when they carry information**: a non-obvious behaviour (idempotency semantics, an async 202 handle), or a DTO field whose purpose is not obvious from its name. `[EndpointSummary("Get customer")]` on the `Get` action of `CustomerController` is noise.
- **Non-trivial shapes get an example**, attached with schema transformers.
- **One OpenAPI specification version across all services.** It is 3.1, the version the built-in generator emits, and every service moves together when that changes.
- **An interactive API explorer (such as Scalar or Swagger UI) is not exposed in production** unless it is deliberately a fully public API, or it is authenticated.
- **Allowed sort and filter fields and the default sort are documented per endpoint.**
