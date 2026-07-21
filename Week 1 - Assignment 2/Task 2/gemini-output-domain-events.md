## Architectural & Semantic Foundations

### 1. In-Process Domain Events vs. Cross-Boundary Integration Events

| Dimension | Domain Events | Integration Events |
| :--- | :--- | :--- |
| **Scope** | **In-Process / Intra-Bounded-Context.** Confined to a single microservice/process memory space. | **Cross-Process / Inter-Bounded-Context.** Distributed across microservices, teams, or systems. |
| **Consistency Model** | **Strong (or Immediate) Consistency.** Executes inside the *same atomic database transaction* context (Unit of Work). | **Eventual Consistency.** Driven via asynchronous messaging infrastructure (e.g., Azure Service Bus, Kafka, RabbitMQ). |
| **Failure Domain** | If a Domain Event handler throws, the unit-of-work transaction fails and **rolls back the entire command**. | Isolated failure domain; requires retry mechanisms, dead-letter queues (DLQs), and idempotent receivers. |
| **Contract Stability** | Internal implementation detail. High elasticity; refactored freely within domain boundary. | Public API contract. Requires strict schema evolution, semantic versioning, and backward compatibility. |

---

### 2. Consistency Guarantees & Transactional Boundaries

In classical DDD, **a single transaction should modify only one Aggregate Root**. Modifying multiple aggregates within a single database transaction creates heavy lock contention and tight temporal coupling.

However, business rules often demand cross-aggregate invariants within the same domain. Domain events resolve this architectural tension:

1. **Immediate Consistency via In-Process Dispatch:** Handlers run *before* `DbContext.Commit()` / `SaveChangesAsync()`. Aggregate $A$ raises an event, the handler modifies Aggregate $B$, and both state changes commit in **one single $ACID$ transaction**.
2. **Eventual Consistency via Outbox Pattern:** Handlers stage an *Integration Event* into an Outbox table within the same $ACID$ transaction. An out-of-process worker (e.g., BackgroundService/Debezium) polls/streams the Outbox to guarantee **At-Least-Once Delivery** to an external event bus without distributed transactions (2PC/XA).

---

## Technical Implementation (Production-Grade C# / .NET)

### 1. Domain Primitive & Aggregate Root Contracts

Domain Events are **past-tense immutable records** that capture domain state changes. The Aggregate Root owns the event lifecycle queue.

```csharp
namespace Domain.SeedWork;

using MediatR;

/// <summary>
/// Marker contract enforcing MediatR INotification compatibility 
/// for domain event dispatching.
/// </summary>
public interface IDomainEvent : INotification
{
    Guid EventId { get; }
    DateTimeOffset OccurredOn { get; }
}

public abstract record DomainEvent : IDomainEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTimeOffset OccurredOn { get; } = DateTimeOffset.UtcNow;
}

public abstract class AggregateRoot
{
    private readonly List<IDomainEvent> _domainEvents = new();
    
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void RaiseDomainEvent(IDomainEvent domainEvent)
    {
        // Enforce invariants or aggregate domain state here if necessary
        _domainEvents.Add(domainEvent);
    }

    public void ClearDomainEvents() => _domainEvents.Clear();
}

### 2. Domain Logic & Event Emission

Aggregates expose intent-revealing methods (Ubiquitous Language) instead of raw setters, raising domain events upon internal state transition.

namespace Domain.Aggregates.OrderAggregate;

using Domain.SeedWork;

public sealed class Order : AggregateRoot
{
    public Guid Id { get; private set; }
    public Guid BuyerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public decimal TotalAmount { get; private set; }

    private Order() { } // EF Core Proxy / Hydration constructor

    public static Order Create(Guid buyerId, decimal totalAmount)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            BuyerId = buyerId,
            TotalAmount = totalAmount,
            Status = OrderStatus.Submitted
        };

        // Explicit explicit side-effect declaration
        order.RaiseDomainEvent(new OrderStartedDomainEvent(order.Id, buyerId, totalAmount));

        return order;
    }
}

public sealed record OrderStartedDomainEvent(
    Guid OrderId, 
    Guid BuyerId, 
    decimal TotalAmount) : DomainEvent;

### 3. Production EF Core Unit of Work Interception

To guarantee strict transactional atomicity, events are intercepted and dispatched within the persistence boundary before or during SaveChangesAsync.

**Critical Edge Case:** Handling nested events. If an event handler modifies another Aggregate that generates new domain events, a simple ToList() dispatch will miss the second wave of events. The implementation below uses a cascade resolution loop.

namespace Infrastructure.Persistence;

using Domain.SeedWork;
using MediatR;
using Microsoft.EntityFrameworkCore;

public class ApplicationDbContext : DbContext
{
    private readonly IPublisher _publisher;

    public ApplicationDbContext(
        DbContextOptions<ApplicationDbContext> options, 
        IPublisher publisher) : base(options)
    {
        _publisher = publisher ?? throw new ArgumentNullException(nameof(publisher));
    }

    public DbSet<Order> Orders => Set<Order>();

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // 1. Dispatch in-process domain events before committing to the database
        await DispatchDomainEventsAsync(cancellationToken);

        // 2. Commit unit of work atomically
        return await base.SaveChangesAsync(cancellationToken);
    }

    private async Task DispatchDomainEventsAsync(CancellationToken cancellationToken)
    {
        // Cascade resolution loop to process events raised by event handlers themselves
        while (true)
        {
            var domainEntities = ChangeTracker
                .Entries<AggregateRoot>()
                .Where(x => x.Entity.DomainEvents.Any())
                .ToList();

            if (!domainEntities.Any())
                break;

            var domainEvents = domainEntities
                .SelectMany(x => x.Entity.DomainEvents)
                .ToList();

            // Clear events FIRST to avoid re-execution / infinite loops
            domainEntities.ForEach(x => x.Entity.ClearDomainEvents());

            foreach (var domainEvent in domainEvents)
            {
                await _publisher.Publish(domainEvent, cancellationToken);
            }
        }
    }
}

### 4. Decoupled Cross-Aggregate Handlers & Integration Bridging

Handlers reside in the Application Layer. They orchestrate domain state changes across aggregates or bridge local domain events to external transport streams.

#### In-Process Consistency Handler

namespace Application.DomainEventHandlers;

using Domain.Aggregates.OrderAggregate;
using MediatR;

public sealed class OrderStartedDomainEventHandler : INotificationHandler<OrderStartedDomainEvent>
{
    private readonly IBuyerRepository _buyerRepository;

    public OrderStartedDomainEventHandler(IBuyerRepository buyerRepository)
    {
        _buyerRepository = buyerRepository;
    }

    public async Task Handle(OrderStartedDomainEvent notification, CancellationToken cancellationToken)
    {
        var buyer = await _buyerRepository.GetAsync(notification.BuyerId, cancellationToken);
        
        if (buyer is null)
        {
            buyer = Buyer.Create(notification.BuyerId);
            await _buyerRepository.AddAsync(buyer, cancellationToken);
        }

        buyer.VerifyPaymentMethod(notification.TotalAmount);
    }
}

#### Outbox Pattern Bridge (Cross-Boundary Integration)

namespace Application.DomainEventHandlers;

using Domain.Aggregates.OrderAggregate;
using Infrastructure.Outbox;
using MediatR;

public sealed class PublishIntegrationEventOnOrderStartedHandler : INotificationHandler<OrderStartedDomainEvent>
{
    private readonly IOutboxStore _outboxStore;

    public PublishIntegrationEventOnOrderStartedHandler(IOutboxStore outboxStore)
    {
        _outboxStore = outboxStore;
    }

    public async Task Handle(OrderStartedDomainEvent notification, CancellationToken cancellationToken)
    {
        // Convert Domain Event -> Integration Event DTO and stage in Outbox table
        var integrationEvent = new OrderStatusChangedToSubmittedIntegrationEvent(
            notification.OrderId, 
            notification.BuyerId);

        await _outboxStore.AddAsync(integrationEvent, cancellationToken);
    }
}

### Key Architectural Pitfalls & Trade-Offs

1. **Transactional Coupling Risks:**
   Executing synchronous handlers inside the same DB transaction guarantees consistency, but increases overall request latency and database lock duration. Keep domain event handlers computationally light.

2. **Side-Effects with External Services:**
   Never invoke external HTTP services or third-party APIs directly inside an in-process Domain Event handler. If the DB transaction fails and rolls back, the HTTP call cannot be undone. Use the Outbox Pattern or asynchronous integration events for side effects that affect external state.

3. **Ordering Dependency:**
   In-process handlers executed via standard mediator interfaces do not offer guaranteed sequence execution unless explicitly piped. Ensure domain event handlers are stateless and functionally independent of one another.