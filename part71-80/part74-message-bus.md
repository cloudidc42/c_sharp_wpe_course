# Part 74: Message Bus & Event-Driven Architecture
## ขั้นตอนที่ 731-740: Messaging Patterns ใน .NET

---

## 🎯 เป้าหมายของ Part นี้
- Message Bus คืออะไร
- RabbitMQ กับ .NET
- Azure Service Bus
- Apache Kafka
- MassTransit abstraction layer
- Publish/Subscribe, Request/Response, Competing Consumers
- Dead Letter Queue (DLQ)
- Message deduplication & idempotency

---

## ขั้นตอนที่ 731: Message Bus Concepts

```
Message Broker Architecture:
─────────────────────────────────────────────

  Producer ──► [ Exchange/Topic ] ──► [ Queue ] ──► Consumer
                                            │
                                            └──► Dead Letter Queue (on failure)

Patterns:
  - Publish/Subscribe: 1 producer → many consumers
  - Point-to-Point:    1 producer → 1 consumer
  - Request/Reply:     synchronous-like over async messaging
  - Fan-out:           broadcast to all subscribers
  - Competing Consumers: multiple consumers from same queue (load balance)
```

```csharp
// When to use message bus:
// ✅ Decoupling services
// ✅ Handling spikes (buffer requests)
// ✅ Reliable async processing (retry, DLQ)
// ✅ Event-driven workflows
// ❌ Real-time (< 100ms) - use SignalR instead
// ❌ Simple request-response in same process
```

---

## ขั้นตอนที่ 732: RabbitMQ ด้วย MassTransit

```csharp
// Install:
// dotnet add package MassTransit
// dotnet add package MassTransit.RabbitMQ

// Shared contracts (separate project)
namespace MyApp.Contracts;

public record OrderCreated(
    Guid OrderId,
    string CustomerId,
    decimal Amount,
    DateTime CreatedAt);

public record OrderShipped(
    Guid OrderId,
    string TrackingNumber,
    DateTime ShippedAt);

// Program.cs - Producer (Order Service)
builder.Services.AddMassTransit(x =>
{
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("rabbitmq://localhost", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
        
        cfg.ConfigureEndpoints(ctx);
    });
});

// Publish an event
public class OrderService
{
    private readonly IPublishEndpoint _publishEndpoint;
    
    public OrderService(IPublishEndpoint publishEndpoint) => _publishEndpoint = publishEndpoint;
    
    public async Task CreateOrderAsync(CreateOrderRequest request)
    {
        // Save to DB...
        var order = new Order { Id = Guid.NewGuid() };
        
        // Publish event to message bus
        await _publishEndpoint.Publish(new OrderCreated(
            OrderId: order.Id,
            CustomerId: request.CustomerId,
            Amount: request.Amount,
            CreatedAt: DateTime.UtcNow));
    }
}
```

---

## ขั้นตอนที่ 733: MassTransit Consumer

```csharp
// Consumer (Notification Service)
public class OrderCreatedConsumer : IConsumer<OrderCreated>
{
    private readonly IEmailService _email;
    private readonly ILogger<OrderCreatedConsumer> _logger;
    
    public OrderCreatedConsumer(IEmailService email, ILogger<OrderCreatedConsumer> logger)
    { _email = email; _logger = logger; }
    
    public async Task Consume(ConsumeContext<OrderCreated> context)
    {
        var order = context.Message;
        _logger.LogInformation("Processing order: {OrderId}", order.OrderId);
        
        await _email.SendOrderConfirmationAsync(order.CustomerId, order.OrderId);
    }
}

// Consumer definition (configure retry, concurrency, etc.)
public class OrderCreatedConsumerDefinition : ConsumerDefinition<OrderCreatedConsumer>
{
    public OrderCreatedConsumerDefinition()
    {
        EndpointName = "notification-order-created";
        ConcurrentMessageLimit = 10;
    }
    
    protected override void ConfigureConsumer(
        IReceiveEndpointConfigurator endpointConfigurator,
        IConsumerConfigurator<OrderCreatedConsumer> consumerConfigurator,
        IRegistrationContext context)
    {
        endpointConfigurator.UseMessageRetry(r =>
        {
            r.Exponential(5, TimeSpan.FromSeconds(1), TimeSpan.FromMinutes(1), TimeSpan.FromSeconds(2));
            r.Ignore<ValidationException>();
        });
        
        // Dead Letter Queue after all retries exhausted
        endpointConfigurator.UseDeadLetterQueue();
        
        endpointConfigurator.UseInMemoryOutbox(context);
    }
}

// Program.cs - Consumer service
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderCreatedConsumer, OrderCreatedConsumerDefinition>();
    
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("rabbitmq://localhost");
        cfg.ConfigureEndpoints(ctx);
    });
});
```

---

## ขั้นตอนที่ 734: Request/Response Pattern

```csharp
// Request/Response: like RPC but over message bus
public record GetProductRequest(int ProductId);
public record GetProductResponse(int ProductId, string Name, decimal Price, bool Found);

// Request handler (consumer)
public class GetProductConsumer : IConsumer<GetProductRequest>
{
    private readonly IProductRepository _repo;
    
    public GetProductConsumer(IProductRepository repo) => _repo = repo;
    
    public async Task Consume(ConsumeContext<GetProductRequest> context)
    {
        var product = await _repo.GetByIdAsync(context.Message.ProductId);
        
        await context.RespondAsync(product != null
            ? new GetProductResponse(product.Id, product.Name, product.Price, true)
            : new GetProductResponse(context.Message.ProductId, "", 0, false));
    }
}

// Client (making request)
public class ProductQueryService
{
    private readonly IRequestClient<GetProductRequest> _client;
    
    public ProductQueryService(IRequestClient<GetProductRequest> client) => _client = client;
    
    public async Task<GetProductResponse> GetProductAsync(int productId)
    {
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
        
        var response = await _client.GetResponse<GetProductResponse>(
            new GetProductRequest(productId),
            cts.Token);
        
        return response.Message;
    }
}

// Register request client
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<GetProductConsumer>();
    x.AddRequestClient<GetProductRequest>(new Uri("queue:product-catalog"));
    
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("rabbitmq://localhost");
        cfg.ConfigureEndpoints(ctx);
    });
});
```

---

## ขั้นตอนที่ 735: Saga / State Machine

```csharp
// Saga: coordinate long-running business processes across services
// Example: Order Processing Saga (Order → Payment → Inventory → Shipping)

public class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = null!;
    public Guid OrderId { get; set; }
    public decimal Amount { get; set; }
    public string? PaymentTransactionId { get; set; }
    public DateTime CreatedAt { get; set; }
}

public class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    public State Submitted { get; private set; } = null!;
    public State PaymentPending { get; private set; } = null!;
    public State InventoryReserved { get; private set; } = null!;
    public State Completed { get; private set; } = null!;
    public State Cancelled { get; private set; } = null!;
    
    public Event<OrderCreated> OrderCreatedEvent { get; private set; } = null!;
    public Event<PaymentCompleted> PaymentCompletedEvent { get; private set; } = null!;
    public Event<PaymentFailed> PaymentFailedEvent { get; private set; } = null!;
    public Event<InventoryReserved> InventoryReservedEvent { get; private set; } = null!;
    
    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState);
        
        // Correlate events to saga instance
        Event(() => OrderCreatedEvent, e => e.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => PaymentCompletedEvent, e => e.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => PaymentFailedEvent, e => e.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => InventoryReservedEvent, e => e.CorrelateById(ctx => ctx.Message.OrderId));
        
        Initially(
            When(OrderCreatedEvent)
                .Then(ctx =>
                {
                    ctx.Saga.OrderId = ctx.Message.OrderId;
                    ctx.Saga.Amount = ctx.Message.Amount;
                    ctx.Saga.CreatedAt = DateTime.UtcNow;
                })
                .PublishAsync(ctx => ctx.Init<ProcessPayment>(new
                {
                    ctx.Message.OrderId,
                    ctx.Message.Amount
                }))
                .TransitionTo(PaymentPending)
        );
        
        During(PaymentPending,
            When(PaymentCompletedEvent)
                .Then(ctx => ctx.Saga.PaymentTransactionId = ctx.Message.TransactionId)
                .PublishAsync(ctx => ctx.Init<ReserveInventory>(new { ctx.Message.OrderId }))
                .TransitionTo(InventoryReserved),
            
            When(PaymentFailedEvent)
                .PublishAsync(ctx => ctx.Init<OrderCancelled>(new
                {
                    ctx.Message.OrderId,
                    Reason = "Payment failed"
                }))
                .TransitionTo(Cancelled)
        );
        
        During(InventoryReserved,
            When(InventoryReservedEvent)
                .TransitionTo(Completed)
                .Finalize()
        );
    }
}
```

---

## ขั้นตอนที่ 736: Azure Service Bus

```csharp
// Azure Service Bus: managed, enterprise-grade message broker
// Install: dotnet add package MassTransit.Azure.ServiceBus.Core

builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderCreatedConsumer, OrderCreatedConsumerDefinition>();
    
    x.UsingAzureServiceBus((ctx, cfg) =>
    {
        cfg.Host(builder.Configuration.GetConnectionString("AzureServiceBus"));
        
        // Topic subscriptions
        cfg.SubscriptionEndpoint<OrderCreated>(
            "notification-service",
            e =>
            {
                e.ConfigureConsumer<OrderCreatedConsumer>(ctx);
                e.MaxDeliveryCount = 5; // DLQ after 5 retries
            });
        
        cfg.ConfigureEndpoints(ctx);
    });
});

// Direct Azure SDK usage
public class ServiceBusPublisher : IAsyncDisposable
{
    private readonly ServiceBusClient _client;
    private readonly ServiceBusSender _sender;
    
    public ServiceBusPublisher(string connectionString, string topicName)
    {
        _client = new ServiceBusClient(connectionString);
        _sender = _client.CreateSender(topicName);
    }
    
    public async Task PublishAsync<T>(T message, string? correlationId = null)
    {
        var body = JsonSerializer.SerializeToUtf8Bytes(message);
        var sbMessage = new ServiceBusMessage(body)
        {
            ContentType = "application/json",
            CorrelationId = correlationId ?? Guid.NewGuid().ToString(),
            Subject = typeof(T).Name,
            MessageId = Guid.NewGuid().ToString()
        };
        
        sbMessage.ApplicationProperties["MessageType"] = typeof(T).FullName;
        
        await _sender.SendMessageAsync(sbMessage);
    }
    
    public async ValueTask DisposeAsync()
    {
        await _sender.DisposeAsync();
        await _client.DisposeAsync();
    }
}
```

---

## ขั้นตอนที่ 737: Apache Kafka

```csharp
// Kafka: high-throughput, log-based message streaming
// Install: dotnet add package Confluent.Kafka
// or: dotnet add package MassTransit.Kafka

// Kafka producer
public class KafkaOrderProducer : IAsyncDisposable
{
    private readonly IProducer<string, string> _producer;
    
    public KafkaOrderProducer(string bootstrapServers)
    {
        var config = new ProducerConfig
        {
            BootstrapServers = bootstrapServers,
            Acks = Acks.All,
            EnableIdempotence = true,
            MessageTimeoutMs = 5000
        };
        _producer = new ProducerBuilder<string, string>(config).Build();
    }
    
    public async Task PublishAsync<T>(string topic, string key, T message)
    {
        var value = JsonSerializer.Serialize(message);
        var kafkaMsg = new Message<string, string> { Key = key, Value = value };
        
        var result = await _producer.ProduceAsync(topic, kafkaMsg);
        Console.WriteLine($"Delivered to {result.TopicPartitionOffset}");
    }
    
    public async ValueTask DisposeAsync()
    {
        _producer.Flush(TimeSpan.FromSeconds(10));
        _producer.Dispose();
        await ValueTask.CompletedTask;
    }
}

// Kafka consumer
public class KafkaOrderConsumer : BackgroundService
{
    private readonly IConsumer<string, string> _consumer;
    private readonly ILogger<KafkaOrderConsumer> _logger;
    
    public KafkaOrderConsumer(string bootstrapServers, ILogger<KafkaOrderConsumer> logger)
    {
        _logger = logger;
        var config = new ConsumerConfig
        {
            BootstrapServers = bootstrapServers,
            GroupId = "notification-service",
            AutoOffsetReset = AutoOffsetReset.Earliest,
            EnableAutoCommit = false // manual commit for at-least-once
        };
        _consumer = new ConsumerBuilder<string, string>(config).Build();
        _consumer.Subscribe("orders");
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await Task.Run(() =>
        {
            while (!stoppingToken.IsCancellationRequested)
            {
                var result = _consumer.Consume(stoppingToken);
                try
                {
                    var order = JsonSerializer.Deserialize<OrderCreated>(result.Message.Value);
                    _logger.LogInformation("Consumed: {OrderId}", order?.OrderId);
                    // Process message...
                    _consumer.Commit(result); // commit only after success
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Error processing message");
                }
            }
        }, stoppingToken);
    }
    
    public override void Dispose()
    {
        _consumer.Close();
        _consumer.Dispose();
        base.Dispose();
    }
}
```

---

## ขั้นตอนที่ 738-740: Dead Letter Queue & Error Handling

```csharp
// Dead Letter Queue (DLQ): messages that failed processing go here
public class OrderCreatedFaultConsumer : IConsumer<Fault<OrderCreated>>
{
    private readonly ILogger<OrderCreatedFaultConsumer> _logger;
    private readonly IAlertService _alerts;
    
    public OrderCreatedFaultConsumer(ILogger<OrderCreatedFaultConsumer> logger, IAlertService alerts)
    { _logger = logger; _alerts = alerts; }
    
    public async Task Consume(ConsumeContext<Fault<OrderCreated>> context)
    {
        var fault = context.Message;
        var original = fault.Message;
        
        _logger.LogError("Failed to process OrderCreated for {OrderId}. Exceptions: {Exceptions}",
            original.OrderId,
            string.Join(", ", fault.Exceptions.Select(e => e.Message)));
        
        // Alert ops team
        await _alerts.SendCriticalAlertAsync(
            $"Order processing failed: {original.OrderId}",
            fault.Exceptions.First().Message);
        
        // Could retry manually, or save for manual inspection
    }
}

// Message deduplication with idempotency key
public class IdempotentOrderConsumer : IConsumer<OrderCreated>
{
    private readonly IProcessedMessageRepository _processed;
    private readonly IOrderService _orderService;
    
    public IdempotentOrderConsumer(IProcessedMessageRepository processed, IOrderService orderService)
    { _processed = processed; _orderService = orderService; }
    
    public async Task Consume(ConsumeContext<OrderCreated> context)
    {
        var messageId = context.MessageId?.ToString() ?? context.Message.OrderId.ToString();
        
        // Check if already processed (idempotency)
        if (await _processed.IsProcessedAsync(messageId))
        {
            context.LogSkipped(); // skip duplicate
            return;
        }
        
        await _orderService.ProcessAsync(context.Message);
        await _processed.MarkProcessedAsync(messageId, TimeSpan.FromDays(7));
    }
}

// Outbox pattern: guarantee publish with DB transaction
public class OrderCommandHandler
{
    private readonly AppDbContext _db;
    private readonly IPublishEndpoint _publish;
    
    public async Task HandleAsync(CreateOrderCommand cmd)
    {
        using var transaction = await _db.Database.BeginTransactionAsync();
        try
        {
            var order = new Order { /* ... */ };
            _db.Orders.Add(order);
            
            // Save outbox message in same transaction
            _db.OutboxMessages.Add(new OutboxMessage
            {
                Id = Guid.NewGuid(),
                Type = nameof(OrderCreated),
                Payload = JsonSerializer.Serialize(new OrderCreated(order.Id, cmd.CustomerId, cmd.Amount, DateTime.UtcNow)),
                CreatedAt = DateTime.UtcNow
            });
            
            await _db.SaveChangesAsync();
            await transaction.CommitAsync();
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

---

## 📝 สรุป Part 74

| Broker | Use Case | Strengths |
|--------|----------|-----------|
| RabbitMQ | General messaging, routing | Flexible routing, low latency |
| Azure Service Bus | Azure cloud, enterprise | Managed, sessions, DLQ built-in |
| Apache Kafka | High-throughput streaming | Log retention, replay, partitions |
| MassTransit | Abstraction layer | Works with all brokers |

Best practices:
- **Idempotent consumers**: check message ID before processing
- **Outbox pattern**: never lose published events
- **Dead Letter Queue**: handle failed messages
- **Correlation ID**: trace messages across services
- **Schema evolution**: use forward/backward compatible changes

---

**ก่อนหน้า → [Part 73: Advanced Testing](part73-advanced-testing.md)**  
**ต่อไป → [Part 75: Background Services](part75-background-services.md)**
