---
id: coding-standards-csharp
scope: scoped
severity: warning
applies: all
applies-to:
  - "**/*.cs"
schemaVersion: 1
---

# Coding standards: C#

The C#/.NET specifics on top of the coding standards for all languages, which are not repeated here.

## Basic Stack
- .NET latest LTS
- ASP.NET Core
- EF Core (SQL Server)
- xUnit v3
- NSubstitute

## File and project organization

- **C# File Type** A `.cs` file is a contract, data shapes, or logic. Never a mix.
- **Interfaces get their own file**, named after the interface (`IOrderRepository.cs`). This includes hexagonal ports. No grouped `Ports.cs`.
- **Small records (up to 5 non-white lines) may share a file** and sit at the top, above the first class. Larger records get their own file or share one with other records only.
- **DTOs and logic never share a file.** The only exception is a private one-line record that is just its own class's return shape.
- **Solution Root Folders.** In the Solution root we separate code in `src`, tests in `test` to organize projects. External tools or other namespaces not directly correlated to the main application and it's testing, can be allocated it's own root folder, though this is rare.
- **One folder per aggregate** inside each project. Namespaces follow folders.
- **File-scoped namespaces** (`namespace Pareto.Hello.Domain;`), never namespace blocks.

## Async

- **Async end to end.** Controllers return `Task<T>`, services await the async APIs of EF Core and `HttpClient`.
- **`.Result`, `.Wait()` and `.GetAwaiter().GetResult()` are banned** Synchronous code that needs an async API becomes async itself.
- **`CancellationToken` is the last parameter** Of every async service and repository method, and it is passed down to every awaited call.
- **No `async void`, no fire-and-forget** (`_ = SomethingAsync()`, un-awaited `Task.Run`).
- **`Task.WhenAll` is for independent external clients only.** Never run two queries on one `DbContext` concurrently. It is not thread-safe.

## Dependency injection and configuration

- **Constructor injection only.** No service locator (`GetRequiredService<T>()` in business code), no `IServiceProvider` lookups.
- **Lifetimes.** Services and repositories are Scoped. Singleton is for stateless helpers only and never captures a scoped dependency. Transient is rare.
- **Each unit of work has its own DI scope.** A request gets one scope. Work outside a request (hosted services, queue and timer handlers, scheduled jobs) creates a scope per message or job with `IServiceScopeFactory`. A scoped service is never used by two operations at once, so parallel work resolves its own (for a `DbContext`, from `IDbContextFactory<T>`).
- **No ambient unit-of-work framework required** Scope and commit are explicit.
- **Configuration is read once, at the boundary, into small typed records.** IOptions<T>, bound from a named section, ValidateDataAnnotations() and ValidateOnStart(), so bad config fails at startup. Only the record travels downstream. No class takes IConfiguration and looks up a key by name, whether the setting is grouped or single. Platform-defined names (for example the Application Insights connection string) are bound into the record at the boundary and nowhere else.
- **No environment-name branching in services.** A difference between environments is a config flag. Environment checks are for registration existence only.

## Logging

- **`ILogger<T>`, constructor-injected, one per class.** `Microsoft.Extensions.Logging` is the only logging API.
- **Message templates, never interpolation.** `log.LogInformation("customer created {CustomerId}", id)`, not `$"..."`. Interpolation builds the string at disabled levels and destroys the structured field.
- **No `Console.WriteLine`.**
- **Errors are logged once**, at the exception middleware boundary. Not in services, not in repositories.

## Mapping

- **Mappers are plain static classes of extension methods**, one direction per method (`hub.ToDomain()`, `order.ToResponseDto()`), living next to the adapter they serve.
- **AutoMapper, Mapster and any reflection-based mapper are banned.**
- **Mappers are pure.** No database access, no exception handling. They are DI-registered only when they need an injected dependency (a clock, an ID generator).
- **Adapter models never reference each other.** They only know the domain type. All cross-adapter translation goes through the domain.

## Time, money and numbers

- **Time is injected** as `TimeProvider`. No `DateTime.Now`, `DateTime.UtcNow` or `DateTimeOffset.UtcNow` in domain or application code. This is also how tests control time (`FakeTimeProvider`).
- **Instants are `DateTimeOffset`, in UTC.** Time zones are a presentation concern.
- **Money is `decimal`, never `float` or `double`.** A money amount travels with its currency.
- **Culture.** Machine-readable parsing and formatting uses `CultureInfo.InvariantCulture`, and string comparison states its `StringComparison` (usually `Ordinal`). 

## Testing in C#

- **Mock only the process boundary** (database, HTTP clients, queues). Value objects, mappers and simple collaborators stay real.
- **Coverage checks uses coverlet** Exclusions use `[ExcludeFromCodeCoverage]`.

## Code Standards for C#

- **Nullable reference types are on in every project.** `null` is a deliberate, typed value. A type that can be absent says so (`Customer?`); everything else is never null. Request DTOs declare required members as non-nullable, since the OpenAPI schema derives `required` from it.
- **Warnings are errors.** `TreatWarningsAsErrors` and `EnforceCodeStyleInBuild` are set in `Directory.Build.props`.
- **Records for data.** DTOs, value objects and immutable domain shapes are records. `required` members plus named arguments turn a forgotten field into a compile error.
- **The Domain project has zero framework package references.** No EF Core, no JSON attributes, no ASP.NET Core.
- **Name encoding exceptions.** The `I` prefix for interfaces, and the `Async` suffix only when a non-async overload with the same name exists (`GetUser`, not `GetUserAsync`).
- **XML `<summary>`** (controllers, request/response DTOs) because Swagger/OpenAPI is generated from it. It is API documentation, not comment smell.
- **null-forgiving operator `!` is banned in code.** Two exceptions: the `= null!` initializer on an EF Core entity property, and test code.
- **Not Found Exceptions** Repositories can return nullable objects for not found. This will cause a service level exception. A service does not return null.
- **Required over Nullable** A data object should enforce with required instead of using nullable
- **Sealed by default** Unless inheritance is desired/planned, a class is sealed.
- **Domain IDs** Domain object IDs are unique ValueObject types to prevent mixing identifiers from different types.
- **Switch Statement** A switch statement always has a defined default arm that often fails loudly. A new option/member/value should be explicitly flagged by this default.
- **Keep Banned APIs** using `Microsoft.CodeAnalysis.BannedApiAnalyzers` apply a strict ban on use of items not allowed in these C# rules like `DateTime.Now` outside of a dedicated time providing class. Alternative logging to ILogger, etc. They then generate an error to easily guard against abuse.
- **Project Naming** `<Company>.<App>.<Layer>` for production projects (`Pareto.Hello.Domain`) and `.Tests` / `.IntegrationTests` for test projects.
- **Types are Explicit.** Types should never use var, it hampers readability. Prefer: `MyClass c = new();` as notation.

## Databases

Databases in C# are always managed through EF Core. Here are the guidelines for that.

### Entities and mapping - All Providers

- **One `DbContext` per app and database.** It is registered Scoped (`AddDbContext`), and never captured by a singleton.
- **Persistence entities live only in the Infrastructure project** (`Persistence/<Aggregate>/`). They are mutable, may use `= null!` initializers, and are never returned from a controller or used as the domain model. A mapper translates at the repository edge.
- **Entities are configured with `IEntityTypeConfiguration<T>` classes**, applied through `ApplyConfigurationsFromAssembly`, instead of data-annotation attributes on the entity. It keeps persistence detail in one place and out of the entity classes.
- **Persisted enums are stored as integers with an explicit number per member** (`Open = 1`). The number is the stored contract. Declaration order and member names may change, the numbers never do.
- **The name a member carries in API payloads and events is set explicitly** (`[JsonStringEnumMemberName("partially_refunded")]`), never derived from the member name, so renaming a member does not change the contract.
- **A persisted enum member is never removed, renumbered or reused.** A retired member stays and is marked `[Obsolete("reason")]`. The warning that generates needs to be wrapped at the definition (`#pragma warning disable CS0618` and `restore`), so `TreatWarningsAsErrors` does not fail the build on the Enum value's continued existence.
- **Enums are guarded at the model.** EF Core materializes any integer without checking it. A value that is not a defined member (`Enum.IsDefined`) is rejected where data enters the model: when the mapper builds the domain object from a stored value, and when an aggregate accepts a new value.
- **Identifiers are generated by the application** with `ValueGeneratedNever()`. The provider never generates them.
- **Strongly typed domain IDs get their value converter once**, in `ConfigureConventions`, so every `OrderId` maps to the provider's identifier type without per-property setup.
- **Audit values are filled by a `SaveChangesInterceptor`**, registered once, over a shared base entity. The interceptor takes its time from the injected `TimeProvider`.
- **The concurrency token is mapped per provider**: `rowversion` with `IsRowVersion()` on SQL Server, ETag on Cosmos DB.
- **Soft delete is a global query filter** (`HasQueryFilter`), and deleted rows are read only in intent-named repository methods (restore, audit), never inline in a general query. If a normal flow needs the row, model an explicit state instead of bypassing the filter.
- **Every global query filter is a named filter**, with the name held in one constant. A bypass disables only the filter it needs (`IgnoreQueryFilters(["SoftDelete"])`), never all of them, because tenancy and authorization filters must stay on. Parameterless `IgnoreQueryFilters()` is banned.
- **Each bypass method has a test** proving that the remaining filters still apply.
- **Never enable lazy-loading proxies.** Related data is loaded explicitly or projected.
- **Raw SQL (`FromSql`/`SqlQuery`) is allowed only when LINQ cannot express the query.** It uses the interpolated forms, never string concatenation, and the SQL inside follows coding-standards-sql.md.
- **`EnableSensitiveDataLogging` and `EnableDetailedErrors` are off everywhere except the Local environment.** Both write parameter values (PII) into logs.
- **The EF Core major version matches the .NET major version.**

### Repositories and transactions - All Providers

- **One repository per aggregate root.** Methods are intent-named and return `Task`. Never expose `IQueryable` or a `DbSet`.
- **Repositories stage, the entry-point use case commits.** One `SaveChangesAsync` per use case. A service called by another service stages and does not commit. No request filter, middleware, pipeline behavior or `Dispose` commits on the service's behalf. The aggregate is the unit of consistency, and what one save guarantees is provider-specific.
- **Read paths use `AsNoTracking()` or a projection** to a read model (`ReadModel`). Write paths load tracked, mutate and save.
- **Outbox rows are added in the same `SaveChangesAsync` as the state change.**
- **Repository tests run against the real provider** (Testcontainers or the provider's emulator), never the EF Core in-memory provider or SQLite stand-ins.

### Entities and mapping - Relational Providers

- **snake_case comes from one convention**, `UseSnakeCaseNamingConvention()` (EFCore.NamingConventions), set once on the `DbContext`. No per-entity `[Table]`/`[Column]` overrides without a stated reason. The app schema is set once with `HasDefaultSchema`.
- **Schemas should be logical** Where the default schema is for regular persistence, auditing or staging tables often warrant a clean separate schema to live in.
- **Nullable reference types decide the schema.** EF Core maps `string` to NOT NULL and `string?` to NULL.

### Repositories and transactions - Relational Providers

- **One `SaveChangesAsync` is one transaction** for everything the change tracker holds. A use case that also uses a direct write (`ExecuteUpdate`/`ExecuteDelete`, raw SQL) opens one explicit transaction around all of it, in the entry-point use case and nowhere else. Never in a repository or controller, and never `TransactionScope`. The idempotency claim (api.md) is the one exception: it commits in its own short transaction before the use case runs.
- **With retry-on-failure enabled, an explicit transaction runs inside the context's execution strategy** (`CreateExecutionStrategy().ExecuteAsync`), because the whole unit is retried, not a single statement.
- **Paging uses `OrderBy` on a stable key** before `Skip`/`Take`.
- **Queries that `Include` more than one collection use `AsSplitQuery()`** (or it is set as the default), to avoid the cartesian explosion of a single joined query.
- **`ExecuteUpdateAsync`, `ExecuteDeleteAsync` and Raw SQL bypass the interceptor and the change tracker.** After one, set audit values explicitly.

### Migrations - Relational Providers

- **Created with `dotnet ef migrations add <PascalCaseName>`**, verb first (`AddCustomersArchivedAt`). They live in the Infrastructure project's `Migrations/` folder.
- **Never edit a deployed migration, `__EFMigrationsHistory`, or migration timestamps.**
- **`Down()` exists because the tool generates it**, but it is never run in acceptance or production.
- **Review the SQL** from `dotnet ef migrations script` in every migration PR.
- **The model snapshot is generated.** A merge conflict in it is resolved by regenerating the later migration on top of the earlier one, never by editing the snapshot.
- **Migrations never reference application types** (enums, domain classes, helpers). They are a frozen snapshot of the schema at that time, and a later rename of an app type would break an old migration.
- **Reject scaffolded `RenameColumn`** (it becomes `sp_rename`) and any column type change in a single step.
- **Views, functions and procedures** are defined with `CREATE OR ALTER` in `migrationBuilder.Sql(...)`. They always get their own migration step for clarity.
- **Reference data uses `InsertData` or `Sql`** in a migration.
- **The pipeline builds a migration bundle** (`dotnet ef migrations bundle`) from the same commit as the app image. `Database.Migrate()` at startup and `dotnet ef database drop` in a pipeline are banned.
- **An integration test applies all migrations to a fresh container** and asserts `Database.HasPendingModelChanges()` is false, to catch a model change committed without its migration. No other migrations tests are required or desired.
- **The `Microsoft.EntityFrameworkCore.Design` package is referenced only in the project that hosts migrations**, with `PrivateAssets="all"`, so design-time tooling does not ship with the app.

### Entities and mapping - Document Providers

- **Child entities embedded in the aggregate are owned types.** The partition key is configured in the entity configuration (`HasPartitionKey`).
- **No `EnsureCreated` in production.** Containers and indexing policy come from infrastructure-as-code.

### Repositories and transactions - Document Providers

- **Queries pass the partition key** (`WithPartitionKey`) so they stay single-partition, and page with continuation tokens instead of `Skip`.
