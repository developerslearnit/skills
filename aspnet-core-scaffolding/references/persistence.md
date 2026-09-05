# Persistence Reference

## 1. Persistence Strategy

| Concern | Technology | Location |
|---|---|---|
| Write persistence | EF Core | Infrastructure |
| Read persistence | Dapper | Infrastructure |
| Database | SQL Server | Infrastructure |
| Write repository contracts | Interfaces | Domain |
| Read repository contracts | Interfaces | Domain |
| Unit of Work | Interface | Domain |
| Unit of Work implementation | EF Core | Infrastructure |
| SQL queries | Dapper/SQL | Infrastructure |
| EF configurations | EF Core | Infrastructure |
| Migrations | EF Core | Infrastructure |

## 2. EF Core Rules

`DbContext` belongs exclusively to Infrastructure.

```csharp
public sealed class ProjectNameDbContext : DbContext
{
    public DbSet<Customer> Customers => Set<Customer>();

    public ProjectNameDbContext(
        DbContextOptions<ProjectNameDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(
            typeof(ProjectNameDbContext).Assembly);
    }
}
```

Do not inject this type into commands, queries, handlers, controllers, or Domain services.

## 3. EF Core Configurations

Keep EF configuration outside Domain entities.

```csharp
public sealed class CustomerConfiguration
    : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> builder)
    {
        builder.ToTable("Customers");
        builder.HasKey(x => x.Id);
        builder.Property(x=>x.Id)
                .HasDefaultValueSql("NEWID()")
                .ValueGeneratedOnAdd();

        builder.Property(x => x.Name)
            .HasMaxLength(200)
            .IsRequired();

        builder.Property(x => x.Email)
            .HasMaxLength(320)
            .IsRequired();
    }
}
```

Prefer Fluent API over EF attributes on Domain classes.

## 4. Write Repository

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

Implementation belongs in Infrastructure:

```csharp
internal sealed class CustomerRepository : ICustomerRepository
{
    private readonly ProjectNameDbContext _dbContext;

    public CustomerRepository(ProjectNameDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public Task<Customer?> GetByIdAsync(
        Guid id,
        CancellationToken cancellationToken)
    {
        return _dbContext.Customers
            .FirstOrDefaultAsync(x => x.Id == id, cancellationToken);
    }

    public async Task AddAsync(
        Customer customer,
        CancellationToken cancellationToken)
    {
        await _dbContext.Customers.AddAsync(
            customer,
            cancellationToken);
    }

    public void Remove(Customer customer)
    {
        _dbContext.Customers.Remove(customer);
    }
}
```

## 5. Dapper Read Repository

```csharp
public interface ICustomerReadRepository
{
    Task<CustomerDto?> GetByIdAsync(
        Guid id,
        CancellationToken cancellationToken);
}
```

Infrastructure implementation:

```csharp
internal sealed class CustomerReadRepository : ICustomerReadRepository
{
    private readonly IDbConnectionFactory _connectionFactory;

    public CustomerReadRepository(IDbConnectionFactory connectionFactory)
    {
        _connectionFactory = connectionFactory;
    }

    public async Task<CustomerDto?> GetByIdAsync(
        Guid id,
        CancellationToken cancellationToken)
    {
        const string sql = """
            SELECT Id, Name, Email
            FROM Customers
            WHERE Id = @Id;
            """;

        using var connection =
            await _connectionFactory.CreateOpenConnectionAsync(
                cancellationToken);

        return await connection.QuerySingleOrDefaultAsync<CustomerDto>(
            new CommandDefinition(
                sql,
                new { Id = id },
                cancellationToken: cancellationToken));
    }
}
```

The Dapper connection must never reach the handler.

## 6. SQL Rules

Always parameterize SQL.

Good:

```sql
SELECT Id, Name
FROM Customers
WHERE Id = @Id;
```

Bad:

```csharp
$"SELECT Id, Name FROM Customers WHERE Id = '{id}'"
```

Never concatenate user input into SQL.

Prefer explicit column lists over `SELECT *`.

## 7. Stored Procedures

Use stored procedures when required by an existing database, a database-owned business operation, a complex operation, or a documented performance/operational requirement.

Do not wrap every simple CRUD operation in a stored procedure by default.

Stored procedure calls belong in Infrastructure.

## 8. Pagination

List queries should use explicit pagination.

```csharp
public sealed record CustomerSearchParameters(
    string? Search,
    int Page,
    int PageSize);
```

Validate page and page size. Enforce a maximum page size.

## 9. Connection Management

A connection factory may expose a Domain-owned abstraction if handlers or other application components require the capability indirectly, but handlers should normally depend on read/write repository abstractions rather than connection abstractions.

Infrastructure can implement:

```csharp
public interface IDbConnectionFactory
{
    Task<DbConnection> CreateOpenConnectionAsync(
        CancellationToken cancellationToken);
}
```

The concrete factory and SQL Server connection management belong in Infrastructure.

## 10. Read/Write Separation

Write path:

```text
CreateCustomerCommand
    ↓
Customer aggregate
    ↓
ICustomerRepository
    ↓
EF Core
    ↓
SQL Server
```

Read path:

```text
GetCustomerByIdQuery
    ↓
ICustomerReadRepository
    ↓
Dapper
    ↓
SQL Server
    ↓
CustomerDto
```

## 11. Tracking

Use EF Core tracking when an entity is expected to be modified in the current Unit of Work.

For read-only EF Core operations that remain necessary, use `AsNoTracking()`.

The preferred general read path is Dapper.

## 12. Concurrency

Use optimistic concurrency where the domain requires protection from concurrent updates.

Possible mechanisms include SQL Server `rowversion`, EF concurrency tokens, or explicit version columns.

Do not add concurrency complexity without a business reason.

## 13. Database Naming

Follow the existing database naming convention when integrating with an existing schema.

For new databases, establish one consistent convention, for example:

```text
Customers
CustomerId
CreatedAt
UpdatedAt
```

Do not mix naming styles arbitrarily.

## 14. Migrations

EF Core migrations belong to Infrastructure.

Before generating/applying a migration:

1. Confirm the correct DbContext.
2. Confirm the correct startup project.
3. Confirm the target configuration.
4. Build successfully.
5. Review the generated migration.
6. Never silently apply schema changes to production.
