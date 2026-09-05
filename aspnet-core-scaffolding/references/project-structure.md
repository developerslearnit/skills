# Project Structure Reference

## 1. Repository Layout

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

Do not create empty folders merely to satisfy a template.

## 2. Domain

```text
ProjectName.Domain/
├── Common/
│   ├── Entity.cs
│   ├── AggregateRoot.cs
│   ├── ValueObject.cs
│   ├── Result.cs
│   ├── Error.cs
│   └── ErrorType.cs
├── Entities/
│   └── Customer.cs
├── ValueObjects/
│   └── EmailAddress.cs
├── Events/
│   └── CustomerCreatedDomainEvent.cs
├── Errors/
│   └── CustomerErrors.cs
├── Exceptions/
│   └── DomainException.cs
├── Repositories/
│   ├── ICustomerRepository.cs
│   └── ICustomerReadRepository.cs
├── UnitOfWork/
│   └── IUnitOfWork.cs
└── Enums/
```

Adjust according to the actual domain.

## 3. Application

```text
ProjectName.Application/
├── Common/
│   └── Messaging/
│       ├── ICommand.cs
│       ├── ICommandHandler.cs
│       ├── IQuery.cs
│       ├── IQueryHandler.cs
│       ├── IDispatcher.cs
│       └── Dispatcher.cs
├── Behaviors/
│   ├── ValidationBehavior.cs
│   └── LoggingBehavior.cs
├── Features/
│   └── Customers/
│       ├── Commands/
│       │   ├── CreateCustomer/
│       │   │   ├── CreateCustomerCommand.cs
│       │   │   ├── CreateCustomerCommandHandler.cs
│       │   │   └── CreateCustomerCommandValidator.cs
│       │   └── UpdateCustomer/
│       │       ├── UpdateCustomerCommand.cs
│       │       ├── UpdateCustomerCommandHandler.cs
│       │       └── UpdateCustomerCommandValidator.cs
│       └── Queries/
│           ├── GetCustomerById/
│           │   ├── GetCustomerByIdQuery.cs
│           │   └── GetCustomerByIdQueryHandler.cs
│           └── SearchCustomers/
│               ├── SearchCustomersQuery.cs
│               └── SearchCustomersQueryHandler.cs
├── DTOs/
│   └── CustomerDto.cs
└── DependencyInjection.cs
```

If a DTO belongs exclusively to one feature, prefer keeping it within that feature.

## 4. Infrastructure

```text
ProjectName.Infrastructure/
├── Persistence/
│   ├── ProjectNameDbContext.cs
│   ├── Configurations/
│   │   └── CustomerConfiguration.cs
│   ├── Repositories/
│   │   ├── CustomerRepository.cs
│   │   └── CustomerReadRepository.cs
│   ├── Connection/
│   │   ├── IDbConnectionFactory.cs
│   │   └── SqlConnectionFactory.cs
│   └── Migrations/
├── Services/
├── ExternalServices/
└── DependencyInjection.cs
```

Keep Infrastructure implementations organized by technical responsibility.

## 5. API

```text
ProjectName.Api/
├── Common/
│   └── CustomResults.cs
├── Endpoints/
│   └── CustomersEndpoints.cs
├── Middleware/
│   ├── ExceptionHandlingMiddleware.cs
│   └── CorrelationMiddleware.cs
├── Extensions/
│   └── ServiceCollectionExtensions.cs
├── Configuration/
├── Program.cs
└── appsettings.json
```


## 6. Tests

### Domain Tests

Test entities, value objects, domain services, domain events, and business rules.

### Application Tests

Test handlers, validators, pipeline behaviors, and application orchestration using Domain-owned abstractions.

### Integration Tests

Test EF Core mappings, Dapper queries, repositories, SQL Server persistence, and important HTTP paths.

Prefer realistic database integration tests for persistence behavior.

## 7. Project References

Expected references:

```text
Domain
  └── none

Application
  └── Domain

Infrastructure
  ├── Domain
  └── Application

Api
  ├── Application
  └── Infrastructure
```

Tests reference only what they need. Avoid circular references.

## 8. Dependency Injection Composition

Application:

```csharp
services.AddApplication();
```

Infrastructure:

```csharp
services.AddInfrastructure(configuration);
```

API composition:

```csharp
builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);
```

Keep registration close to the project that owns the implementation.

## 9. Configuration

Use strongly typed options for grouped configuration.

```csharp
public sealed class DatabaseOptions
{
    public const string SectionName = "Database";

    public string ConnectionString { get; init; } = string.Empty;
}
```

Do not scatter raw configuration keys throughout business code.

## 10. Environment Configuration

Use environment-specific configuration appropriately:

```text
appsettings.json
appsettings.Development.json
appsettings.Staging.json
appsettings.Production.json
```

Secrets should be supplied through appropriate secret management rather than committed to source control.

## 11. Directory.Build.props

Use it for shared build settings such as nullable reference types, implicit usings, common analyzers, warning policy, and other repository-wide compiler settings.

Example:

```xml
<Project>
	<PropertyGroup>
		<TargetFramework>net10.0</TargetFramework>
		<ImplicitUsings>enable</ImplicitUsings>
		<Nullable>enable</Nullable>
		<AnalysisLevel>latest</AnalysisLevel>
		<AnalysisMode>All</AnalysisMode>
		<TreatWarningsAsErrors>true</TreatWarningsAsErrors>
		<CodeAnalysisTreatWarningsAsErrors>true</CodeAnalysisTreatWarningsAsErrors>
		<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
	</PropertyGroup>

	<ItemGroup Condition="'$(MSBuildProjectExtension)' != '.dcproj'">
		<PackageReference Include="SonarAnalyzer.CSharp">
			<PrivateAssets>all</PrivateAssets>
			<IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
		</PackageReference>
	</ItemGroup>
</Project>
```

## 12. Directory.Packages.props

Use Central Package Management.

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>

  <ItemGroup>
    <PackageVersion Include="FluentValidation" Version="..." />
    <PackageVersion Include="FluentValidation.DependencyInjectionExtensions" Version="..." />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.SqlServer" Version="..." />
    <PackageVersion Include="Dapper" Version="..." />
    <PackageVersion Include="Scrutor" Version="..." />
  </ItemGroup>
</Project>
```

Use package versions compatible with the selected target framework. Do not invent versions when the repository already establishes them.

## 13. Naming

Prefer:

```text
CreateCustomerCommand
CreateCustomerCommandHandler
CreateCustomerCommandValidator
GetCustomerByIdQuery
GetCustomerByIdQueryHandler
ICustomerRepository
ICustomerReadRepository
CustomerRepository
CustomerReadRepository
```

Avoid vague names such as `Manager`, `Helper`, `Utility`, `CommonService`, or `RepositoryBase` unless the responsibility genuinely requires them.
