# Architecture Reference

## 1. Architectural Goals

The architecture should optimize for:

- Clear separation of concerns
- Domain independence
- Testability
- Explicit dependencies
- Business rules isolated from infrastructure
- Replaceable infrastructure
- Maintainable feature development
- Transactional consistency
- Efficient read performance

Do not introduce abstractions merely to satisfy a pattern. Every abstraction must have a clear responsibility.

## 2. Layer Responsibilities

### Domain

The Domain is the business core.

It owns entities, aggregate roots, value objects, domain events, Result primitives (`Result`, `Result<TValue>`, `Error`, `ErrorType`), domain error definitions, domain exceptions (for catastrophic/unexpected failures only), domain services, repository interfaces, read repository interfaces, Unit of Work abstraction, and domain-facing contracts.

It must not reference ASP.NET Core, EF Core, Dapper, SQL Server client libraries, Infrastructure, or API.

### Application

Application coordinates use cases. It owns commands, queries, handlers, DTOs, validators, CQRS dispatcher (`IDispatcher`), mapping, and application orchestration.

Application depends on Domain and must not depend on Infrastructure.

### Infrastructure

Infrastructure contains implementation details such as EF Core, `DbContext`, entity configurations, migrations, Dapper, SQL, repository implementations, connection factories, external service implementations, messaging, files, email, and caching.

### API

API is the transport boundary. It contains controllers/endpoints, authentication/authorization setup, middleware, filters, DI composition, OpenAPI configuration, and HTTP response handling.

API must not contain business rules or database access.

## 3. Dependency Rule

```text
              ┌───────────────┐
              │      API      │
              └───────┬───────┘
                      │
                      ▼
              ┌───────────────┐
              │  Application  │
              └───────┬───────┘
                      │
                      ▼
              ┌───────────────┐
              │    Domain     │
              └───────────────┘
                      ▲
                      │
              ┌───────┴───────┐
              │ Infrastructure│
              └───────────────┘
```

Infrastructure implements contracts without leaking implementation details into handlers.

## 4. Domain-Driven Design

Use DDD deliberately rather than ceremonially.

### Entities

Entities have identity and lifecycle. Prefer controlled mutation when invariants matter.

```csharp
public sealed class Customer
{
    public Guid Id { get; private set; }
    public string Name { get; private set; } = string.Empty;
    public string Email { get; private set; } = string.Empty;

    private Customer() { }

    public static Result<Customer> Create(string name, string email)
    {
        if (string.IsNullOrWhiteSpace(name))
        {
            return Result.Failure<Customer>(CustomerErrors.EmptyName);
        }

        if (string.IsNullOrWhiteSpace(email))
        {
            return Result.Failure<Customer>(CustomerErrors.EmptyEmail);
        }

        return new Customer
        {
            Id = Guid.NewGuid(),
            Name = name,
            Email = email
        };
    }
}
```

### Value Objects

Use value objects for concepts whose identity is defined by their values, such as email addresses, money, addresses, or tax identifiers, when they contain meaningful behavior or invariants.

### Aggregate Roots

An aggregate root controls changes to its aggregate.

Prefer domain methods such as:

```csharp
order.AddItem(productId, quantity);
```

over exposing internal collections for arbitrary mutation.

## 5. Domain Events

Use domain events for meaningful business occurrences such as `CustomerRegistered`, `OrderSubmitted`, or `PaymentCompleted`.

Do not create events for every property change without a business reason.

## 6. Result Pattern & Functional Error Handling

Do not use exceptions for expected business errors or control flow (e.g., validation failure, entity not found, duplicate key). Exceptions are reserved for truly exceptional, unrecoverable technical failures (e.g., network failure, database unavailable).

Domain and Application layers model success and expected failure explicitly using `Result` and `Result<TValue>`.

### Result and Result<TValue>

```csharp
public class Result
{
    protected Result(bool isSuccess, Error error)
    {
        if (isSuccess && error != Error.None)
        {
            throw new InvalidOperationException("Success result cannot have an error.");
        }

        if (!isSuccess && error == Error.None)
        {
            throw new InvalidOperationException("Failure result must have an error.");
        }

        IsSuccess = isSuccess;
        Error = error;
    }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public Error Error { get; }

    public static Result Success() => new(true, Error.None);
    public static Result Failure(Error error) => new(false, error);
    public static Result<TValue> Success<TValue>(TValue value) => new(value, true, Error.None);
    public static Result<TValue> Failure<TValue>(Error error) => new(default, false, error);
    public static Result<TValue> Create<TValue>(TValue? value) =>
        value is not null ? Success(value) : Failure<TValue>(Error.NullValue);
}

public sealed class Result<TValue> : Result
{
    private readonly TValue? _value;

    internal Result(TValue? value, bool isSuccess, Error error)
        : base(isSuccess, error)
    {
        _value = value;
    }

    [NotNull]
    public TValue Value =>
        IsSuccess
            ? _value!
            : throw new InvalidOperationException("The value of a failure result cannot be accessed.");

    public static implicit operator Result<TValue>(TValue? value) => Create(value);
    public static implicit operator Result<TValue>(Error error) => Failure<TValue>(error);

    public TResult Match<TResult>(
        Func<TValue, TResult> onSuccess,
        Func<Error, TResult> onFailure) =>
        IsSuccess ? onSuccess(Value) : onFailure(Error);
}
```

### Error and ErrorType

```csharp
public record Error(string Code, string Description, ErrorType Type = ErrorType.Failure)
{
    public static readonly Error None = new(string.Empty, string.Empty, ErrorType.Failure);
    public static readonly Error NullValue = new("Error.NullValue", "The specified result value is null.", ErrorType.Failure);

    public static Error Failure(string code, string description) => new(code, description, ErrorType.Failure);
    public static Error NotFound(string code, string description) => new(code, description, ErrorType.NotFound);
    public static Error Validation(string code, string description) => new(code, description, ErrorType.Validation);
    public static Error Conflict(string code, string description) => new(code, description, ErrorType.Conflict);
    public static Error Unauthorized(string code, string description) => new(code, description, ErrorType.Unauthorized);
    public static Error Forbidden(string code, string description) => new(code, description, ErrorType.Forbidden);
}

public enum ErrorType
{
    Failure = 0,
    Validation = 1,
    NotFound = 2,
    Conflict = 3,
    Unauthorized = 4,
    Forbidden = 5
}
```

### Domain Errors

Domain errors are defined in static classes within Domain:

```csharp
public static class CustomerErrors
{
    public static readonly Error EmptyName =
        Error.Validation("Customer.EmptyName", "Customer name cannot be empty.");

    public static readonly Error EmptyEmail =
        Error.Validation("Customer.EmptyEmail", "Customer email cannot be empty.");

    public static readonly Error EmailNotUnique =
        Error.Conflict("Customer.EmailNotUnique", "The specified email is already in use.");

    public static Error NotFound(Guid id) =>
        Error.NotFound("Customer.NotFound", $"Customer with Id '{id}' was not found.");
}
```

## 7. CQRS & Messaging Abstractions

Commands change state. Queries retrieve state. All commands and queries return `Result` or `Result<TValue>`.

Application defines explicit marker and handler interfaces for commands, queries, and in-process dispatching:

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

Examples:

```text
CreateCustomerCommand : ICommand<Result<Guid>>
UpdateCustomerCommand : ICommand
DeleteCustomerCommand : ICommand

GetCustomerByIdQuery : IQuery<Result<CustomerDto>>
SearchCustomersQuery : IQuery<Result<IReadOnlyList<CustomerListItemDto>>>
GetCustomerSummaryQuery : IQuery<Result<CustomerSummaryDto>>
```

Each request should represent one clear use case.

## 8. Dispatching and Pipeline Flow

Handlers should be small and focused, executing their logic asynchronously via `HandleAsync`.

Preferred flow:

```text
Endpoint / Controller
  ↓
IDispatcher (SendAsync / QueryAsync)
  ↓
ICommandHandler / IQueryHandler (HandleAsync)
  ↓
Domain behavior (returns Result<T>)
  ↓
Repository abstraction
  ↓
Infrastructure
  ↓
Result<TValue> (Success or Failure)
  ↓
API Endpoint (TypedResults / ProblemDetails)
```

Infrastructure flow must remain invisible to handlers.

## 9. Unit of Work

Define a Domain-owned abstraction:

```csharp
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(
        CancellationToken cancellationToken);
}
```

Infrastructure implements it using EF Core.

Handlers use the abstraction rather than exposing `DbContext`.

## 10. Transaction Boundaries

A command that modifies state should have an explicit transaction boundary.

For EF Core persistence, the Unit of Work normally represents that boundary.

Do not manually open database transactions inside handlers.

If a use case spans multiple infrastructure systems, define an explicit orchestration strategy rather than hiding distributed transaction behavior inside repositories.

## 11. Feature Organization

Prefer vertical/feature-oriented Application organization:

```text
Features/
└── Orders/
    ├── Commands/
    │   ├── CreateOrder/
    │   └── CancelOrder/
    └── Queries/
        ├── GetOrderById/
        └── SearchOrders/
```

This keeps use-case code discoverable.

## 12. Dependency Injection Principle

Handlers should describe what they need:

```csharp
ICustomerRepository
IUnitOfWork
ICustomerReadRepository
```

They should not describe how the dependency works:

```csharp
CustomerDbContext
SqlConnection
SqlCommand
Dapper
CustomerRepository
```

This is a core rule of this skill.
