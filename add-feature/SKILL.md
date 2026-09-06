---
name: add-feature
description: Interactively adds vertical slice features (Commands, Queries, Domain Entities, EF Core configurations, Dapper read repositories, FluentValidation, Minimal APIs) to an existing ASP.NET Core Clean Architecture solution. Asks targeted clarifying questions for missing details such as database table names, column schemas, business invariants, and feature specifics while strictly enforcing CQRS, IDispatcher, Result pattern, and persistence conventions from aspnet-core-scaffolding.
metadata:
  author: Adesina Mark Omoniyi
  version: "1.0"
  stack: "ASP.NET Core, .NET, Clean Architecture, CQRS, IDispatcher, Result Pattern, EF Core, Dapper, SQL Server, FluentValidation"
  tags:
    - dotnet
    - aspnetcore
    - clean-architecture
    - cqrs
    - feature-scaffolding
    - dapper
    - efcore
    - result-pattern
    - dispatcher
    - fluentvalidation
---

# Add Feature to ASP.NET Core Solution

## Purpose

Use this skill whenever adding a new feature, use case, command, query, or domain aggregate into an existing ASP.NET Core solution adhering to **Clean Architecture**, **Domain-Driven Design (DDD)**, **CQRS with `IDispatcher`**, **Result Pattern**, and **Dual Persistence (EF Core for writes, Dapper for reads)**.

This skill ensures that requirements (such as database table names, column specifications, business invariants, and API routes) are **interactively clarified and confirmed** before code is generated.

---

## 🛑 Non-Negotiable Architectural Rules

Every feature added must comply with the foundational rules established in `aspnet-core-scaffolding`:

1. **Dependency Direction**:
   - `Domain` has zero dependencies on other projects.
   - `Application` depends only on `Domain`.
   - `Infrastructure` implements abstractions defined in `Domain` and `Application`.
   - `Api` references `Application` and `Infrastructure` for composition and routing.
2. **CQRS with In-Process `IDispatcher`**:
   - Every state-mutating operation is an `ICommand<Result<TResponse>>` or `ICommand`.
   - Every read operation is an `IQuery<Result<TResponse>>`.
   - Handlers implement `ICommandHandler<TCommand, TResponse>` or `IQueryHandler<TQuery, TResponse>` via `HandleAsync`.
   - All dispatches go through `IDispatcher.SendAsync` or `IDispatcher.QueryAsync`.
   - **Do NOT use MediatR**.
3. **Result Pattern**:
   - Return strongly typed `Result` / `Result<TValue>` with explicit domain `Error` definitions.
   - Never throw exceptions for expected business rule violations, validation failures, or missing entities.
4. **Persistence Separation**:
   - **Writes**: EF Core (`DbContext`, `IEntityTypeConfiguration<T>`) isolated strictly within `Infrastructure`.
   - **Reads**: Dapper (`IDbConnectionFactory`, parameterized SQL queries) isolated strictly within `Infrastructure`.
   - **Contracts**: Repository interfaces (`I{Feature}Repository`, `I{Feature}ReadRepository`) are owned by `Domain`.
   - **Zero Leaks**: Handlers must **never** inject `DbContext`, `IDbConnection`, `SqlConnection`, or concrete Infrastructure classes.
5. **Validation**:
   - Use **FluentValidation (version 11.6.0)** for request/command validation (`AbstractValidator<T>`).
   - Domain invariants are encapsulated directly in Domain entities / factory methods (`Entity.Create(...)`).
6. **API Transport**:
   - Prefer **Minimal APIs** using `TypedResults` or `Results.*`.
   - Map `Result.Failure(Error)` to appropriate HTTP ProblemDetails (`400`, `404`, `409`, `422`).
   - Propagate `CancellationToken` through all asynchronous calls.

---

## 🔄 Feature Workflow & Lifecycle

```text
┌────────────────────────────────────────────────────────┐
│ Phase 1: Interactive Requirement Discovery & Prompts   │
├────────────────────────────────────────────────────────┤
│ Phase 2: Domain Modeling & Contracts                   │
├────────────────────────────────────────────────────────┤
│ Phase 3: Application CQRS Command / Query & Validation │
├────────────────────────────────────────────────────────┤
│ Phase 4: Infrastructure Persistence (EF + Dapper)      │
├────────────────────────────────────────────────────────┤
│ Phase 5: API Minimal Endpoint & DI Registration        │
├────────────────────────────────────────────────────────┤
│ Phase 6: Unit / Integration Tests                      │
├────────────────────────────────────────────────────────┤
│ Phase 7: Build & Verification Checklist                │
└────────────────────────────────────────────────────────┘
```

---

## ❓ Phase 1: Interactive Requirement Discovery

Before writing code, inspect the user's request and workspace context. If any of the following details are missing or ambiguous, **prompt the user with clarifying questions**:

### Clarification Checklist:

1. **Database Table & Schema**:
   - What is the exact database table name? (e.g., `dbo.Orders`, `billing.Subscriptions`)
   - What are the primary keys, column names, data types, nullability, and max length constraints?
   - Are there foreign keys or relationships to existing tables?
2. **Feature Details & Use Case**:
   - What is the use case name? (e.g., `CreateCustomer`, `ApproveLoanApplication`, `GetOrderSummaryById`)
   - Is it a **Command** (state-mutation), a **Query** (read-only projection), or a **Full Aggregate Slice** (Entity + Command + Query)?
3. **Business Rules & Invariants**:
   - What domain invariants must be validated before persisting?
   - What domain error codes should be returned on failure (e.g., `OrderErrors.AlreadyShipped`)?
   - What input validation rules apply (lengths, required fields, formats)?
4. **Transport & API Route**:
   - What is the Minimal API route and HTTP method? (e.g., `POST /api/orders`, `GET /api/orders/{id}`)
   - What are the expected HTTP response status codes?

> See [references/interactive-prompts.md](references/interactive-prompts.md) for pre-formatted questionnaires.

---

## 🚀 Phase 2 to 6: Layer-by-Layer Implementation Guide

### 1. Domain Layer (`src/ProjectName.Domain/`)

- **Entity / Aggregate Root** (`Entities/{EntityName}.cs`):
  - Private constructor for EF Core.
  - Factory method returning `Result<TEntity>` with invariant protection.
- **Domain Errors** (`Errors/{EntityName}Errors.cs`):
  - Static class defining `Error.NotFound`, `Error.Validation`, `Error.Conflict`.
- **Repository Contracts** (`Repositories/`):
  - Write contract: `I{EntityName}Repository.cs`
  - Read contract: `I{EntityName}ReadRepository.cs`

### 2. Application Layer (`src/ProjectName.Application/Features/{FeatureGroup}/`)

- **Commands** (`Commands/{CommandName}/`):
  - `Create{Entity}Command.cs` implementing `ICommand<Result<Guid>>` or `ICommand<Result<TResponse>>`.
  - `Create{Entity}CommandValidator.cs` inheriting `AbstractValidator<Create{Entity}Command>`.
  - `Create{Entity}CommandHandler.cs` implementing `ICommandHandler<Create{Entity}Command, Result<Guid>>`.
- **Queries** (`Queries/{QueryName}/`):
  - `Get{Entity}ByIdQuery.cs` implementing `IQuery<Result<{Entity}Dto>>`.
  - `{Entity}Dto.cs` read model record.
  - `Get{Entity}ByIdQueryHandler.cs` implementing `IQueryHandler<Get{Entity}ByIdQuery, Result<{Entity}Dto>>`.

### 3. Infrastructure Layer (`src/ProjectName.Infrastructure/`)

- **EF Core Configuration** (`Persistence/Configurations/{EntityName}Configuration.cs`):
  - Implements `IEntityTypeConfiguration<TEntity>`.
  - Maps to the user's specified table: `builder.ToTable("TableName");`.
  - Configures keys, column types, max lengths, indexes.
- **EF Core Write Repository** (`Persistence/Repositories/{EntityName}Repository.cs`):
  - Implements `I{EntityName}Repository`.
  - Injects `ProjectNameDbContext`.
- **Dapper Read Repository** (`Persistence/Repositories/{EntityName}ReadRepository.cs`):
  - Implements `I{EntityName}ReadRepository`.
  - Injects `IDbConnectionFactory`.
  - Executes parameterized SQL queries (e.g. `SELECT Column1, Column2 FROM TableName WHERE Id = @Id;`).
- **DbContext & DI Registration**:
  - Add `DbSet<TEntity>` to `ProjectNameDbContext.cs`.
  - Ensure registrations are wired in `DependencyInjection.cs` (`services.AddScoped<I...Repository, ...Repository>()`).

### 4. API Layer (`src/ProjectName.Api/`)

- **Endpoints** (`Endpoints/{EntityName}Endpoints.cs`):
  - Extension method `Map{EntityName}Endpoints(this IEndpointRouteBuilder app)`.
  - Uses `app.MapGroup("/api/{resource}")`.
  - Dispatches commands with `await dispatcher.SendAsync(command, cancellationToken)`.
  - Dispatches queries with `await dispatcher.QueryAsync(query, cancellationToken)`.
  - Converts `Result` failures into RFC 7807 `ProblemDetails`.

### 5. Testing Layer (`tests/`)

- **Domain Tests** (`tests/ProjectName.Domain.Tests/`): Unit test entity creation and invariant enforcement.
- **Application Tests** (`tests/ProjectName.Application.Tests/`): Unit test handler execution, mocked repositories, validator execution.

> See [references/feature-scaffolding-guide.md](references/feature-scaffolding-guide.md) for full code templates.

---

## ✅ Phase 7: Verification & Definition of Done

A feature is complete only when:

- [ ] All required details (table name, schema, business logic, endpoints) were collected and confirmed.
- [ ] Code strictly follows Clean Architecture dependency boundaries.
- [ ] No handler injects `DbContext`, `IDbConnection`, or concrete infrastructure.
- [ ] All read queries use Dapper with parameterized SQL queries.
- [ ] All write commands use EF Core and `IUnitOfWork.SaveChangesAsync`.
- [ ] Dispatching is handled exclusively via `IDispatcher` (no MediatR).
- [ ] Result pattern is used for all operations; no business exceptions thrown.
- [ ] FluentValidation (v11.6.0) validates request inputs.
- [ ] `CancellationToken` is passed to all asynchronous calls.
- [ ] The solution builds successfully (`dotnet build`).
- [ ] Unit and integration tests pass (`dotnet test`).

---

## 📚 References & Cross-Links

- [Interactive Discovery Prompts](references/interactive-prompts.md)
- [Feature Scaffolding Guide & Code Templates](references/feature-scaffolding-guide.md)
- [ASP.NET Core Scaffolding Core Standards](../aspnet-core-scaffolding/SKILL.md)
- [Architecture Guidelines](../aspnet-core-scaffolding/references/architecture.md)
- [Persistence & Dapper Guidelines](../aspnet-core-scaffolding/references/persistence.md)
- [Coding Standards](../aspnet-core-scaffolding/references/coding-standards.md)
