# SharedKernel Reference

The `SharedKernel` is a centralized, reusable core class library referenced by every microservice in the distributed system. It encapsulates cross-cutting domain primitives, CQRS abstractions, the zero-overhead in-process `IDispatcher`, the Result pattern, validation behaviors, and shared integration contracts.

---

## 1. Project Directory Layout

```text
SolutionName.SharedKernel/
├── CQRS/
│   ├── ICommand.cs
│   ├── ICommandHandler.cs
│   ├── IQuery.cs
│   ├── IQueryHandler.cs
│   ├── IDispatcher.cs
│   └── Dispatcher.cs
├── Results/
│   ├── Error.cs
│   ├── ErrorType.cs
│   └── Result.cs
├── Domain/
│   ├── Entity.cs
│   ├── AggregateRoot.cs
│   ├── IDomainEvent.cs
│   ├── IIntegrationEvent.cs
│   └── IUnitOfWork.cs
├── Behaviors/
│   └── ValidationBehavior.cs
├── Extensions/
│   └── SharedKernelServiceExtensions.cs
└── SolutionName.SharedKernel.csproj
```

---

## 2. CQRS Abstractions & In-Process `IDispatcher`

### `ICommand.cs` & `ICommandHandler.cs`

```csharp
namespace SolutionName.SharedKernel.CQRS;

using SolutionName.SharedKernel.Results;

/// <summary>
/// Marker interface representing a CQRS command returning a typed Result.
/// </summary>
public interface ICommand<TResponse>
{
}

/// <summary>
/// Marker interface representing a CQRS command returning a standard Result.
/// </summary>
public interface ICommand : ICommand<Result>
{
}

/// <summary>
/// Handler responsible for executing a specific command.
/// </summary>
public interface ICommandHandler<in TCommand, TResponse>
    where TCommand : ICommand<TResponse>
{
    Task<TResponse> HandleAsync(TCommand command, CancellationToken cancellationToken = default);
}

/// <summary>
/// Handler responsible for executing a non-generic command returning a standard Result.
/// </summary>
public interface ICommandHandler<in TCommand> : ICommandHandler<TCommand, Result>
    where TCommand : ICommand
{
}
```

### `IQuery.cs` & `IQueryHandler.cs`

```csharp
namespace SolutionName.SharedKernel.CQRS;

/// <summary>
/// Marker interface representing a CQRS query returning a typed response.
/// </summary>
public interface IQuery<TResponse>
{
}

/// <summary>
/// Handler responsible for executing a specific query.
/// </summary>
public interface IQueryHandler<in TQuery, TResponse>
    where TQuery : IQuery<TResponse>
{
    Task<TResponse> HandleAsync(TQuery query, CancellationToken cancellationToken = default);
}
```

### `IDispatcher.cs` & `Dispatcher.cs`

```csharp
namespace SolutionName.SharedKernel.CQRS;

using Microsoft.Extensions.DependencyInjection;

public interface IDispatcher
{
    Task<TResponse> SendAsync<TResponse>(ICommand<TResponse> command, CancellationToken cancellationToken = default);
    Task<Result> SendAsync(ICommand command, CancellationToken cancellationToken = default);
    Task<TResponse> QueryAsync<TResponse>(IQuery<TResponse> query, CancellationToken cancellationToken = default);
}

public sealed class Dispatcher(IServiceProvider serviceProvider) : IDispatcher
{
    public Task<TResponse> SendAsync<TResponse>(ICommand<TResponse> command, CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(command);

        var handlerType = typeof(ICommandHandler<,>).MakeGenericType(command.GetType(), typeof(TResponse));
        var handler = serviceProvider.GetRequiredService(handlerType);

        var method = handlerType.GetMethod(nameof(ICommandHandler<ICommand<TResponse>, TResponse>.HandleAsync))
            ?? throw new InvalidOperationException($"HandleAsync method not found on handler for {command.GetType().Name}");

        return (Task<TResponse>)method.Invoke(handler, [command, cancellationToken])!;
    }

    public Task<Result> SendAsync(ICommand command, CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(command);

        var handlerType = typeof(ICommandHandler<>).MakeGenericType(command.GetType());
        var handler = serviceProvider.GetRequiredService(handlerType);

        var method = handlerType.GetMethod(nameof(ICommandHandler<ICommand>.HandleAsync))
            ?? throw new InvalidOperationException($"HandleAsync method not found on handler for {command.GetType().Name}");

        return (Task<Result>)method.Invoke(handler, [command, cancellationToken])!;
    }

    public Task<TResponse> QueryAsync<TResponse>(IQuery<TResponse> query, CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(query);

        var handlerType = typeof(IQueryHandler<,>).MakeGenericType(query.GetType(), typeof(TResponse));
        var handler = serviceProvider.GetRequiredService(handlerType);

        var method = handlerType.GetMethod(nameof(IQueryHandler<IQuery<TResponse>, TResponse>.HandleAsync))
            ?? throw new InvalidOperationException($"HandleAsync method not found on handler for {query.GetType().Name}");

        return (Task<TResponse>)method.Invoke(handler, [query, cancellationToken])!;
    }
}
```

---

## 3. Result Pattern Primitives

### `ErrorType.cs` & `Error.cs`

```csharp
namespace SolutionName.SharedKernel.Results;

public enum ErrorType
{
    Failure = 0,
    Validation = 1,
    NotFound = 2,
    Conflict = 3,
    Unauthorized = 4,
    Forbidden = 5,
    InvalidOperation = 6
}

public sealed record Error(string Code, string Description, ErrorType Type)
{
    public static readonly Error None = new(string.Empty, string.Empty, ErrorType.Failure);
    public static readonly Error NullValue = new("Error.NullValue", "The specified result value is null.", ErrorType.Failure);

    public static Error Failure(string code, string description) => new(code, description, ErrorType.Failure);
    public static Error Validation(string code, string description) => new(code, description, ErrorType.Validation);
    public static Error NotFound(string code, string description) => new(code, description, ErrorType.NotFound);
    public static Error Conflict(string code, string description) => new(code, description, ErrorType.Conflict);
    public static Error Unauthorized(string code, string description) => new(code, description, ErrorType.Unauthorized);
    public static Error Forbidden(string code, string description) => new(code, description, ErrorType.Forbidden);
    public static Error InvalidOperation(string code, string description) => new(code, description, ErrorType.InvalidOperation);
}
```

### `Result.cs` & `Result<TValue>.cs`

```csharp
namespace SolutionName.SharedKernel.Results;

public class Result
{
    protected internal Result(bool isSuccess, Error error)
    {
        if (isSuccess && error != Error.None)
            throw new InvalidOperationException("A successful result cannot contain an error.");

        if (!isSuccess && error == Error.None)
            throw new InvalidOperationException("A failed result must contain an error.");

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
}

public sealed class Result<TValue> : Result
{
    private readonly TValue? _value;

    internal Result(TValue? value, bool isSuccess, Error error)
        : base(isSuccess, error)
    {
        _value = value;
    }

    public TValue Value => IsSuccess
        ? _value!
        : throw new InvalidOperationException("The value of a failure result cannot be accessed.");

    public static implicit operator Result<TValue>(TValue? value) =>
        value is not null ? Success(value) : Failure<TValue>(Error.NullValue);

    public static implicit operator Result<TValue>(Error error) => Failure<TValue>(error);
}
```

---

## 4. Domain & Event Primitives

### `Entity.cs` & `AggregateRoot.cs`

```csharp
namespace SolutionName.SharedKernel.Domain;

public abstract class Entity<TId> where TId : notnull
{
    public TId Id { get; protected set; } = default!;

    public override bool Equals(object? obj)
    {
        if (obj is not Entity<TId> other) return false;
        if (ReferenceEquals(this, other)) return true;
        if (GetType() != other.GetType()) return false;
        return EqualityComparer<TId>.Default.Equals(Id, other.Id);
    }

    public override int GetHashCode() => EqualityComparer<TId>.Default.GetHashCode(Id);

    public static bool operator ==(Entity<TId>? left, Entity<TId>? right) => Equals(left, right);
    public static bool operator !=(Entity<TId>? left, Entity<TId>? right) => !Equals(left, right);
}

public abstract class AggregateRoot<TId> : Entity<TId> where TId : notnull
{
    private readonly List<IDomainEvent> _domainEvents = [];
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    public void AddDomainEvent(IDomainEvent domainEvent) => _domainEvents.Add(domainEvent);
    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

### `IDomainEvent.cs` & `IIntegrationEvent.cs`

```csharp
namespace SolutionName.SharedKernel.Domain;

/// <summary>
/// In-process event raised by domain aggregate roots.
/// </summary>
public interface IDomainEvent
{
    Guid EventId => Guid.NewGuid();
    DateTime OccurredOnUtc => DateTime.UtcNow;
}

/// <summary>
/// Cross-boundary event published over the message broker (RabbitMQ/MassTransit).
/// </summary>
public interface IIntegrationEvent
{
    Guid EventId { get; }
    DateTime OccurredOnUtc { get; }
}
```

### `IUnitOfWork.cs`

```csharp
namespace SolutionName.SharedKernel.Domain;

public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

---

## 5. Dependency Injection Registration

```csharp
namespace SolutionName.SharedKernel.Extensions;

using System.Reflection;
using Microsoft.Extensions.DependencyInjection;
using SolutionName.SharedKernel.CQRS;

public static class SharedKernelServiceExtensions
{
    public static IServiceCollection AddSharedKernel(this IServiceCollection services, params Assembly[] assembliesToScan)
    {
        services.AddScoped<IDispatcher, Dispatcher>();

        var assemblies = assembliesToScan.Length > 0 ? assembliesToScan : [Assembly.GetCallingAssembly()];

        foreach (var assembly in assemblies)
        {
            // Register all Command Handlers (Generic & Standard)
            services.Scan(scan => scan
                .FromAssemblies(assembly)
                .AddClasses(classes => classes.AssignableTo(typeof(ICommandHandler<,>)))
                .AsImplementedInterfaces()
                .WithScopedLifetime());

            services.Scan(scan => scan
                .FromAssemblies(assembly)
                .AddClasses(classes => classes.AssignableTo(typeof(ICommandHandler<>)))
                .AsImplementedInterfaces()
                .WithScopedLifetime());

            // Register all Query Handlers
            services.Scan(scan => scan
                .FromAssemblies(assembly)
                .AddClasses(classes => classes.AssignableTo(typeof(IQueryHandler<,>)))
                .AsImplementedInterfaces()
                .WithScopedLifetime());
        }

        return services;
    }
}
```
