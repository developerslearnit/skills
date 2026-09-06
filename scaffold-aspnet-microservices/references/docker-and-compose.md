# Docker & Containerization Reference

This reference provides the production-grade multi-stage Dockerfiles and `docker-compose.yml` orchestration for running the ASP.NET Core Microservices system locally or in cloud environments (AKS, ECS, Docker Swarm).

---

## 1. Microservice Multi-Stage `Dockerfile`

Every microservice API project uses an optimized multi-stage build that compiles code, caches NuGet dependencies via Central Package Management, and runs in a lightweight, non-root .NET runtime container.

Place this file at `src/Services/{ServiceName}/{ServiceName}.Api/Dockerfile`:

```dockerfile
# ---------------------------------------------------------
# 1. Base Runtime Image (Distroless / Non-root for security)
# ---------------------------------------------------------
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

# ---------------------------------------------------------
# 2. Build Stage
# ---------------------------------------------------------
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

# Copy Solution-level package configurations
COPY ["Directory.Build.props", "./"]
COPY ["Directory.Packages.props", "./"]

# Copy SharedKernel project file
COPY ["src/Shared/SolutionName.SharedKernel/SolutionName.SharedKernel.csproj", "src/Shared/SolutionName.SharedKernel/"]

# Copy Target Service project files
COPY ["src/Services/Tickets/Tickets.Domain/Tickets.Domain.csproj", "src/Services/Tickets/Tickets.Domain/"]
COPY ["src/Services/Tickets/Tickets.Application/Tickets.Application.csproj", "src/Services/Tickets/Tickets.Application/"]
COPY ["src/Services/Tickets/Tickets.Infrastructure/Tickets.Infrastructure.csproj", "src/Services/Tickets/Tickets.Infrastructure/"]
COPY ["src/Services/Tickets/Tickets.Api/Tickets.Api.csproj", "src/Services/Tickets/Tickets.Api/"]

# Restore dependencies
RUN dotnet restore "src/Services/Tickets/Tickets.Api/Tickets.Api.csproj"

# Copy full source trees
COPY ["src/Shared/SolutionName.SharedKernel/", "src/Shared/SolutionName.SharedKernel/"]
COPY ["src/Services/Tickets/", "src/Services/Tickets/"]

WORKDIR "/src/src/Services/Tickets/Tickets.Api"
RUN dotnet build "Tickets.Api.csproj" -c $BUILD_CONFIGURATION -o /app/build

# ---------------------------------------------------------
# 3. Publish Stage
# ---------------------------------------------------------
FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "Tickets.Api.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

# ---------------------------------------------------------
# 4. Final Runtime Image
# ---------------------------------------------------------
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Tickets.Api.dll"]
```

---

## 2. YARP API Gateway `Dockerfile`

Place this file at `src/Gateways/SolutionName.ApiGateway/Dockerfile`:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

COPY ["Directory.Build.props", "./"]
COPY ["Directory.Packages.props", "./"]
COPY ["src/Gateways/SolutionName.ApiGateway/SolutionName.ApiGateway.csproj", "src/Gateways/SolutionName.ApiGateway/"]

RUN dotnet restore "src/Gateways/SolutionName.ApiGateway/SolutionName.ApiGateway.csproj"
COPY ["src/Gateways/SolutionName.ApiGateway/", "src/Gateways/SolutionName.ApiGateway/"]

WORKDIR "/src/src/Gateways/SolutionName.ApiGateway"
RUN dotnet build "SolutionName.ApiGateway.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "SolutionName.ApiGateway.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "SolutionName.ApiGateway.dll"]
```

---

## 3. Root `docker-compose.yml`

```yaml
version: '3.8'

services:
  # ---------------------------------------------------------
  # Infrastructure: PostgreSQL Database
  # ---------------------------------------------------------
  postgres:
    image: pgvector/pgvector:pg16
    container_name: supportly-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: Password123!
      POSTGRES_DB: supportly_db
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
      - supportly-network

  # ---------------------------------------------------------
  # Infrastructure: Redis Cache
  # ---------------------------------------------------------
  redis:
    image: redis:7-alpine
    container_name: supportly-redis
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - supportly-network

  # ---------------------------------------------------------
  # Infrastructure: RabbitMQ (Event Bus)
  # ---------------------------------------------------------
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: supportly-rabbitmq
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
      - supportly-network

  # ---------------------------------------------------------
  # YARP API Gateway
  # ---------------------------------------------------------
  api-gateway:
    image: supportly/api-gateway:latest
    container_name: supportly-api-gateway
    build:
      context: .
      dockerfile: src/Gateways/SolutionName.ApiGateway/Dockerfile
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_HTTP_PORTS=8080
    depends_on:
      tickets-service:
        condition: service_started
      customers-service:
        condition: service_started
    networks:
      - supportly-network

  # ---------------------------------------------------------
  # Tickets Microservice
  # ---------------------------------------------------------
  tickets-service:
    image: supportly/tickets-service:latest
    container_name: supportly-tickets-service
    build:
      context: .
      dockerfile: src/Services/Tickets/Tickets.Api/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_HTTP_PORTS=8080
      - ConnectionStrings__Database=Host=postgres;Port=5432;Database=tickets_db;Username=postgres;Password=Password123!
      - ConnectionStrings__Redis=redis:6379
      - RabbitMQ__Host=rabbitmq
      - RabbitMQ__Username=guest
      - RabbitMQ__Password=guest
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - supportly-network

  # ---------------------------------------------------------
  # Customers Microservice
  # ---------------------------------------------------------
  customers-service:
    image: supportly/customers-service:latest
    container_name: supportly-customers-service
    build:
      context: .
      dockerfile: src/Services/Customers/Customers.Api/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_HTTP_PORTS=8080
      - ConnectionStrings__Database=Host=postgres;Port=5432;Database=customers_db;Username=postgres;Password=Password123!
      - ConnectionStrings__Redis=redis:6379
      - RabbitMQ__Host=rabbitmq
      - RabbitMQ__Username=guest
      - RabbitMQ__Password=guest
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks:
      - supportly-network

networks:
  supportly-network:
    driver: bridge

volumes:
  postgres_data:
    driver: local
```

---

## 4. `.dockerignore`

```text
**/.git
**/.vs
**/.vscode
**/bin
**/obj
**/*.user
**/*.suo
**/Dockerfile*
**/docker-compose*
**/README.md
```
