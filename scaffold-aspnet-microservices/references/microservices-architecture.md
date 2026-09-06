# Microservices Architecture Reference

This reference details the design principles, communication patterns, data isolation rules, and architectural standards for building autonomous, scalable ASP.NET Core Microservices.

---

## 1. Core Principles

```text
┌────────────────────────────────────────────────────────────────────────┐
│                              API GATEWAY                               │
│                         (YARP Reverse Proxy)                           │
└───────────────┬────────────────────────┬───────────────────────────────┘
                │                        │
       HTTP     │               HTTP     │
                ▼                        ▼
     ┌──────────────────────┐ ┌──────────────────────┐
     │   Tickets Service    │ │  Customers Service   │
     │  (Clean Architecture)│ │ (Clean Architecture) │
     │  - Domain            │ │ - Domain             │
     │  - Application       │ │ - Application        │
     │  - Infrastructure    │ │ - Infrastructure     │
     │  - Api               │ │ - Api                │
     └──────────┬───────────┘ └──────────┬───────────┘
                │                        │
                │        RabbitMQ        │
                │◄──────────────────────►│
                │ (Async Event Broker)   │
                │                        │
     ┌──────────▼───────────┐ ┌──────────▼───────────┐
     │  tickets_db (Postgres│ │ customers_db (Postgres│
     └──────────────────────┘ └──────────────────────┘
```

1. **Autonomous Services**: Each microservice is an independent, deployable unit implementing Clean Architecture (`Domain`, `Application`, `Infrastructure`, `Api`).
2. **Database-Per-Service**: Microservices NEVER share database tables directly. Data consistency across boundaries is maintained asynchronously via Integration Events.
3. **SharedKernel as Core Foundation**: CQRS abstractions, in-process `IDispatcher`, the `Result` pattern, domain base classes, and cross-cutting behaviors live in the shared class library `SolutionName.SharedKernel`.
4. **Asynchronous by Default**: Inter-service communication favors async Publish/Subscribe using MassTransit over RabbitMQ.
5. **Resilient Synchronous Calls**: When synchronous HTTP queries are required across services, use typed HTTP clients (Refit/HttpClient) configured with `Microsoft.Extensions.Http.Resilience` (retries, circuit breakers, hedging).

---

## 2. Asynchronous Event-Driven Messaging (MassTransit + RabbitMQ)

Integration events cross service boundaries.

### Defining Integration Events in `SharedKernel` or Shared Contracts:

```csharp
namespace SolutionName.SharedKernel.Domain;

public record TicketCreatedIntegrationEvent(
    Guid EventId,
    Guid TicketId,
    Guid CustomerId,
    string Subject,
    string Priority,
    DateTime OccurredOnUtc
) : IIntegrationEvent;
```

### Publishing in Command Handlers:

```csharp
namespace Tickets.Application.Features.Tickets.Commands.CreateTicket;

using MassTransit;
using SolutionName.SharedKernel.CQRS;
using SolutionName.SharedKernel.Domain;
using SolutionName.SharedKernel.Results;
using Tickets.Domain.Tickets;

public sealed class CreateTicketCommandHandler(
    ITicketRepository ticketRepository,
    IUnitOfWork unitOfWork,
    IPublishEndpoint publishEndpoint
) : ICommandHandler<CreateTicketCommand, Result<Guid>>
{
    public async Task<Result<Guid>> HandleAsync(CreateTicketCommand command, CancellationToken cancellationToken = default)
    {
        var ticketResult = Ticket.Create(command.CustomerId, command.Subject, command.Description, command.Priority);
        if (ticketResult.IsFailure)
        {
            return Result.Failure<Guid>(ticketResult.Error);
        }

        var ticket = ticketResult.Value;
        await ticketRepository.AddAsync(ticket, cancellationToken);
        await unitOfWork.SaveChangesAsync(cancellationToken);

        // Publish to RabbitMQ
        await publishEndpoint.Publish(new TicketCreatedIntegrationEvent(
            Guid.NewGuid(),
            ticket.Id,
            ticket.CustomerId,
            ticket.Subject,
            ticket.Priority.ToString(),
            DateTime.UtcNow
        ), cancellationToken);

        return Result.Success(ticket.Id);
    }
}
```

### Consuming in Other Microservices (e.g., Notifications / Analytics):

```csharp
namespace Notifications.Application.Consumers;

using MassTransit;
using SolutionName.SharedKernel.Domain;

public sealed class TicketCreatedConsumer(ILogger<TicketCreatedConsumer> logger) 
    : IConsumer<TicketCreatedIntegrationEvent>
{
    public async Task Consume(ConsumeContext<TicketCreatedIntegrationEvent> context)
    {
        var message = context.Message;
        logger.LogInformation("Processing ticket created notification for TicketId {TicketId}", message.TicketId);
        
        // Handle email / push notification dispatching
        await Task.CompletedTask;
    }
}
```

---

## 3. Transactional Outbox Pattern

To guarantee that a database write and its corresponding integration event succeed or fail atomically, configure the EF Core Outbox with MassTransit in each service's `Infrastructure` layer:

```csharp
services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<ApplicationDbContext>(o =>
    {
        o.UsePostgres();
        o.UseBusOutbox();
    });

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host(configuration["RabbitMQ:Host"], "/", h =>
        {
            h.Username(configuration["RabbitMQ:Username"]);
            h.Password(configuration["RabbitMQ:Password"]);
        });

        cfg.ConfigureEndpoints(context);
    });
});
```
