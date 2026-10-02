# Part 89: Capstone - Event-Driven Architecture
## ขั้นตอนที่ 881-890: Event-Driven ShopThai

---

## 🎯 เป้าหมายของ Part นี้
- Domain Events vs Integration Events
- Event Store with Event Sourcing
- Event-Driven Saga Orchestration
- CQRS Read Model projection
- Event versioning & schema evolution
- Dead Letter Queue handling
- Event replay

---

## ขั้นตอนที่ 881: Domain Events vs Integration Events

```csharp
// Domain Events: within same bounded context, synchronous
// Integration Events: cross-service, async via message bus

// Domain Event (handled in same transaction)
public record OrderConfirmed(OrderId OrderId, DateTime ConfirmedAt) : IDomainEvent;

// Domain Event Handler (within same DbContext SaveChanges)
public class OrderConfirmedDomainEventHandler : INotificationHandler<OrderConfirmed>
{
    private readonly IInventoryService _inventory;
    
    public async Task Handle(OrderConfirmed evt, CancellationToken ct)
    {
        // Reserve inventory (same transaction as order confirmation)
        await _inventory.ReserveForOrderAsync(evt.OrderId);
    }
}

// Integration Event (published after commit)
public record OrderPlacedIntegrationEvent(
    Guid OrderId,
    Guid CustomerId,
    decimal TotalAmount,
    string Currency,
    IReadOnlyList<OrderLineDto> Lines) : IIntegrationEvent;

// Dispatch after SaveChanges via Outbox Pattern
public class DomainEventDispatcher
{
    public async Task DispatchAndClearAsync(IEnumerable<IDomainEvent> events)
    {
        foreach (var evt in events)
        {
            await _mediator.Publish(evt); // synchronous domain handlers
            
            if (evt is IHasIntegrationEvent hasIntegration)
                await _outbox.EnqueueAsync(hasIntegration.ToIntegrationEvent()); // outbox
        }
    }
}
```

---

## ขั้นตอนที่ 882: Event Sourcing Store

```csharp
// Event Sourcing: store all state changes as events
public class EventStore : IEventStore
{
    private readonly AppDbContext _db;
    
    public async Task AppendAsync(Guid streamId, string streamType, 
        IEnumerable<DomainEvent> events, long expectedVersion)
    {
        // Optimistic concurrency: ensure no other events added since we loaded
        var currentVersion = await _db.EventStream
            .Where(e => e.StreamId == streamId)
            .MaxAsync(e => (long?)e.Version) ?? -1;
        
        if (currentVersion != expectedVersion)
            throw new ConcurrencyException(
                $"Stream {streamId} version conflict: expected {expectedVersion}, got {currentVersion}");
        
        var version = expectedVersion;
        foreach (var evt in events)
        {
            await _db.EventStream.AddAsync(new StoredEvent
            {
                StreamId = streamId,
                StreamType = streamType,
                EventType = evt.GetType().AssemblyQualifiedName!,
                Payload = JsonSerializer.Serialize(evt, evt.GetType()),
                Version = ++version,
                OccurredAt = DateTime.UtcNow,
                CorrelationId = Activity.Current?.Id
            });
        }
        
        await _db.SaveChangesAsync();
    }
    
    public async Task<IEnumerable<DomainEvent>> LoadStreamAsync(Guid streamId, long fromVersion = 0)
    {
        var stored = await _db.EventStream
            .Where(e => e.StreamId == streamId && e.Version >= fromVersion)
            .OrderBy(e => e.Version)
            .ToListAsync();
        
        return stored.Select(e =>
        {
            var type = Type.GetType(e.EventType)!;
            return (DomainEvent)JsonSerializer.Deserialize(e.Payload, type)!;
        });
    }
}

// Event-Sourced Aggregate
public abstract class EventSourcedAggregate
{
    private readonly List<DomainEvent> _uncommittedEvents = new();
    public long Version { get; private set; } = -1;
    
    protected void Apply(DomainEvent evt)
    {
        HandleEvent(evt);
        _uncommittedEvents.Add(evt);
        Version++;
    }
    
    public IReadOnlyList<DomainEvent> GetUncommittedEvents() => _uncommittedEvents;
    
    public void MarkEventsAsCommitted() => _uncommittedEvents.Clear();
    
    // Reconstruct from event history
    public void LoadFromHistory(IEnumerable<DomainEvent> events)
    {
        foreach (var evt in events)
        {
            HandleEvent(evt);
            Version++;
        }
    }
    
    protected abstract void HandleEvent(DomainEvent evt);
}

// Order aggregate using event sourcing
public class OrderAggregate : EventSourcedAggregate
{
    public Guid Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public List<OrderLineSnapshot> Lines { get; private set; } = new();
    
    public void Place(Guid customerId, IEnumerable<OrderLineDto> lines)
        => Apply(new OrderPlaced(Guid.NewGuid(), customerId, lines.ToList(), DateTime.UtcNow));
    
    public void Confirm(string paymentTransactionId)
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("ORDER_NOT_PENDING", "Order is not in pending state");
        Apply(new OrderConfirmedEvent(Id, paymentTransactionId, DateTime.UtcNow));
    }
    
    protected override void HandleEvent(DomainEvent evt)
    {
        switch (evt)
        {
            case OrderPlaced e:
                Id = e.OrderId;
                Status = OrderStatus.Pending;
                Lines = e.Lines.Select(l => new OrderLineSnapshot(l.ProductId, l.Quantity, l.Price)).ToList();
                break;
            case OrderConfirmedEvent:
                Status = OrderStatus.Confirmed;
                break;
            case OrderCancelledEvent:
                Status = OrderStatus.Cancelled;
                break;
        }
    }
}
```

---

## ขั้นตอนที่ 883: Saga Orchestration (Step-by-step)

```csharp
// Order Processing Saga: coordinates multiple services
// Uses State Machine pattern via MassTransit

public class OrderSagaState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = null!;
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public decimal TotalAmount { get; set; }
    public string? PaymentTransactionId { get; set; }
    public string? FailureReason { get; set; }
}

public class OrderSaga : MassTransitStateMachine<OrderSagaState>
{
    public State AwaitingPayment { get; } = null!;
    public State AwaitingInventory { get; } = null!;
    public State AwaitingShipment { get; } = null!;
    public State Completed { get; } = null!;
    public State Failed { get; } = null!;
    
    public Event<OrderPlacedIntegrationEvent> OrderPlaced { get; } = null!;
    public Event<PaymentProcessedEvent> PaymentProcessed { get; } = null!;
    public Event<PaymentFailedEvent> PaymentFailed { get; } = null!;
    public Event<InventoryReservedEvent> InventoryReserved { get; } = null!;
    public Event<ShipmentCreatedEvent> ShipmentCreated { get; } = null!;
    
    public OrderSaga()
    {
        InstanceState(x => x.CurrentState);
        
        Event(() => OrderPlaced, x => x.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => PaymentProcessed, x => x.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => PaymentFailed, x => x.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => InventoryReserved, x => x.CorrelateById(ctx => ctx.Message.OrderId));
        
        Initially(
            When(OrderPlaced)
                .Then(ctx =>
                {
                    ctx.Saga.OrderId = ctx.Message.OrderId;
                    ctx.Saga.CustomerId = ctx.Message.CustomerId;
                    ctx.Saga.TotalAmount = ctx.Message.TotalAmount;
                })
                .Publish(ctx => new ProcessPaymentCommand(ctx.Saga.OrderId, ctx.Saga.TotalAmount))
                .TransitionTo(AwaitingPayment));
        
        During(AwaitingPayment,
            When(PaymentProcessed)
                .Then(ctx => ctx.Saga.PaymentTransactionId = ctx.Message.TransactionId)
                .Publish(ctx => new ReserveInventoryCommand(ctx.Saga.OrderId))
                .TransitionTo(AwaitingInventory),
            
            When(PaymentFailed)
                .Then(ctx => ctx.Saga.FailureReason = ctx.Message.Reason)
                .Publish(ctx => new CancelOrderCommand(ctx.Saga.OrderId, ctx.Saga.FailureReason!))
                .TransitionTo(Failed));
        
        During(AwaitingInventory,
            When(InventoryReserved)
                .Publish(ctx => new CreateShipmentCommand(ctx.Saga.OrderId))
                .TransitionTo(AwaitingShipment));
        
        During(AwaitingShipment,
            When(ShipmentCreated)
                .Publish(ctx => new ConfirmOrderCommand(ctx.Saga.OrderId, ctx.Saga.PaymentTransactionId!))
                .Finalize());
        
        SetCompletedWhenFinalized();
    }
}
```

---

## ขั้นตอนที่ 884: Read Model Projection

```csharp
// Project events into read-optimized views
public class OrderReadModelProjector :
    INotificationHandler<OrderPlaced>,
    INotificationHandler<OrderConfirmedEvent>,
    INotificationHandler<OrderCancelledEvent>
{
    private readonly ReadDbContext _readDb;
    
    public async Task Handle(OrderPlaced evt, CancellationToken ct)
    {
        await _readDb.OrderSummaries.AddAsync(new OrderSummaryReadModel
        {
            OrderId = evt.OrderId,
            CustomerId = evt.CustomerId,
            Status = "Pending",
            TotalAmount = evt.Lines.Sum(l => l.Price * l.Quantity),
            ItemCount = evt.Lines.Count,
            CreatedAt = evt.OccurredAt
        }, ct);
        await _readDb.SaveChangesAsync(ct);
    }
    
    public async Task Handle(OrderConfirmedEvent evt, CancellationToken ct)
    {
        await _readDb.OrderSummaries
            .Where(o => o.OrderId == evt.OrderId)
            .ExecuteUpdateAsync(s => s
                .SetProperty(o => o.Status, "Confirmed")
                .SetProperty(o => o.ConfirmedAt, evt.OccurredAt), ct);
    }
    
    public async Task Handle(OrderCancelledEvent evt, CancellationToken ct)
    {
        await _readDb.OrderSummaries
            .Where(o => o.OrderId == evt.OrderId)
            .ExecuteUpdateAsync(s => s
                .SetProperty(o => o.Status, "Cancelled")
                .SetProperty(o => o.CancelReason, evt.Reason), ct);
    }
}

// Rebuild projections from event store (event replay)
public class ProjectionRebuildService
{
    public async Task RebuildOrderSummariesAsync()
    {
        // Truncate read model
        await _readDb.Database.ExecuteSqlRawAsync("TRUNCATE TABLE order_summaries");
        
        // Replay all Order events
        var events = await _eventStore.LoadAllStreamsByTypeAsync("Order");
        
        foreach (var evt in events.OrderBy(e => e.OccurredAt))
            await _mediator.Publish(evt); // re-runs projectors
    }
}
```

---

## ขั้นตอนที่ 885-890: Event Versioning

```csharp
// Events evolve over time — must handle old versions
public interface IEventUpgrader<TOld, TNew>
    where TOld : IDomainEvent
    where TNew : IDomainEvent
{
    TNew Upgrade(TOld old);
}

// V1 event (original)
public record OrderPlacedV1(Guid OrderId, Guid CustomerId, decimal TotalAmount) : IDomainEvent;

// V2 event (added currency)
public record OrderPlacedV2(Guid OrderId, Guid CustomerId, decimal TotalAmount, string Currency) : IDomainEvent;

// Upgrader
public class OrderPlacedV1ToV2 : IEventUpgrader<OrderPlacedV1, OrderPlacedV2>
{
    public OrderPlacedV2 Upgrade(OrderPlacedV1 old)
        => new(old.OrderId, old.CustomerId, old.TotalAmount, "THB"); // default currency
}

// Event serializer with version handling
public class VersionedEventSerializer
{
    private readonly Dictionary<string, Func<string, IDomainEvent>> _upgraders;
    
    public IDomainEvent Deserialize(string typeName, string json)
    {
        if (_upgraders.TryGetValue(typeName, out var upgrader))
            return upgrader(json); // returns current version
        
        var type = Type.GetType(typeName) 
            ?? throw new InvalidOperationException($"Unknown event type: {typeName}");
        return (IDomainEvent)JsonSerializer.Deserialize(json, type)!;
    }
}

// Dead Letter Queue handling
public class DeadLetterConsumer : IConsumer<Fault<IIntegrationEvent>>
{
    public async Task Consume(ConsumeContext<Fault<IIntegrationEvent>> context)
    {
        var fault = context.Message;
        
        _logger.LogError("Message failed after {RetryCount} retries: {MessageType}. Error: {Error}",
            fault.RetryCount, fault.Message.GetType().Name, fault.Exceptions.First().Message);
        
        // Store in DLQ table for manual inspection
        await _db.DeadLetterMessages.AddAsync(new DeadLetterMessage
        {
            MessageType = fault.Message.GetType().Name,
            Payload = JsonSerializer.Serialize(fault.Message),
            Error = fault.Exceptions.First().Message,
            RetryCount = fault.RetryCount,
            FailedAt = DateTime.UtcNow
        });
        await _db.SaveChangesAsync();
        
        // Alert ops team
        await _alerts.SendAsync($"DLQ message: {fault.Message.GetType().Name}");
    }
}
```

---

## 📝 สรุป Part 89

```
Event-Driven Architecture Patterns:

Domain Events → synchronous, same transaction
Integration Events → async, cross-service (Outbox Pattern)
Event Sourcing → store all state as events, rebuild from history
Saga → orchestrate multi-step processes across services
Read Model → project events into read-optimized views
Event Replay → rebuild projections from scratch
Event Versioning → evolve events without breaking consumers
DLQ → handle failed messages gracefully
```

| Pattern | เมื่อไร |
|---------|---------|
| Domain Events | Business rules ใน aggregate |
| Integration Events | Cross-service communication |
| Event Sourcing | ต้องการ audit trail สมบูรณ์ |
| Saga | Long-running business processes |
| CQRS + Projections | Read/write optimization |

---

**ก่อนหน้า → [Part 88: Capstone Security](part88-capstone-security.md)**  
**ต่อไป → [Part 90: Capstone - ML & AI Integration](part90-capstone-ai-ml.md)**
