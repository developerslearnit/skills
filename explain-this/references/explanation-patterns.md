# Explanation Patterns & Reference Guide

This guide provides specialized templates and structured examples for explaining different categories of code within Clean Architecture and modern .NET applications.

---

## 1. CQRS Command Handler Explanation Template

When explaining a command handler:

### Template

```markdown
### 1. Executive Summary
The `{CommandHandlerName}` orchestrates the `{BusinessAction}` business process. It validates domain invariants, creates or mutates the `{AggregateName}` aggregate, and persists the state change transactionally using `{UnitOfWork}`.

### 2. Contract & Dependencies
- **Command**: `{CommandName}` (carries payload: `{Property1}`, `{Property2}`)
- **Return Type**: `Result<{ReturnType}>` indicating success or functional domain error.
- **Injected Dependencies**:
  - `{IRepository}`: Used to check uniqueness / persist the aggregate.
  - `IUnitOfWork`: Used to commit the transactional boundary.

### 3. Step-by-Step Flow
1. **Precondition & Validation**: Verifies that the command payload satisfies invariants (e.g., unique email or valid status).
2. **Domain Aggregate Operation**: Calls the domain factory or method `{Aggregate}.Create(...)` or `{aggregate}.Update(...)`.
3. **Persistence**: Registers the entity with the repository (`AddAsync` / tracking).
4. **Commit**: Commits changes via `_unitOfWork.SaveChangesAsync(cancellationToken)`.
5. **Outcome**: Returns `Result.Success(id)` on success or propagates domain failure errors.

### 4. Architecture & Design Patterns
- **CQRS**: Separates the state-mutation logic from query representations.
- **Result Pattern**: Encapsulates functional success or failure without throwing costly exceptions.
- **Encapsulated Aggregate Root**: Domain rules remain inside the domain layer, not leaked into the handler.

### 5. Edge Cases & Resilience
- Propagates `CancellationToken` to avoid zombie database queries on client cancellation.
- Returns explicit `Error.Conflict` or `Error.NotFound` if domain invariants fail.
```

---

## 2. CQRS Query Handler Explanation Template

When explaining a query handler:

### Template

```markdown
### 1. Executive Summary
The `{QueryHandlerName}` retrieves `{ReadModelName}` data for `{QueryPurpose}` without mutating state or tracking entities.

### 2. Contract & Data Source
- **Query**: `{QueryName}` (filters: `{Filter1}`, `{Filter2}`)
- **Return Type**: `Result<{DtoName}>`
- **Data Access**: Reads directly from `{Database}` via Dapper SQL / projection for maximum read throughput.

### 3. Step-by-Step Flow
1. **Query Execution**: Issues a parameterized SQL query / read repository call.
2. **Result Mapping**: Maps raw database rows directly to the `{DtoName}` record.
3. **Null Check**: If no record matches the key, returns `Result.Failure(Error.NotFound(...))`.
4. **Outcome**: Returns `Result.Success(dto)`.

### 4. Performance & Best Practices
- **No Tracking Overhead**: Avoids EF Core change tracker overhead for read operations.
- **Explicit Parameterization**: Protects against SQL injection.
- **Read-Only Independence**: Read queries do not depend on domain aggregate classes.
```

---

## 3. Minimal API Endpoint Explanation Template

When explaining an endpoint:

### Template

```markdown
### 1. Executive Summary
The `{EndpointName}` endpoint handles HTTP `{METHOD}` requests to `{Route}`. It receives the transport request, forwards it to the application dispatcher, and maps the resulting `Result<T>` to an appropriate HTTP response.

### 2. HTTP Contract
- **Route**: `{METHOD} {Path}`
- **Parameters**: `[FromBody] {RequestDto}`, `[FromServices] IDispatcher dispatcher`, `CancellationToken`
- **Responses**:
  - `200 OK` / `201 Created` with response body.
  - `400 Bad Request` on validation failure.
  - `404 Not Found` if the requested resource does not exist.
  - `500 Internal Server Error` on unexpected failures.

### 3. Step-by-Step Flow
1. **Transport Binding**: ASP.NET Core binds JSON payload and injects `IDispatcher`.
2. **Dispatching**: Converts request to `{Command/Query}` and invokes `dispatcher.SendAsync(...)` or `dispatcher.QueryAsync(...)`.
3. **Result Matching**: Inspects `result.IsSuccess` vs `result.IsFailure` and returns strongly typed `TypedResults`.
```
