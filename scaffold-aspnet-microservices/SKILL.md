---
name: scaffold-aspnet-microservices
description: Scaffolds enterprise-grade ASP.NET Core Microservices architectures with Clean Architecture per service, a SharedKernel containing CQRS and IDispatcher, YARP API Gateway, and full Docker containerization. Use when creating a distributed microservices system, scaffolding multiple autonomous services, configuring YARP reverse proxy, or setting up Docker Compose.
metadata:
  author: Mark
  version: "1.0"
  stack: "ASP.NET Core, .NET, Microservices, Clean Architecture, CQRS, IDispatcher, SharedKernel, YARP, Docker, Docker Compose, MassTransit, RabbitMQ, PostgreSQL, Dapper"
---

# ASP.NET Core Microservices Scaffolding

## Purpose

Use this skill whenever scaffolding or extending an enterprise-grade ASP.NET Core Microservices architecture. 

The architecture guarantees:
1. **Per-Service Clean Architecture**: Every microservice follows the 4-layer structure (`Domain`, `Application`, `Infrastructure`, `Api`) and database-per-service isolation from `aspnet-core-scaffolding`.
2. **Centralized SharedKernel**: The in-process `IDispatcher`, CQRS abstractions (`ICommand`, `IQuery`, `ICommandHandler`, `IQueryHandler`), Result pattern (`Result`, `Result<TValue>`, `Error`), and Domain primitives live in a reusable `SolutionName.SharedKernel` library.
3. **YARP API Gateway**: Single entry point reverse proxy managing route forwarding, cluster load balancing, rate limiting, and Swagger aggregation.
4. **Complete Docker Orchestration**: Multi-stage `Dockerfile` for each service and gateway, plus root `docker-compose.yml` pre-configured with PostgreSQL, Redis, RabbitMQ, and health checks.
5. **Central Package Management**: Centralized NuGet versions via `Directory.Packages.props` and common build settings via `Directory.Build.props`.

---

## Non-Negotiable Rules

1. **Clean Architecture per Microservice**: Each microservice must be isolated into `Domain`, `Application`, `Infrastructure`, and `Api` projects.
2. **Database-Per-Service**: Microservices NEVER share database tables. Each service owns its schema and database instance or logical database.
3. **SharedKernel for Cross-Cutting CQRS & Primitives**: `IDispatcher`, `Dispatcher`, `ICommand<T>`, `ICommand`, `IQuery<T>`, `ICommandHandler`, `IQueryHandler`, `Result`, `Result<T>`, `Error`, `Entity<TId>`, and `AggregateRoot<TId>` MUST reside in `SolutionName.SharedKernel`.
4. **No MediatR**: Always use the in-process zero-overhead `IDispatcher` from `SharedKernel`.
5. **Result Pattern**: Never throw exceptions for expected business failures. Return `Result` or `Result<TValue>`.
6. **Dual-Persistence per Service**:
   * Use EF Core for transactional writes in `Infrastructure`.
   * Use Dapper with parameterized queries for read projections.
7. **YARP as API Gateway**: Use Microsoft's YARP (`Yarp.ReverseProxy`) for the API Gateway. Never expose internal microservice ports directly to public clients in production.
8. **Asynchronous Messaging with MassTransit**: Inter-service state changes must be broadcast using integration events (`IIntegrationEvent`) via MassTransit over RabbitMQ.
9. **Docker & Compose Ready**: Every service and the gateway must include a multi-stage `Dockerfile`. The root repository must include a `docker-compose.yml` with health checks.
10. **Central Package Management**: All NuGet versions must be specified in `Directory.Packages.props`.

---

## Solution Structure

```text
SolutionName/
├── src/
│   ├── Shared/
│   │   └── SolutionName.SharedKernel/
│   │       ├── CQRS/
│   │       │   ├── ICommand.cs
│   │       │   ├── ICommandHandler.cs
│   │       │   ├── IQuery.cs
│   │       │   ├── IQueryHandler.cs
│   │       │   ├── IDispatcher.cs
│   │       │   └── Dispatcher.cs
│   │       ├── Results/
│   │       │   ├── Error.cs
│   │       │   ├── ErrorType.cs
│   │       │   └── Result.cs
│   │       ├── Domain/
│   │       │   ├── Entity.cs
│   │       │   ├── AggregateRoot.cs
│   │       │   ├── IDomainEvent.cs
│   │       │   ├── IIntegrationEvent.cs
│   │       │   └── IUnitOfWork.cs
│   │       ├── Extensions/
│   │       │   └── SharedKernelServiceExtensions.cs
│   │       └── SolutionName.SharedKernel.csproj
│   │
│   ├── Gateways/
│   │   └── SolutionName.ApiGateway/
│   │       ├── appsettings.json
│   │       ├── Program.cs
│   │       ├── Dockerfile
│   │       └── SolutionName.ApiGateway.csproj
│   │
│   └── Services/
│       ├── ServiceA/
│       │   ├── ServiceA.Domain/
│       │   ├── ServiceA.Application/
│       │   ├── ServiceA.Infrastructure/
│       │   └── ServiceA.Api/
│       │       ├── Dockerfile
│       │       └── Program.cs
│       └── ServiceB/
│           ├── ServiceB.Domain/
│           ├── ServiceB.Application/
│           ├── ServiceB.Infrastructure/
│           └── ServiceB.Api/
│               ├── Dockerfile
│               └── Program.cs
│
├── docker/
│   ├── docker-compose.yml
│   └── docker-compose.override.yml
│
├── Directory.Build.props
├── Directory.Packages.props
├── .editorconfig
├── .dockerignore
├── .gitignore
├── SolutionName.sln
└── README.md
```

---

## Dependency Graph

```text
               ┌───────────────────────────────┐
               │      YARP API Gateway         │
               └───────────────┬───────────────┘
                               │ (HTTP / Routing)
           ┌───────────────────┴───────────────────┐
           ▼                                       ▼
┌──────────────────────┐               ┌──────────────────────┐
│   Service A (API)    │               │   Service B (API)    │
└──────────┬───────────┘               └──────────┬───────────┘
           │                                       │
           ▼                                       ▼
┌──────────────────────┐               ┌──────────────────────┐
│Service A Application │               │Service B Application │
└──────────┬───────────┘               └──────────┬───────────┘
           │                                       │
           ▼                                       ▼
┌──────────────────────┐               ┌──────────────────────┐
│   Service A Domain   │               │   Service B Domain   │
└──────────┬───────────┘               └──────────┬───────────┘
           │                                       │
           └───────────────────┬───────────────────┘
                               │
                               ▼
               ┌───────────────────────────────┐
               │   SolutionName.SharedKernel   │
               │ (CQRS, Dispatcher, Results)   │
               └───────────────────────────────┘
```

---

## SharedKernel Component Details

The `SharedKernel` project is referenced by all microservice `Domain` and `Application` projects.

### 1. CQRS & Dispatcher (`SolutionName.SharedKernel.CQRS`)

```csharp
namespace SolutionName.SharedKernel.CQRS;

using SolutionName.SharedKernel.Results;

public interface ICommand<TResponse> { }
public interface ICommand : ICommand<Result> { }

public interface ICommandHandler<in TCommand, TResponse>
    where TCommand : ICommand<TResponse>
{
    Task<TResponse> HandleAsync(TCommand command, CancellationToken cancellationToken = default);
}

public interface ICommandHandler<in TCommand> : ICommandHandler<TCommand, Result>
    where TCommand : ICommand { }

public interface IQuery<TResponse> { }

public interface IQueryHandler<in TQuery, TResponse>
    where TQuery : IQuery<TResponse>
{
    Task<TResponse> HandleAsync(TQuery query, CancellationToken cancellationToken = default);
}

public interface IDispatcher
{
    Task<TResponse> SendAsync<TResponse>(ICommand<TResponse> command, CancellationToken cancellationToken = default);
    Task<Result> SendAsync(ICommand command, CancellationToken cancellationToken = default);
    Task<TResponse> QueryAsync<TResponse>(IQuery<TResponse> query, CancellationToken cancellationToken = default);
}
```

### 2. Automatic Dependency Injection in Each Microservice

In each microservice `Program.cs`:

```csharp
// Registers IDispatcher and auto-scans all ICommandHandler & IQueryHandler implementations
builder.Services.AddSharedKernel(typeof(Program).Assembly, typeof(SomeApplicationClass).Assembly);
```

---

## YARP API Gateway Configuration

### `src/Gateways/SolutionName.ApiGateway/Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    options.AddFixedWindowLimiter("fixed-policy", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromSeconds(60);
    });
});

builder.Services.AddCors(options =>
{
    options.AddPolicy("GatewayCors", p => p.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader());
});

builder.Services.AddHealthChecks();

var app = builder.Build();

app.UseCors("GatewayCors");
app.UseRateLimiter();
app.MapHealthChecks("/health");
app.MapReverseProxy();

app.Run();
```

### `src/Gateways/SolutionName.ApiGateway/appsettings.json`

```json
{
  "ReverseProxy": {
    "Routes": {
      "tickets-route": {
        "ClusterId": "tickets-cluster",
        "RateLimiterPolicy": "fixed-policy",
        "Match": { "Path": "/api/v1/tickets/{**catch-all}" },
        "Transforms": [{ "PathPattern": "/api/v1/tickets/{**catch-all}" }]
      },
      "customers-route": {
        "ClusterId": "customers-cluster",
        "RateLimiterPolicy": "fixed-policy",
        "Match": { "Path": "/api/v1/customers/{**catch-all}" },
        "Transforms": [{ "PathPattern": "/api/v1/customers/{**catch-all}" }]
      }
    },
    "Clusters": {
      "tickets-cluster": {
        "Destinations": {
          "tickets-svc": { "Address": "http://tickets-service:8080" }
        }
      },
      "customers-cluster": {
        "Destinations": {
          "customers-svc": { "Address": "http://customers-service:8080" }
        }
      }
    }
  }
}
```

---

## Docker & Compose Orchestration

### Microservice `Dockerfile` Template

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

COPY ["Directory.Build.props", "./"]
COPY ["Directory.Packages.props", "./"]
COPY ["src/Shared/SolutionName.SharedKernel/SolutionName.SharedKernel.csproj", "src/Shared/SolutionName.SharedKernel/"]
COPY ["src/Services/{ServiceName}/{ServiceName}.Domain/{ServiceName}.Domain.csproj", "src/Services/{ServiceName}/{ServiceName}.Domain/"]
COPY ["src/Services/{ServiceName}/{ServiceName}.Application/{ServiceName}.Application.csproj", "src/Services/{ServiceName}/{ServiceName}.Application/"]
COPY ["src/Services/{ServiceName}/{ServiceName}.Infrastructure/{ServiceName}.Infrastructure.csproj", "src/Services/{ServiceName}/{ServiceName}.Infrastructure/"]
COPY ["src/Services/{ServiceName}/{ServiceName}.Api/{ServiceName}.Api.csproj", "src/Services/{ServiceName}/{ServiceName}.Api/"]

RUN dotnet restore "src/Services/{ServiceName}/{ServiceName}.Api/{ServiceName}.Api.csproj"
COPY ["src/Shared/SolutionName.SharedKernel/", "src/Shared/SolutionName.SharedKernel/"]
COPY ["src/Services/{ServiceName}/", "src/Services/{ServiceName}/"]

WORKDIR "/src/src/Services/{ServiceName}/{ServiceName}.Api"
RUN dotnet build -c $BUILD_CONFIGURATION -o /app/build
RUN dotnet publish -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "{ServiceName}.Api.dll"]
```

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  postgres:
    image: pgvector/pgvector:pg16
    container_name: app-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: Password123!
      POSTGRES_DB: app_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: app-rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    ports:
      - "5672:5672"
      - "15672:15672"
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    container_name: app-redis
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  api-gateway:
    image: app/api-gateway:latest
    container_name: app-api-gateway
    build:
      context: .
      dockerfile: src/Gateways/SolutionName.ApiGateway/Dockerfile
    ports:
      - "5000:8080"
    depends_on:
      - service-a
      - service-b
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres_data:
    driver: local
```

---

## Scaffolding Execution Workflow for AI Agents

When instructed to scaffold an ASP.NET Core Microservices solution, follow these steps:

1. **Discover Solution & Service Names**:
   * Clarify the solution name (e.g. `SupportlyAI`).
   * Clarify the initial microservices to scaffold (e.g. `Tickets`, `Customers`, `KnowledgeBase`, `Identity`).
   * Clarify the database engine (PostgreSQL + pgvector or SQL Server).
2. **Generate Root Package Configurations**:
   * Create `Directory.Build.props` (target `net10.0`, nullable enabled, treat warnings as errors).
   * Create `Directory.Packages.props` with all central package versions.
   * Create `.editorconfig`, `.gitignore`, and `.dockerignore`.
3. **Scaffold SharedKernel**:
   * Create `src/Shared/{SolutionName}.SharedKernel`.
   * Implement `CQRS/` (`ICommand`, `IQuery`, `ICommandHandler`, `IQueryHandler`, `IDispatcher`, `Dispatcher`).
   * Implement `Results/` (`Result`, `Result<TValue>`, `Error`, `ErrorType`).
   * Implement `Domain/` (`Entity<TId>`, `AggregateRoot<TId>`, `IDomainEvent`, `IIntegrationEvent`, `IUnitOfWork`).
   * Implement `Extensions/` (`SharedKernelServiceExtensions`).
4. **Scaffold YARP API Gateway**:
   * Create `src/Gateways/{SolutionName}.ApiGateway`.
   * Configure YARP routes, clusters, rate limiting, and CORS in `appsettings.json` and `Program.cs`.
   * Create multi-stage `Dockerfile`.
5. **Scaffold Each Microservice**:
   * Create `Domain`, `Application`, `Infrastructure`, and `Api` projects for each bounded context.
   * Connect project references: `Domain -> SharedKernel`, `Application -> Domain`, `Infrastructure -> Application`, `Api -> Infrastructure`.
   * Register `AddSharedKernel()`, EF Core `DbContext`, Dapper `IDbConnectionFactory`, and MassTransit in `Infrastructure`/`Program.cs`.
   * Create multi-stage `Dockerfile` per service.
6. **Generate Docker Compose**:
   * Create root `docker-compose.yml` and `docker-compose.override.yml`.
7. **Build & Verify**:
   * Run `dotnet build` to ensure the entire multi-service solution compiles with 0 errors and 0 warnings.

---

## References

- [Microservices Architecture Reference](references/microservices-architecture.md)
- [SharedKernel Implementation Details](references/shared-kernel.md)
- [YARP API Gateway Setup & Routing](references/yarp-gateway.md)
- [Docker & Compose Orchestration](references/docker-and-compose.md)
- [Project Structure & Scaffolding Scripts](references/project-structure.md)
