# Feature Scaffolding Guide

This reference provides concrete code patterns and templates for scaffolding vertical slice features into an ASP.NET Core solution adhering to `aspnet-core-scaffolding` architectural standards.

---

## 🏗️ 1. Directory Structure

Place all code in feature-oriented folders:

```text
src/
├── ProjectName.Domain/
│   ├── Entities/
│   │   └── SupportTicket.cs
│   ├── Errors/
│   │   └── SupportTicketErrors.cs
│   └── Repositories/
│       ├── ISupportTicketRepository.cs
│       └── ISupportTicketReadRepository.cs
│
├── ProjectName.Application/
│   └── Features/
│       └── SupportTickets/
│           ├── Commands/
│           │   └── CreateSupportTicket/
│           │       ├── CreateSupportTicketCommand.cs
│           │       ├── CreateSupportTicketCommandHandler.cs
│           │       └── CreateSupportTicketCommandValidator.cs
│           └── Queries/
│               └── GetSupportTicketById/
│                   ├── GetSupportTicketByIdQuery.cs
│                   ├── GetSupportTicketByIdQueryHandler.cs
│                   └── SupportTicketDto.cs
│
├── ProjectName.Infrastructure/
│   └── Persistence/
│       ├── Configurations/
│       │   └── SupportTicketConfiguration.cs
│       └── Repositories/
│           ├── SupportTicketRepository.cs
│           └── SupportTicketReadRepository.cs
│
└── ProjectName.Api/
    └── Endpoints/
        └── SupportTicketEndpoints.cs
```

---

## 🏛️ 2. Domain Layer Templates

### Aggregate Root / Entity
```csharp
namespace ProjectName.Domain.Entities;

public sealed class SupportTicket
{
    public Guid Id { get; private set; }
    public string Title { get; private set; } = string.Empty;
    public string Description { get; private set; } = string.Empty;
    public string Status { get; private set; } = "Open";
    public DateTimeOffset CreatedAtUtc { get; private set; }

    private SupportTicket() { } // EF Core private constructor

    public static Result<SupportTicket> Create(string title, string description)
    {
        if (string.IsNullOrWhiteSpace(title))
        {
            return Result.Failure<SupportTicket>(SupportTicketErrors.EmptyTitle);
        }

        if (string.IsNullOrWhiteSpace(description))
        {
            return Result.Failure<SupportTicket>(SupportTicketErrors.EmptyDescription);
        }

        return new SupportTicket
        {
            Id = Guid.NewGuid(),
            Title = title.Trim(),
            Description = description.Trim(),
            Status = "Open",
            CreatedAtUtc = DateTimeOffset.UtcNow
        };
    }
}
```

### Domain Errors Definition
```csharp
namespace ProjectName.Domain.Errors;

public static class SupportTicketErrors
{
    public static readonly Error EmptyTitle = Error.Validation(
        "SupportTicket.EmptyTitle",
        "Support ticket title cannot be empty.");

    public static readonly Error EmptyDescription = Error.Validation(
        "SupportTicket.EmptyDescription",
        "Support ticket description cannot be empty.");

    public static Error NotFound(Guid id) => Error.NotFound(
        "SupportTicket.NotFound",
        $"Support ticket with ID '{id}' was not found.");
}
```

### Domain Repository Contracts
```csharp
namespace ProjectName.Domain.Repositories;

public interface ISupportTicketRepository
{
    Task<SupportTicket?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task AddAsync(SupportTicket ticket, CancellationToken cancellationToken = default);
    void Remove(SupportTicket ticket);
}

public interface ISupportTicketReadRepository
{
    Task<SupportTicketDto?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<SupportTicketListItemDto>> ListAsync(CancellationToken cancellationToken = default);
}
```

---

## ⚡ 3. Application Layer Templates

### Command & Handler
```csharp
namespace ProjectName.Application.Features.SupportTickets.Commands.CreateSupportTicket;

public sealed record CreateSupportTicketCommand(
    string Title,
    string Description) : ICommand<Result<Guid>>;

public sealed class CreateSupportTicketCommandHandler
    : ICommandHandler<CreateSupportTicketCommand, Result<Guid>>
{
    private readonly ISupportTicketRepository _repository;
    private readonly IUnitOfWork _unitOfWork;

    public CreateSupportTicketCommandHandler(
        ISupportTicketRepository repository,
        IUnitOfWork unitOfWork)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
    }

    public async Task<Result<Guid>> HandleAsync(
        CreateSupportTicketCommand command,
        CancellationToken cancellationToken = default)
    {
        var entityResult = SupportTicket.Create(command.Title, command.Description);
        if (entityResult.IsFailure)
        {
            return Result.Failure<Guid>(entityResult.Error);
        }

        var ticket = entityResult.Value;

        await _repository.AddAsync(ticket, cancellationToken);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        return ticket.Id;
    }
}
```

### FluentValidation (v11.6.0)
```csharp
namespace ProjectName.Application.Features.SupportTickets.Commands.CreateSupportTicket;

public sealed class CreateSupportTicketCommandValidator : AbstractValidator<CreateSupportTicketCommand>
{
    public CreateSupportTicketCommandValidator()
    {
        RuleFor(x => x.Title)
            .NotEmpty().WithMessage("Title is required.")
            .MaximumLength(200).WithMessage("Title must not exceed 200 characters.");

        RuleFor(x => x.Description)
            .NotEmpty().WithMessage("Description is required.")
            .MaximumLength(4000).WithMessage("Description must not exceed 4000 characters.");
    }
}
```

### Query & Handler
```csharp
namespace ProjectName.Application.Features.SupportTickets.Queries.GetSupportTicketById;

public sealed record GetSupportTicketByIdQuery(Guid Id) : IQuery<Result<SupportTicketDto>>;

public sealed record SupportTicketDto(
    Guid Id,
    string Title,
    string Description,
    string Status,
    DateTimeOffset CreatedAtUtc);

public sealed class GetSupportTicketByIdQueryHandler
    : IQueryHandler<GetSupportTicketByIdQuery, Result<SupportTicketDto>>
{
    private readonly ISupportTicketReadRepository _readRepository;

    public GetSupportTicketByIdQueryHandler(ISupportTicketReadRepository readRepository)
    {
        _readRepository = readRepository;
    }

    public async Task<Result<SupportTicketDto>> HandleAsync(
        GetSupportTicketByIdQuery query,
        CancellationToken cancellationToken = default)
    {
        var ticket = await _readRepository.GetByIdAsync(query.Id, cancellationToken);
        if (ticket is null)
        {
            return Result.Failure<SupportTicketDto>(SupportTicketErrors.NotFound(query.Id));
        }

        return ticket;
    }
}
```

---

## 🗄️ 4. Infrastructure Layer Templates

### EF Core Entity Configuration (Writes)
```csharp
namespace ProjectName.Infrastructure.Persistence.Configurations;

public sealed class SupportTicketConfiguration : IEntityTypeConfiguration<SupportTicket>
{
    public void Configure(EntityTypeBuilder<SupportTicket> builder)
    {
        builder.ToTable("SupportTickets"); // Configured database table name

        builder.HasKey(x => x.Id);

        builder.Property(x => x.Id)
            .ValueGeneratedNever();

        builder.Property(x => x.Title)
            .HasMaxLength(200)
            .IsRequired();

        builder.Property(x => x.Description)
            .HasMaxLength(4000)
            .IsRequired();

        builder.Property(x => x.Status)
            .HasMaxLength(50)
            .IsRequired();

        builder.Property(x => x.CreatedAtUtc)
            .IsRequired();
    }
}
```

### EF Core Write Repository
```csharp
namespace ProjectName.Infrastructure.Persistence.Repositories;

internal sealed class SupportTicketRepository : ISupportTicketRepository
{
    private readonly ProjectNameDbContext _dbContext;

    public SupportTicketRepository(ProjectNameDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public Task<SupportTicket?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        return _dbContext.Set<SupportTicket>()
            .FirstOrDefaultAsync(x => x.Id == id, cancellationToken);
    }

    public async Task AddAsync(SupportTicket ticket, CancellationToken cancellationToken = default)
    {
        await _dbContext.Set<SupportTicket>().AddAsync(ticket, cancellationToken);
    }

    public void Remove(SupportTicket ticket)
    {
        _dbContext.Set<SupportTicket>().Remove(ticket);
    }
}
```

### Dapper Read Repository (Reads)
```csharp
namespace ProjectName.Infrastructure.Persistence.Repositories;

internal sealed class SupportTicketReadRepository : ISupportTicketReadRepository
{
    private readonly IDbConnectionFactory _connectionFactory;

    public SupportTicketReadRepository(IDbConnectionFactory connectionFactory)
    {
        _connectionFactory = connectionFactory;
    }

    public async Task<SupportTicketDto?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        const string sql = """
            SELECT
                Id,
                Title,
                Description,
                Status,
                CreatedAtUtc
            FROM SupportTickets
            WHERE Id = @Id;
            """;

        using var connection = await _connectionFactory.CreateOpenConnectionAsync(cancellationToken);

        return await connection.QuerySingleOrDefaultAsync<SupportTicketDto>(
            new CommandDefinition(sql, new { Id = id }, cancellationToken: cancellationToken));
    }

    public async Task<IReadOnlyList<SupportTicketListItemDto>> ListAsync(CancellationToken cancellationToken = default)
    {
        const string sql = """
            SELECT
                Id,
                Title,
                Status,
                CreatedAtUtc
            FROM SupportTickets
            ORDER BY CreatedAtUtc DESC;
            """;

        using var connection = await _connectionFactory.CreateOpenConnectionAsync(cancellationToken);

        var items = await connection.QueryAsync<SupportTicketListItemDto>(
            new CommandDefinition(sql, cancellationToken: cancellationToken));

        return items.ToList();
    }
}
```

---

## 🌐 5. API Minimal Endpoints

```csharp
namespace ProjectName.Api.Endpoints;

public static class SupportTicketEndpoints
{
    public static void MapSupportTicketEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/support-tickets")
            .WithTags("SupportTickets");

        group.MapPost("/", CreateTicketAsync)
            .WithName("CreateSupportTicket")
            .WithOpenApi();

        group.MapGet("/{id:guid}", GetTicketByIdAsync)
            .WithName("GetSupportTicketById")
            .WithOpenApi();
    }

    public static async Task<IResult> CreateTicketAsync(
        [FromBody] CreateSupportTicketRequest request,
        [FromServices] IDispatcher dispatcher,
        CancellationToken cancellationToken)
    {
        var command = new CreateSupportTicketCommand(request.Title, request.Description);
        var result = await dispatcher.SendAsync(command, cancellationToken);

        if (result.IsFailure)
        {
            return result.Error.ToProblemDetails();
        }

        return Results.CreatedAtRoute("GetSupportTicketById", new { id = result.Value }, new { id = result.Value });
    }

    public static async Task<IResult> GetTicketByIdAsync(
        [FromRoute] Guid id,
        [FromServices] IDispatcher dispatcher,
        CancellationToken cancellationToken)
    {
        var query = new GetSupportTicketByIdQuery(id);
        var result = await dispatcher.QueryAsync(query, cancellationToken);

        if (result.IsFailure)
        {
            return result.Error.ToProblemDetails();
        }

        return Results.Ok(result.Value);
    }
}

public sealed record CreateSupportTicketRequest(string Title, string Description);
```
