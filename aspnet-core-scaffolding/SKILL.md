---
name: aspnet-core-scaffolding
description: Scaffolds and evolves production-ready ASP.NET Core applications using Clean Architecture, Domain-Driven Design, CQRS, in-process IDispatcher, Result pattern, SQL Server, EF Core for writes, Dapper for reads, Domain-owned interfaces, Infrastructure implementations, FluentValidation, dependency injection, and Central Package Management. Use when creating a new ASP.NET Core solution, establishing its architecture, or adding features.
metadata:
  author: Mark
  version: "1.0"
  stack: "ASP.NET Core, .NET, DDD, Clean Architecture, CQRS, IDispatcher, Result Pattern, SQL Server, EF Core, Dapper"
---

# ASP.NET Core Scaffolding

## Purpose

Use this skill whenever scaffolding or extending an ASP.NET Core solution that must follow Clean Architecture, Domain-Driven Design, CQRS, in-process IDispatcher, SQL Server, and the project's repository/persistence conventions.

The generated solution must be production-oriented, testable, maintainable, explicit about dependencies, and free from unnecessary abstractions.

## Non-Negotiable Rules

1. Use Clean Architecture with clear dependency direction.
2. Apply DDD where it adds genuine domain value.
3. Use CQRS with explicit command and query abstractions (`ICommand<TResponse>`, `ICommand`, `ICommandHandler<in TCommand, TResponse>`, `IQuery<TResponse>`, `IQueryHandler<in TQuery, TResponse>`) dispatched via the in-process `IDispatcher`. Do not use MediatR.
4. Always use the Result pattern (`Result`, `Result<TValue>`, and `Error`) for domain operations, commands, queries, and endpoint responses. Never throw exceptions for expected business failures or control flow.
5. Use SQL Server/MSSQL as the relational database.
6. Use EF Core for writes and transactional persistence.
7. Use Dapper for reads and query-oriented projections.
8. Put persistence/service interfaces required by handlers in the **Domain** project, following the project's explicit architecture convention.
9. Put implementations in **Infrastructure**.
10. Never inject `DbContext` into command or query handlers.
11. Never inject `IDbConnection`, `SqlConnection`, Dapper, EF Core infrastructure, or concrete Infrastructure classes into handlers.
12. Handlers depend on abstractions only.
13. Prefer explicit feature-specific repositories over generic repositories.
14. Keep endpoints thin.
15. Propagate `CancellationToken` through application and infrastructure I/O.
16. Use dependency injection; do not manually instantiate Infrastructure services.
17. Use Central Package Management with `Directory.Packages.props`.
18. Keep business rules in Domain, orchestration in Application, implementation details in Infrastructure, and transport concerns in API.
19. Always use the in-process `IDispatcher` / `Dispatcher` for command and query routing.
20. Always use FluentValidation version 11.6.0
21. Always use Minimal APIs for new endpoints unless there is a documented reason to use controllers.

## Project Structure

```text
ProjectName/
├── src/
│   ├── ProjectName.Domain/
│   ├── ProjectName.Application/
│   ├── ProjectName.Infrastructure/
│   └── ProjectName.Api/
├── tests/
│   ├── ProjectName.Domain.Tests/
│   ├── ProjectName.Application.Tests/
│   └── ProjectName.IntegrationTests/
├── Directory.Build.props
├── Directory.Packages.props
├── .editorconfig
├── .gitignore
├── ProjectName.sln
└── README.md
```

See [project-structure.md](references/project-structure.md) for the detailed layout.

## Dependency Direction

```text
Api
 ├── Application
 └── Infrastructure

Application
 └── Domain

Infrastructure
 ├── Application
 └── Domain

Domain
 └── no project dependency on Application, Infrastructure, or API
```

Infrastructure implements abstractions owned by Domain/Application as defined by this skill. Application must not depend on Infrastructure.

## Domain

Domain owns:

- Entities and aggregate roots
- Value objects
- Domain services
- Domain events
- Result pattern primitives (`Result`, `Result<TValue>`, `Error`, `ErrorType`)
- Domain error definitions (e.g., `CustomerErrors`)
- Domain exceptions (for truly exceptional/unrecoverable technical failures)
- Enums
- Repository interfaces
- Read repository interfaces
- `IUnitOfWork`
- Other domain-facing contracts required by handlers

Domain must not contain:

- `DbContext`
- EF Core configurations
- Dapper
- SQL
- `SqlConnection`
- ASP.NET Core implementation concerns
- Infrastructure implementations

## Application

Application owns:

- CQRS abstractions (`ICommand`, `ICommandHandler`, `IQuery`, `IQueryHandler`, `IDispatcher`, `Dispatcher`)
- Commands
- Command handlers
- Queries
- Query handlers
- DTOs/read models
- FluentValidation validators
- Application orchestration
- Application mapping

Handlers must inject Domain-owned abstractions only.

### CQRS Abstractions & Dispatcher

Application defines the core messaging and dispatcher interfaces:

```csharp
/// <summary>
/// Marker interface representing a CQRS command that mutates state and returns a strongly typed response.
/// </summary>
/// <typeparam name="TResponse">The type of response returned upon handler execution.</typeparam>
public interface ICommand<TResponse>
{
}

/// <summary>
/// Marker interface representing a parameterless or non-generic CQRS command returning a standard <see cref="Result"/>.
/// </summary>
public interface ICommand : ICommand<Result>
{
}

/// <summary>
/// Defines a handler responsible for executing a specific command of type <typeparamref name="TCommand"/>.
/// </summary>
/// <typeparam name="TCommand">The command type being processed.</typeparam>
/// <typeparam name="TResponse">The expected response payload type.</typeparam>
public interface ICommandHandler<in TCommand, TResponse>
    where TCommand : ICommand<TResponse>
{
    /// <summary>
    /// Executes the business logic associated with the incoming command.
    /// </summary>
    /// <param name="command">The strongly typed command instance.</param>
    /// <param name="cancellationToken">Cancellation token to observe during asynchronous execution.</param>
    /// <returns>A task representing the asynchronous operation with the command outcome.</returns>
    Task<TResponse> HandleAsync(TCommand command, CancellationToken cancellationToken = default);
}

/// <summary>
/// Defines a handler responsible for executing a non-generic command of type <typeparamref name="TCommand"/> returning a standard <see cref="Result"/>.
/// </summary>
/// <typeparam name="TCommand">The command type being processed.</typeparam>
public interface ICommandHandler<in TCommand> : ICommandHandler<TCommand, Result>
    where TCommand : ICommand
{
}

/// <summary>
/// Marker interface representing a CQRS query that retrieves data without side effects.
/// </summary>
/// <typeparam name="TResponse">The query result payload type.</typeparam>
public interface IQuery<TResponse>
{
}

/// <summary>
/// Defines a handler responsible for executing a specific query of type <typeparamref name="TQuery"/>.
/// </summary>
/// <typeparam name="TQuery">The query type being evaluated.</typeparam>
/// <typeparam name="TResponse">The expected response type.</typeparam>
public interface IQueryHandler<in TQuery, TResponse>
    where TQuery : IQuery<TResponse>
{
    /// <summary>
    /// Executes the read-only query logic asynchronously.
    /// </summary>
    /// <param name="query">The strongly typed query instance.</param>
    /// <param name="cancellationToken">Cancellation token to observe.</param>
    /// <returns>A task representing the query execution returning the result.</returns>
    Task<TResponse> HandleAsync(TQuery query, CancellationToken cancellationToken = default);
}

/// <summary>
/// In-process mediator/dispatcher abstraction that routes commands and queries to their respective registered handlers.
/// </summary>
public interface IDispatcher
{
    /// <summary>
    /// Dispatches a command to its corresponding <see cref="ICommandHandler{TCommand, TResponse}"/>.
    /// </summary>
    /// <typeparam name="TResponse">The expected response type.</typeparam>
    /// <param name="command">The command instance to execute.</param>
    /// <param name="cancellationToken">Cancellation token to observe.</param>
    /// <returns>The response produced by the resolved command handler.</returns>
    Task<TResponse> SendAsync<TResponse>(ICommand<TResponse> command, CancellationToken cancellationToken = default);

    /// <summary>
    /// Dispatches a query to its corresponding <see cref="IQueryHandler{TQuery, TResponse}"/>.
    /// </summary>
    /// <typeparam name="TResponse">The expected response type.</typeparam>
    /// <param name="query">The query instance to execute.</param>
    /// <param name="cancellationToken">Cancellation token to observe.</param>
    /// <returns>The response produced by the resolved query handler.</returns>
    Task<TResponse> QueryAsync<TResponse>(IQuery<TResponse> query, CancellationToken cancellationToken = default);
}

/// <summary>
/// In-process CQRS mediator that dynamically resolves and invokes command and query handlers via Dependency Injection.
/// </summary>
public sealed class Dispatcher : IDispatcher
{
    private readonly IServiceProvider _serviceProvider;

    /// <summary>
    /// Initializes a new instance of the <see cref="Dispatcher"/> class.
    /// </summary>
    /// <param name="serviceProvider">The root or scoped service provider to resolve handlers from.</param>
    public Dispatcher(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider ?? throw new ArgumentNullException(nameof(serviceProvider));
    }

    /// <summary>
    /// Resolves the corresponding <see cref="ICommandHandler{TCommand, TResponse}"/> and executes it asynchronously.
    /// </summary>
    /// <typeparam name="TResponse">The expected response payload type.</typeparam>
    /// <param name="command">The command instance to dispatch.</param>
    /// <param name="cancellationToken">Cancellation token to observe.</param>
    /// <returns>The result returned by the resolved handler.</returns>
    /// <exception cref="InvalidOperationException">Thrown if no matching handler is registered in DI.</exception>
    public async Task<TResponse> SendAsync<TResponse>(ICommand<TResponse> command, CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(command);

        var handlerType = typeof(ICommandHandler<,>).MakeGenericType(command.GetType(), typeof(TResponse));
        var handler = _serviceProvider.GetRequiredService(handlerType);

        var method = handlerType.GetMethod(nameof(ICommandHandler<ICommand<TResponse>, TResponse>.HandleAsync))
            ?? throw new InvalidOperationException($"HandleAsync method not found on {handlerType.Name}");

        var task = (Task<TResponse>)method.Invoke(handler, [command, cancellationToken])!;
        return await task;
    }

    /// <summary>
    /// Resolves the corresponding <see cref="IQueryHandler{TQuery, TResponse}"/> and executes it asynchronously.
    /// </summary>
    /// <typeparam name="TResponse">The expected query response type.</typeparam>
    /// <param name="query">The query instance to dispatch.</param>
    /// <param name="cancellationToken">Cancellation token to observe.</param>
    /// <returns>The result returned by the resolved handler.</returns>
    /// <exception cref="InvalidOperationException">Thrown if no matching handler is registered in DI.</exception>
    public async Task<TResponse> QueryAsync<TResponse>(IQuery<TResponse> query, CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(query);

        var handlerType = typeof(IQueryHandler<,>).MakeGenericType(query.GetType(), typeof(TResponse));
        var handler = _serviceProvider.GetRequiredService(handlerType);

        var method = handlerType.GetMethod(nameof(IQueryHandler<IQuery<TResponse>, TResponse>.HandleAsync))
            ?? throw new InvalidOperationException($"HandleAsync method not found on {handlerType.Name}");

        var task = (Task<TResponse>)method.Invoke(handler, [query, cancellationToken])!;
        return await task;
    }
}
```

### Command pattern

```csharp
public sealed record CreateCustomerCommand(
    string Name,
    string Email) : ICommand<Result<Guid>>;

public sealed class CreateCustomerCommandHandler
    : ICommandHandler<CreateCustomerCommand, Result<Guid>>
{
    private readonly ICustomerRepository _customers;
    private readonly IUnitOfWork _unitOfWork;

    public CreateCustomerCommandHandler(
        ICustomerRepository customers,
        IUnitOfWork unitOfWork)
    {
        _customers = customers;
        _unitOfWork = unitOfWork;
    }

    public async Task<Result<Guid>> HandleAsync(
        CreateCustomerCommand command,
        CancellationToken cancellationToken = default)
    {
        var customerResult = Customer.Create(command.Name, command.Email);
        if (customerResult.IsFailure)
        {
            return Result.Failure<Guid>(customerResult.Error);
        }

        var customer = customerResult.Value;

        await _customers.AddAsync(customer, cancellationToken);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        return customer.Id;
    }
}
```

### Query pattern

```csharp
public sealed record GetCustomerByIdQuery(
    Guid Id) : IQuery<Result<CustomerDto>>;

public sealed class GetCustomerByIdQueryHandler
    : IQueryHandler<GetCustomerByIdQuery, Result<CustomerDto>>
{
    private readonly ICustomerReadRepository _customers;

    public GetCustomerByIdQueryHandler(
        ICustomerReadRepository customers)
    {
        _customers = customers;
    }

    public async Task<Result<CustomerDto>> HandleAsync(
        GetCustomerByIdQuery query,
        CancellationToken cancellationToken = default)
    {
        var customer = await _customers.GetByIdAsync(query.Id, cancellationToken);
        if (customer is null)
        {
            return Result.Failure<CustomerDto>(CustomerErrors.NotFound(query.Id));
        }

        return customer;
    }
}
```

## Persistence

### Writes

EF Core belongs exclusively to Infrastructure.

Use it for aggregate persistence, updates, deletes, change tracking, transactions, and migrations.

Handlers must never access `DbContext` directly.

### Reads

Dapper belongs exclusively to Infrastructure.

Use it for DTO projections, lists, search, reporting, joins, filtering, and performance-sensitive reads.

Handlers must never access Dapper or database connections directly.

See [persistence.md](references/persistence.md).

## Repositories

Prefer explicit contracts.

```csharp
public interface ICustomerRepository
{
    Task<Customer?> GetByIdAsync(
        Guid id,
        CancellationToken cancellationToken);

    Task AddAsync(
        Customer customer,
        CancellationToken cancellationToken);

    void Remove(Customer customer);
}
```

For reads:

```csharp
public interface ICustomerReadRepository
{
    Task<CustomerDto?> GetByIdAsync(
        Guid id,
        CancellationToken cancellationToken);

    Task<IReadOnlyList<CustomerListItemDto>> SearchAsync(
        CustomerSearchParameters parameters,
        CancellationToken cancellationToken);
}
```

Avoid creating `IRepository<T>` or `RepositoryBase<T>` unless there is a documented reason.

## CQRS and Dispatching

Each command/query represents one clear use case.

Commands change state. Queries retrieve state.

`IDispatcher` routes commands via `SendAsync` to `ICommandHandler<TCommand, TResponse>` and queries via `QueryAsync` to `IQueryHandler<TQuery, TResponse>`.

A handler should orchestrate the use case, not implement infrastructure.

Bad handler responsibilities:

- Opening SQL connections
- Building SQL
- Calling Dapper
- Calling EF Core directly
- Managing transactions manually
- Constructing infrastructure implementations
- Implementing complex domain rules

## Validation

Use FluentValidation for command/query input validation.

Validate request inputs in endpoints, handlers, or validation decorators.

Keep business invariants inside Domain. Do not rely exclusively on request validation to protect domain state.

## Dependency Injection

Application should expose its registration through an extension such as:

```csharp
public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddScoped<IDispatcher, Dispatcher>();

        // Register all command and query handlers in the assembly
        services.Scan(scan => scan
            .FromAssemblyOf<Dispatcher>()
            .AddClasses(classes => classes.AssignableTo(typeof(ICommandHandler<,>)))
                .AsImplementedInterfaces()
                .WithScopedLifetime()
            .AddClasses(classes => classes.AssignableTo(typeof(IQueryHandler<,>)))
                .AsImplementedInterfaces()
                .WithScopedLifetime());

        services.AddValidatorsFromAssembly(typeof(DependencyInjection).Assembly);

        return services;
    }
}
```

Infrastructure should expose its registration through:

```csharp
services.AddInfrastructure(configuration);
```

Infrastructure registration should include `DbContext`, repositories, Dapper connection factory, Unit of Work, and other infrastructure services.

## Central Package Management

Use `Directory.Packages.props` at the solution/repository root.

Package versions belong there.

Project files should normally omit package versions. Do not scatter package versions across `.csproj` files.

## Feature Organization

Prefer feature-oriented Application organization:

```text
Application/
└── Features/
    └── Customers/
        ├── Commands/
        │   └── CreateCustomer/
        │       ├── CreateCustomerCommand.cs
        │       ├── CreateCustomerCommandHandler.cs
        │       └── CreateCustomerCommandValidator.cs
        └── Queries/
            └── GetCustomerById/
                ├── GetCustomerByIdQuery.cs
                └── GetCustomerByIdQueryHandler.cs
```

Keep closely related request, handler, validator, and DTO code together.

## API

Controllers/minimal API endpoints should:

1. Receive HTTP input.
2. Map transport input to a command/query.
3. Send through `IDispatcher` (`dispatcher.SendAsync` or `dispatcher.QueryAsync`).
4. Return an HTTP result using `TypedResults` or `Result` matching.

Example Minimal API endpoints:

```csharp
app.MapPost("/api/customers", async (
    [FromBody] CreateCustomerRequest request,
    [FromServices] IDispatcher dispatcher,
    CancellationToken cancellationToken) =>
{
    var command = new CreateCustomerCommand(request.Name, request.Email);
    var result = await dispatcher.SendAsync(command, cancellationToken);

    if (result.IsFailure)
    {
        var error = result.Error;
        if (error.Type == ErrorType.Validation)
        {
            return Results.ValidationProblem(new Dictionary<string, string[]> { [error.Code] = [error.Description] });
        }
        return Results.Problem(title: error.Code, detail: error.Description, statusCode: CustomResults.GetStatusCode(error.Type));
    }

    return Results.CreatedAtRoute("GetCustomerById", new { id = result.Value }, new { id = result.Value });
});

public static async Task<Results<Ok<CustomerDto>, ProblemHttpResult>> GetCustomerByIdAsync(
    [FromRoute] Guid id,
    [FromServices] IDispatcher dispatcher,
    CancellationToken cancellationToken)
{
    var query = new GetCustomerByIdQuery(id);
    var result = await dispatcher.QueryAsync(query, cancellationToken);

    if (result.IsFailure)
    {
        var error = result.Error;
        return TypedResults.Problem(
            title: error.Code,
            detail: error.Description,
            statusCode: CustomResults.GetStatusCode(error.Type));
    }

    return TypedResults.Ok(result.Value);
}
```

Do not put SQL, Dapper, EF Core, repository implementations, or business rules in API endpoints.

## Scaffolding Workflow

When creating a new solution:

1. Determine solution and project names.
2. Determine the target .NET version; use the requested version when specified.
3. Create Domain, Application, Infrastructure, and API projects.
4. Add test projects appropriate to the solution.
5. Add project references according to the dependency rules.
6. Create `Directory.Build.props`.
7. Create `Directory.Packages.props`.
8. Establish Domain primitives (`Entity`, `AggregateRoot`, `ValueObject`, `Result`, `Error`) and interfaces.
9. Establish Application CQRS command, query, handler, and `IDispatcher` structure.
10. Establish Infrastructure persistence and dependency injection.
11. Establish API composition, middleware, endpoint mapping, and OpenAPI configuration.
12. Add validation and error mapping.
13. Add representative tests.
14. Build the solution.
15. Run tests.
16. Fix compilation, dependency, and architecture violations before completion.
17. Create .editorconfig and .gitignore files.
18. Commit the initial scaffold to source control.

## Adding a Feature

For each new feature:

1. Identify the aggregate/domain behavior involved.
2. Add or update Domain abstractions and domain error definitions where required.
3. Add the command/query and handler in Application returning `Result` / `Result<TValue>`.
4. Add validation where required.
5. Add repository implementations in Infrastructure.
6. Add API endpoint mapping dispatching via `IDispatcher`.
7. Add unit/integration tests.
8. Build and test.

## Definition of Done

The scaffold is complete only when:

- The solution builds successfully.
- Tests pass.
- Project references follow Clean Architecture.
- Domain has no Infrastructure dependency.
- Application has no Infrastructure dependency.
- CQRS commands and queries implement explicit `ICommand` / `IQuery` interfaces with `ICommandHandler` / `IQueryHandler` executing via `HandleAsync` and dispatched via `IDispatcher`.
- No MediatR package dependency exists.
- All domain operations, commands, and queries use the Result pattern (`Result` / `Result<TValue>`) instead of throwing exceptions for business errors.
- Handlers inject interfaces only.
- No handler injects `DbContext`.
- No handler uses Dapper or SQL.
- EF Core is isolated to Infrastructure.
- Dapper is isolated to Infrastructure.
- SQL Server configuration is isolated to Infrastructure.
- Validation is wired into the request pipeline.
- Cancellation tokens are propagated.
- Central Package Management is configured.
- Endpoints remain thin and map `Result` failures to appropriate HTTP ProblemDetails responses.
- Important domain/application behavior has tests.
- No unnecessary generic repository/service abstraction exists.

## Reference Documents

- [Architecture](references/architecture.md)
- [Persistence](references/persistence.md)
- [Project Structure](references/project-structure.md)
- [Coding Standards](references/coding-standards.md)
