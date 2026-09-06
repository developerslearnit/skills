# Project Structure Reference

This reference defines the file and directory layout for ASP.NET Core Microservices solutions, Central Package Management (`Directory.Packages.props`), and the automation scripts to scaffold the structure from scratch.

---

## 1. Solution Tree Structure

```text
SupportlyAI/
├── src/
│   ├── Shared/
│   │   └── SupportlyAI.SharedKernel/
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
│   │       └── SupportlyAI.SharedKernel.csproj
│   │
│   ├── Gateways/
│   │   └── SupportlyAI.ApiGateway/
│   │       ├── Configuration/
│   │       ├── appsettings.json
│   │       ├── appsettings.Development.json
│   │       ├── Program.cs
│   │       ├── Dockerfile
│   │       └── SupportlyAI.ApiGateway.csproj
│   │
│   └── Services/
│       ├── Tickets/
│       │   ├── Tickets.Domain/
│       │   ├── Tickets.Application/
│       │   ├── Tickets.Infrastructure/
│       │   └── Tickets.Api/
│       │       ├── Dockerfile
│       │       └── Program.cs
│       ├── Customers/
│       │   ├── Customers.Domain/
│       │   ├── Customers.Application/
│       │   ├── Customers.Infrastructure/
│       │   └── Customers.Api/
│       │       ├── Dockerfile
│       │       └── Program.cs
│       └── Knowledge/
│           ├── Knowledge.Domain/
│           ├── Knowledge.Application/
│           ├── Knowledge.Infrastructure/
│           └── Knowledge.Api/
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
├── .gitignore
├── SupportlyAI.sln
└── README.md
```

---

## 2. `Directory.Packages.props` (Central Package Management)

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
  </PropertyGroup>
  <ItemGroup>
    <!-- ASP.NET Core & Runtime -->
    <PackageVersion Include="Microsoft.AspNetCore.OpenApi" Version="10.0.0" />
    <PackageVersion Include="Swashbuckle.AspNetCore" Version="6.5.0" />
    
    <!-- YARP Gateway -->
    <PackageVersion Include="Yarp.ReverseProxy" Version="2.2.0" />

    <!-- Dependency Injection & Scanning -->
    <PackageVersion Include="Scrutor" Version="5.0.2" />

    <!-- Validation -->
    <PackageVersion Include="FluentValidation" Version="11.6.0" />
    <PackageVersion Include="FluentValidation.DependencyInjectionExtensions" Version="11.6.0" />

    <!-- Persistence: PostgreSQL & EF Core -->
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="9.0.0" />
    <PackageVersion Include="Pgvector.EntityFrameworkCore" Version="0.2.1" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="10.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.0" />
    <PackageVersion Include="Dapper" Version="2.1.35" />

    <!-- Messaging: MassTransit & RabbitMQ -->
    <PackageVersion Include="MassTransit" Version="8.3.0" />
    <PackageVersion Include="MassTransit.RabbitMQ" Version="8.3.0" />
    <PackageVersion Include="MassTransit.EntityFrameworkCore" Version="8.3.0" />

    <!-- Distributed Caching & Redis -->
    <PackageVersion Include="Microsoft.Extensions.Caching.StackExchangeRedis" Version="10.0.0" />

    <!-- Telemetry & Health Checks -->
    <PackageVersion Include="AspNetCore.HealthChecks.NpgSql" Version="9.0.0" />
    <PackageVersion Include="AspNetCore.HealthChecks.Rabbitmq" Version="9.0.0" />
    <PackageVersion Include="AspNetCore.HealthChecks.Redis" Version="9.0.0" />
  </ItemGroup>
</Project>
```

---

## 3. `Directory.Build.props`

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  </PropertyGroup>
</Project>
```

---

## 4. Scaffolding CLI Script (PowerShell)

```powershell
$SolutionName = "SupportlyAI"
$Services = @("Tickets", "Customers", "Knowledge")

# 1. Create Solution Root
dotnet new sln -n $SolutionName

# 2. Create SharedKernel
dotnet new classlib -o "src/Shared/$SolutionName.SharedKernel"
dotnet sln add "src/Shared/$SolutionName.SharedKernel/$SolutionName.SharedKernel.csproj"

# 3. Create YARP API Gateway
dotnet new web -o "src/Gateways/$SolutionName.ApiGateway"
dotnet sln add "src/Gateways/$SolutionName.ApiGateway/$SolutionName.ApiGateway.csproj"

# 4. Create Microservices
foreach ($service in $Services) {
    dotnet new classlib -o "src/Services/$service/$service.Domain"
    dotnet new classlib -o "src/Services/$service/$service.Application"
    dotnet new classlib -o "src/Services/$service/$service.Infrastructure"
    dotnet new webapi -o "src/Services/$service/$service.Api"

    # Add to Solution
    dotnet sln add "src/Services/$service/$service.Domain/$service.Domain.csproj"
    dotnet sln add "src/Services/$service/$service.Application/$service.Application.csproj"
    dotnet sln add "src/Services/$service/$service.Infrastructure/$service.Infrastructure.csproj"
    dotnet sln add "src/Services/$service/$service.Api/$service.Api.csproj"

    # Reference Dependencies
    dotnet add "src/Services/$service/$service.Domain/$service.Domain.csproj" reference "src/Shared/$SolutionName.SharedKernel/$SolutionName.SharedKernel.csproj"
    dotnet add "src/Services/$service/$service.Application/$service.Application.csproj" reference "src/Services/$service/$service.Domain/$service.Domain.csproj"
    dotnet add "src/Services/$service/$service.Infrastructure/$service.Infrastructure.csproj" reference "src/Services/$service/$service.Application/$service.Application.csproj"
    dotnet add "src/Services/$service/$service.Api/$service.Api.csproj" reference "src/Services/$service/$service.Infrastructure/$service.Infrastructure.csproj"
}
```
