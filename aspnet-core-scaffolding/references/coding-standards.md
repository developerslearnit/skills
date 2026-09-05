# Coding Standards Reference

## 1. General Principles

Write code that is:

- Explicit
- Readable
- Testable
- Cohesive
- Small in responsibility
- Easy to diagnose
- Consistent with the existing solution

Prefer straightforward code over clever abstractions.

## 2. Naming

Use descriptive names.

Good:

```csharp
ICustomerRepository
ICustomerReadRepository
CreateCustomerCommand
CreateCustomerCommandHandler : ICommandHandler<CreateCustomerCommand, Result<Guid>>
GetCustomerByIdQuery
GetCustomerByIdQueryHandler : IQueryHandler<GetCustomerByIdQuery, Result<CustomerDto>>
```

Avoid:

```csharp
IRepo
IManager
IService2
Helper
Util
```

Use standard C# naming conventions: PascalCase for types/public members, camelCase for parameters/locals, and `_camelCase` for private fields.

## 3. Classes

Prefer one primary responsibility per class.

A command handler implements `ICommandHandler<TCommand, TResponse>` and orchestrates one command use case via `HandleAsync`.

A query handler implements `IQueryHandler<TQuery, TResponse>` and retrieves data for one query use case via `HandleAsync`.

A repository handles persistence for a defined concern.

A validator validates.

A domain entity protects domain invariants.

## 4. Async

Use asynchronous APIs for I/O.

Avoid blocking calls:

```csharp
.Result
.Wait()
.GetAwaiter().GetResult()
```

Do not use `Task.Run` for ordinary database or HTTP I/O.

## 5. CancellationToken

Application and infrastructure I/O should accept and propagate cancellation.

Good:

```csharp
await repository.GetByIdAsync(
    request.Id,
    cancellationToken);
```

Do not replace the caller's token with `CancellationToken.None` unless cancellation is intentionally irrelevant.

## 6. Nullability

Enable nullable reference types.

Represent valid absence explicitly with nullable reference types.

Avoid using the null-forgiving operator `!` merely to silence warnings.

## 7. Collections

Prefer read-only abstractions at boundaries:

```csharp
IReadOnlyList<CustomerDto>
```

Encapsulate mutable collections inside aggregates where domain invariants require controlled modification.

## 8. Exceptions & Error Handling

Do not use exceptions for expected business rule violations, validation failures, or control flow.

Always use the Result pattern (`Result`, `Result<TValue>`, `Error`) for predictable domain and application outcomes.

Use exceptions exclusively for truly unrecoverable or exceptional technical conditions (e.g., database connection loss, hardware failure).

Handle unexpected exceptions centrally with global exception handling middleware (`UseExceptionHandler` or ProblemDetails middleware). Do not catch and swallow exceptions without handling them.

## 9. Validation

Use FluentValidation for request validation.

```csharp
public sealed class CreateCustomerCommandValidator
    : AbstractValidator<CreateCustomerCommand>
{
    public CreateCustomerCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(200);

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress();
    }
}
```

Do not duplicate every validation rule inside handlers. Domain invariants must still be protected by Domain behavior.

## 10. Logging

Use structured logging.

Good:

```csharp
logger.LogInformation(
    "Customer {CustomerId} created.",
    customer.Id);
```

Do not log passwords, tokens, secrets, connection strings, or unnecessary sensitive information.

## 11. API Responses & Result Mapping

Map `Result` and `Result<TValue>` to HTTP responses consistently using an endpoint helper such as `CustomResults.Problem`:

```csharp
public static class CustomResults
{
    public static IResult Problem(Result result)
    {
        if (result.IsSuccess)
        {
            throw new InvalidOperationException("Cannot create problem details for a successful result.");
        }

        return Results.Problem(
            title: GetTitle(result.Error.Type),
            detail: result.Error.Description,
            type: GetType(result.Error.Type),
            statusCode: GetStatusCode(result.Error.Type),
            extensions: new Dictionary<string, object?>
            {
                ["errors"] = new[] { result.Error }
            });
    }

    private static string GetTitle(ErrorType errorType) =>
        errorType switch
        {
            ErrorType.Validation => "Bad Request",
            ErrorType.NotFound => "Not Found",
            ErrorType.Conflict => "Conflict",
            ErrorType.Unauthorized => "Unauthorized",
            ErrorType.Forbidden => "Forbidden",
            _ => "Internal Server Error"
        };

    private static string GetType(ErrorType errorType) =>
        errorType switch
        {
            ErrorType.Validation => "https://tools.ietf.org/html/rfc7231#section-6.5.1",
            ErrorType.NotFound => "https://tools.ietf.org/html/rfc7231#section-6.5.4",
            ErrorType.Conflict => "https://tools.ietf.org/html/rfc7231#section-6.5.8",
            ErrorType.Unauthorized => "https://tools.ietf.org/html/rfc7235#section-3.1",
            ErrorType.Forbidden => "https://tools.ietf.org/html/rfc7231#section-6.5.3",
            _ => "https://tools.ietf.org/html/rfc7231#section-6.6.1"
        };

    private static int GetStatusCode(ErrorType errorType) =>
        errorType switch
        {
            ErrorType.Validation => StatusCodes.Status400BadRequest,
            ErrorType.NotFound => StatusCodes.Status404NotFound,
            ErrorType.Conflict => StatusCodes.Status409Conflict,
            ErrorType.Unauthorized => StatusCodes.Status401Unauthorized,
            ErrorType.Forbidden => StatusCodes.Status403Forbidden,
            _ => StatusCodes.Status500InternalServerError
        };
}
```

Handle unexpected technical exceptions centrally rather than duplicating `try/catch` blocks in endpoints.

## 12. Endpoints & Controllers

Prefer Minimal APIs. Keep endpoints thin. Delegate logic to `IDispatcher` and translate `Result` to HTTP responses using `TypedResults` or endpoint result mapping.

Minimal API examples:

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
        return Results.Problem(
            title: error.Code,
            detail: error.Description,
            statusCode: CustomResults.GetStatusCode(error.Type));
    }

    return Results.CreatedAtRoute("GetCustomerById", new { id = result.Value }, new { id = result.Value });
});

public static async Task<Results<Ok<ChatResponseDto>, ValidationProblem, ProblemHttpResult>> QuickChatAsync(
    [FromBody] ChatRequest request,
    [FromServices] IDispatcher dispatcher,
    CancellationToken cancellationToken)
{
    var command = new ChatWithAgentCommand(string.Empty, request.Message, request.CustomerId);
    var result = await dispatcher.SendAsync(command, cancellationToken);

    if (result.IsFailure)
    {
        var error = result.Error;
        if (error.Type == ErrorType.Validation)
        {
            return TypedResults.ValidationProblem(new Dictionary<string, string[]> { [error.Code] = [error.Description] });
        }
        return TypedResults.Problem(
            title: "Chat Error",
            detail: error.Description,
            statusCode: StatusCodes.Status500InternalServerError);
    }

    return TypedResults.Ok(result.Value);
}
```

Controller example (when controllers are explicitly required):

```csharp
[HttpPost]
public async Task<IActionResult> Create(
    [FromBody] CreateCustomerRequest request,
    [FromServices] IDispatcher dispatcher,
    CancellationToken cancellationToken)
{
    var command = new CreateCustomerCommand(
        request.Name,
        request.Email);

    var result = await dispatcher.SendAsync(
        command,
        cancellationToken);

    return result.Match<IActionResult>(
        id => CreatedAtAction(nameof(GetById), new { id }, new { id }),
        error => Problem(
            detail: error.Description,
            title: error.Code,
            statusCode: CustomResults.GetStatusCode(error.Type)));
}
```

Do not perform database operations in controllers or endpoints.

## 13. Dependency Injection

Prefer constructor injection.

```csharp
public sealed class CustomerService
{
    private readonly ICustomerRepository _repository;

    public CustomerService(ICustomerRepository repository)
    {
        _repository = repository;
    }
}
```

Do not resolve services manually from `IServiceProvider` unless implementing a legitimate framework/infrastructure boundary.

## 14. Configuration

Use strongly typed options for grouped settings.

Avoid spreading raw configuration key access through application/business code.

## 15. Package Management

Use Central Package Management.

Do not add package versions directly to `.csproj` files unless an explicit exception is required.

## 16. Comments

Comments should explain why something unusual exists, business decisions, external constraints, or non-obvious infrastructure behavior.

Avoid comments that simply restate code.

## 17. Tests

Test behavior rather than implementation details.

Prefer:

```text
CreateCustomer_WithDuplicateEmail_ShouldFail
```

over:

```text
CreateCustomer_ShouldCallRepository
```

Mock Domain-owned abstractions for application unit tests. Use integration tests for database-specific behavior.

## 18. Quality Gate

Before declaring a scaffold complete:

```text
[ ] Solution builds
[ ] Tests pass
[ ] No circular project references
[ ] Domain has no Infrastructure dependency
[ ] Application has no Infrastructure dependency
[ ] Handlers inject interfaces only
[ ] CQRS commands implement ICommand<TResponse> / ICommand and queries implement IQuery<TResponse>
[ ] Handlers implement ICommandHandler / IQueryHandler with HandleAsync
[ ] Commands and queries dispatched via in-process IDispatcher (no MediatR dependency)
[ ] Result pattern (Result, Result<TValue>, Error) used for domain operations and CQRS handlers
[ ] No exceptions thrown for expected business logic or control flow
[ ] No DbContext in handlers
[ ] No Dapper in handlers
[ ] No SQL in handlers
[ ] EF Core isolated to Infrastructure
[ ] Dapper isolated to Infrastructure
[ ] Validation registered
[ ] CancellationToken propagated
[ ] Central package management configured
[ ] Configuration is strongly typed
[ ] API endpoints remain thin and map Result failures to Problem Details
```
