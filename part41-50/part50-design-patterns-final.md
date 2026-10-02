# Part 50: Design Patterns - Final Project
## ขั้นตอนที่ 491-500: E-Commerce Order System ด้วย Design Patterns

---

## 🎯 เป้าหมายของ Part นี้
รวม Design Patterns ทั้งหมดในโปรเจกต์จริง:
- Creational: Singleton, Factory, Builder
- Structural: Adapter, Decorator, Facade
- Behavioral: Observer, Strategy, Command, State

---

## ขั้นตอนที่ 491: Project Structure

```
OrderSystem/
├── Core/
│   ├── Entities/        # Domain objects
│   ├── Interfaces/      # Abstractions
│   └── Events/          # Domain events
├── Infrastructure/
│   ├── Adapters/        # External service adapters
│   ├── Repositories/    # Data access
│   └── Services/        # Infrastructure services  
├── Application/
│   ├── Commands/        # Command pattern (CQRS)
│   ├── Handlers/        # Command handlers
│   └── Services/        # Application services
└── Program.cs
```

---

## ขั้นตอนที่ 492: Domain Entities & State Pattern

```csharp
// Core/Entities/Order.cs - State Pattern
public enum OrderStatus { Pending, Confirmed, Processing, Shipped, Delivered, Cancelled }

public class Order
{
    public int Id { get; private set; }
    public string CustomerId { get; private set; }
    public List<OrderLine> Lines { get; private set; } = new();
    public OrderStatus Status { get; private set; } = OrderStatus.Pending;
    public DateTime CreatedAt { get; private set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; private set; }
    
    private readonly List<IDomainEvent> _events = new();
    public IReadOnlyList<IDomainEvent> Events => _events.AsReadOnly();
    
    private Order() { CustomerId = ""; }
    
    public static Order Create(string customerId, IEnumerable<(int ProductId, int Qty, decimal Price)> items)
    {
        var order = new Order { CustomerId = customerId };
        foreach (var (pid, qty, price) in items)
            order.Lines.Add(new OrderLine(pid, qty, price));
        order._events.Add(new OrderCreatedEvent(order));
        return order;
    }
    
    public void Confirm()
    {
        EnsureStatus(OrderStatus.Pending);
        Status = OrderStatus.Confirmed;
        _events.Add(new OrderStatusChangedEvent(Id, OrderStatus.Confirmed));
        Touch();
    }
    
    public void StartProcessing()
    {
        EnsureStatus(OrderStatus.Confirmed);
        Status = OrderStatus.Processing;
        Touch();
    }
    
    public void Ship(string trackingNumber)
    {
        EnsureStatus(OrderStatus.Processing);
        Status = OrderStatus.Shipped;
        TrackingNumber = trackingNumber;
        _events.Add(new OrderShippedEvent(Id, trackingNumber));
        Touch();
    }
    
    public void Deliver()
    {
        EnsureStatus(OrderStatus.Shipped);
        Status = OrderStatus.Delivered;
        Touch();
    }
    
    public void Cancel(string reason)
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Delivered)
            throw new InvalidOperationException("ไม่สามารถยกเลิกคำสั่งซื้อที่จัดส่งแล้ว");
        Status = OrderStatus.Cancelled;
        CancelReason = reason;
        _events.Add(new OrderCancelledEvent(Id, reason));
        Touch();
    }
    
    public decimal TotalAmount => Lines.Sum(l => l.UnitPrice * l.Quantity);
    public string? TrackingNumber { get; private set; }
    public string? CancelReason { get; private set; }
    
    private void EnsureStatus(OrderStatus expected)
    {
        if (Status != expected)
            throw new InvalidOperationException($"Order ต้องอยู่ในสถานะ {expected} แต่ปัจจุบันเป็น {Status}");
    }
    
    private void Touch() => UpdatedAt = DateTime.UtcNow;
    
    public void ClearEvents() => _events.Clear();
}

public record OrderLine(int ProductId, int Quantity, decimal UnitPrice);

// Domain Events - Observer Pattern foundation
public interface IDomainEvent { DateTime OccurredAt { get; } }
public record OrderCreatedEvent(Order Order) : IDomainEvent { public DateTime OccurredAt { get; } = DateTime.UtcNow; }
public record OrderStatusChangedEvent(int OrderId, OrderStatus NewStatus) : IDomainEvent { public DateTime OccurredAt { get; } = DateTime.UtcNow; }
public record OrderShippedEvent(int OrderId, string TrackingNumber) : IDomainEvent { public DateTime OccurredAt { get; } = DateTime.UtcNow; }
public record OrderCancelledEvent(int OrderId, string Reason) : IDomainEvent { public DateTime OccurredAt { get; } = DateTime.UtcNow; }
```

---

## ขั้นตอนที่ 493: Strategy Pattern - Payment & Discount

```csharp
// Strategy: Payment
public interface IPaymentStrategy
{
    string Name { get; }
    Task<PaymentResult> ProcessAsync(decimal amount, PaymentDetails details);
}

public class CreditCardStrategy : IPaymentStrategy
{
    public string Name => "Credit Card";
    public async Task<PaymentResult> ProcessAsync(decimal amount, PaymentDetails details)
    {
        // Process credit card payment
        await Task.Delay(200); // simulate API call
        return PaymentResult.Success($"CC-{Guid.NewGuid():N}");
    }
}

public class PromptPayStrategy : IPaymentStrategy
{
    public string Name => "PromptPay";
    public async Task<PaymentResult> ProcessAsync(decimal amount, PaymentDetails details)
    {
        await Task.Delay(100);
        return PaymentResult.Success($"PP-{Guid.NewGuid():N}");
    }
}

// Strategy: Discount
public interface IDiscountStrategy
{
    decimal Calculate(Order order, string? couponCode = null);
}

public class MemberDiscountStrategy : IDiscountStrategy
{
    private readonly IMemberRepository _members;
    public MemberDiscountStrategy(IMemberRepository members) => _members = members;
    
    public decimal Calculate(Order order, string? couponCode = null)
    {
        var member = _members.GetByCustomerId(order.CustomerId);
        return member?.Tier switch
        {
            MemberTier.Gold => order.TotalAmount * 0.10m,
            MemberTier.Platinum => order.TotalAmount * 0.15m,
            _ => 0
        };
    }
}

public class CouponDiscountStrategy : IDiscountStrategy
{
    private readonly ICouponRepository _coupons;
    public CouponDiscountStrategy(ICouponRepository coupons) => _coupons = coupons;
    
    public decimal Calculate(Order order, string? couponCode = null)
    {
        if (string.IsNullOrEmpty(couponCode)) return 0;
        var coupon = _coupons.GetByCode(couponCode);
        if (coupon == null || coupon.IsExpired) return 0;
        return coupon.IsPercentage ? order.TotalAmount * coupon.Value / 100 : coupon.Value;
    }
}

public class CompositeDiscountStrategy : IDiscountStrategy
{
    private readonly IEnumerable<IDiscountStrategy> _strategies;
    public CompositeDiscountStrategy(IEnumerable<IDiscountStrategy> strategies)
        => _strategies = strategies;
    
    public decimal Calculate(Order order, string? couponCode = null)
        => _strategies.Sum(s => s.Calculate(order, couponCode));
}
```

---

## ขั้นตอนที่ 494: Adapter Pattern - External Services

```csharp
// Adapter: Wrap 3rd party SMS provider
public interface ISmsService
{
    Task<bool> SendAsync(string phone, string message);
}

// External library (can't modify)
public class TwilioClient
{
    public void SendSms(string to, string from, string body) { /* Twilio API */ }
}

// Adapter
public class TwilioSmsAdapter : ISmsService
{
    private readonly TwilioClient _twilio;
    private readonly string _fromNumber;
    
    public TwilioSmsAdapter(TwilioClient twilio, string fromNumber)
    { _twilio = twilio; _fromNumber = fromNumber; }
    
    public Task<bool> SendAsync(string phone, string message)
    {
        try
        {
            _twilio.SendSms(phone, _fromNumber, message);
            return Task.FromResult(true);
        }
        catch { return Task.FromResult(false); }
    }
}

// Adapter: Legacy payment gateway
public class LegacyPaymentSystem
{
    public string DoPayment(string cardNum, string exp, int amount) 
        => amount > 0 ? "SUCCESS" : "FAILED";
}

public class LegacyPaymentAdapter : IPaymentStrategy
{
    private readonly LegacyPaymentSystem _legacy;
    public string Name => "Legacy Card";
    
    public LegacyPaymentAdapter(LegacyPaymentSystem legacy) => _legacy = legacy;
    
    public Task<PaymentResult> ProcessAsync(decimal amount, PaymentDetails details)
    {
        var result = _legacy.DoPayment(details.CardNumber!, details.Expiry!, (int)(amount * 100));
        return Task.FromResult(result == "SUCCESS" 
            ? PaymentResult.Success($"LEG-{DateTime.Now.Ticks}")
            : PaymentResult.Fail("Payment declined"));
    }
}
```

---

## ขั้นตอนที่ 495: Decorator Pattern - Resilient Services

```csharp
// Decorator: Retry + Circuit Breaker for HTTP calls
public interface IHttpService
{
    Task<T> GetAsync<T>(string url);
    Task<TResponse> PostAsync<TRequest, TResponse>(string url, TRequest body);
}

// Base implementation
public class HttpService : IHttpService
{
    private readonly HttpClient _http;
    public HttpService(HttpClient http) => _http = http;
    
    public async Task<T> GetAsync<T>(string url)
    {
        var response = await _http.GetAsync(url);
        response.EnsureSuccessStatusCode();
        return (await response.Content.ReadFromJsonAsync<T>())!;
    }
    
    public async Task<TResponse> PostAsync<TRequest, TResponse>(string url, TRequest body)
    {
        var response = await _http.PostAsJsonAsync(url, body);
        response.EnsureSuccessStatusCode();
        return (await response.Content.ReadFromJsonAsync<TResponse>())!;
    }
}

// Decorator: Retry
public class RetryHttpService : IHttpService
{
    private readonly IHttpService _inner;
    private readonly int _maxRetries;
    
    public RetryHttpService(IHttpService inner, int maxRetries = 3)
    { _inner = inner; _maxRetries = maxRetries; }
    
    public async Task<T> GetAsync<T>(string url)
    {
        for (int i = 0; i <= _maxRetries; i++)
        {
            try { return await _inner.GetAsync<T>(url); }
            catch (HttpRequestException) when (i < _maxRetries)
            { await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, i))); }
        }
        throw new Exception("Max retries exceeded");
    }
    
    public async Task<TRes> PostAsync<TReq, TRes>(string url, TReq body)
    {
        for (int i = 0; i <= _maxRetries; i++)
        {
            try { return await _inner.PostAsync<TReq, TRes>(url, body); }
            catch when (i < _maxRetries)
            { await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, i))); }
        }
        throw new Exception("Max retries exceeded");
    }
}

// Decorator: Logging
public class LoggingHttpService : IHttpService
{
    private readonly IHttpService _inner;
    private readonly ILogger _logger;
    
    public LoggingHttpService(IHttpService inner, ILogger logger)
    { _inner = inner; _logger = logger; }
    
    public async Task<T> GetAsync<T>(string url)
    {
        _logger.LogInformation("GET {Url}", url);
        var sw = Stopwatch.StartNew();
        try
        {
            var result = await _inner.GetAsync<T>(url);
            _logger.LogInformation("GET {Url} completed in {Ms}ms", url, sw.ElapsedMilliseconds);
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "GET {Url} failed after {Ms}ms", url, sw.ElapsedMilliseconds);
            throw;
        }
    }
    
    public async Task<TRes> PostAsync<TReq, TRes>(string url, TReq body)
    {
        _logger.LogInformation("POST {Url}", url);
        return await _inner.PostAsync<TReq, TRes>(url, body);
    }
}

// Build decorated chain
IHttpService httpService = new LoggingHttpService(
    new RetryHttpService(
        new HttpService(new HttpClient()),
        maxRetries: 3
    ),
    logger
);
```

---

## ขั้นตอนที่ 496: Facade + Observer

```csharp
// Facade: OrderFacade - ซ่อน complexity
public class OrderFacade
{
    private readonly IOrderRepository _orders;
    private readonly IPaymentStrategy _payment;
    private readonly IDiscountStrategy _discount;
    private readonly ISmsService _sms;
    private readonly IEventPublisher _events;
    
    public OrderFacade(
        IOrderRepository orders,
        IPaymentStrategy payment,
        IDiscountStrategy discount,
        ISmsService sms,
        IEventPublisher events)
    {
        _orders = orders; _payment = payment; _discount = discount;
        _sms = sms; _events = events;
    }
    
    public async Task<OrderResult> PlaceOrderAsync(PlaceOrderRequest request)
    {
        // 1. Create order
        var order = Order.Create(request.CustomerId, request.Items);
        
        // 2. Apply discounts
        var discount = _discount.Calculate(order, request.CouponCode);
        var finalAmount = order.TotalAmount - discount;
        
        // 3. Process payment
        var payment = await _payment.ProcessAsync(finalAmount, request.PaymentDetails);
        if (!payment.IsSuccess)
            return OrderResult.Fail($"ชำระเงินไม่สำเร็จ: {payment.Message}");
        
        // 4. Confirm order
        order.Confirm();
        await _orders.SaveAsync(order);
        
        // 5. Publish events (Observer)
        foreach (var evt in order.Events)
            await _events.PublishAsync(evt);
        order.ClearEvents();
        
        // 6. Notify customer
        await _sms.SendAsync(request.CustomerPhone, 
            $"คำสั่งซื้อ #{order.Id} ได้รับการยืนยัน ยอดชำระ ฿{finalAmount:N0}");
        
        return OrderResult.Success(order, payment.TransactionId!);
    }
}

// Observer: Event Publisher/Subscriber
public interface IEventHandler<T> where T : IDomainEvent
{
    Task HandleAsync(T evt);
}

public class EventPublisher : IEventPublisher
{
    private readonly IServiceProvider _services;
    public EventPublisher(IServiceProvider services) => _services = services;
    
    public async Task PublishAsync<T>(T evt) where T : IDomainEvent
    {
        var handlers = _services.GetServices<IEventHandler<T>>();
        foreach (var handler in handlers)
            await handler.HandleAsync(evt);
    }
}

// Event handlers
public class SendEmailOnOrderCreated : IEventHandler<OrderCreatedEvent>
{
    private readonly IEmailService _email;
    public SendEmailOnOrderCreated(IEmailService email) => _email = email;
    
    public Task HandleAsync(OrderCreatedEvent evt)
        => _email.SendAsync(new Email.Builder()
            .From("noreply@shop.com")
            .To(evt.Order.CustomerId)
            .Subject($"ยืนยันคำสั่งซื้อ #{evt.Order.Id}")
            .Body($"<h1>ขอบคุณสำหรับการสั่งซื้อ ฿{evt.Order.TotalAmount:N0}</h1>", isHtml: true)
            .Build());
}

public class UpdateStockOnOrderConfirmed : IEventHandler<OrderStatusChangedEvent>
{
    private readonly IInventoryService _inventory;
    public UpdateStockOnOrderConfirmed(IInventoryService inventory) => _inventory = inventory;
    
    public async Task HandleAsync(OrderStatusChangedEvent evt)
    {
        if (evt.NewStatus == OrderStatus.Confirmed)
            await _inventory.ReserveStockAsync(evt.OrderId);
    }
}
```

---

## ขั้นตอนที่ 497: Command Pattern - Undo/Redo

```csharp
// Command: Order management commands with undo support
public interface IOrderCommand
{
    Task ExecuteAsync(Order order);
    Task UndoAsync(Order order);
    string Description { get; }
}

public class ApplyDiscountCommand : IOrderCommand
{
    private readonly decimal _discount;
    private decimal _previousDiscount;
    
    public ApplyDiscountCommand(decimal discount) => _discount = discount;
    public string Description => $"ส่วนลด ฿{_discount:N0}";
    
    public Task ExecuteAsync(Order order)
    {
        _previousDiscount = order.AppliedDiscount;
        order.ApplyDiscount(_discount);
        return Task.CompletedTask;
    }
    
    public Task UndoAsync(Order order)
    {
        order.ApplyDiscount(_previousDiscount);
        return Task.CompletedTask;
    }
}

public class OrderCommandManager
{
    private readonly Stack<IOrderCommand> _history = new();
    private readonly Stack<IOrderCommand> _redoStack = new();
    
    public async Task ExecuteAsync(IOrderCommand command, Order order)
    {
        await command.ExecuteAsync(order);
        _history.Push(command);
        _redoStack.Clear(); // Clear redo stack on new command
    }
    
    public async Task UndoAsync(Order order)
    {
        if (!_history.Any()) return;
        var command = _history.Pop();
        await command.UndoAsync(order);
        _redoStack.Push(command);
    }
    
    public async Task RedoAsync(Order order)
    {
        if (!_redoStack.Any()) return;
        var command = _redoStack.Pop();
        await command.ExecuteAsync(order);
        _history.Push(command);
    }
    
    public IReadOnlyList<string> History => _history.Select(c => c.Description).ToList();
}
```

---

## ขั้นตอนที่ 498: Builder + Factory

```csharp
// Factory: Create appropriate payment strategy
public class PaymentStrategyFactory
{
    private readonly Dictionary<string, Func<IPaymentStrategy>> _registry = new();
    
    public PaymentStrategyFactory Register(string name, Func<IPaymentStrategy> factory)
    {
        _registry[name] = factory;
        return this;
    }
    
    public IPaymentStrategy Create(string name)
    {
        if (_registry.TryGetValue(name, out var factory))
            return factory();
        throw new ArgumentException($"Unknown payment method: {name}");
    }
}

// Builder: Build order request
public class PlaceOrderRequestBuilder
{
    private string _customerId = "";
    private string _customerPhone = "";
    private readonly List<(int ProductId, int Qty, decimal Price)> _items = new();
    private PaymentDetails _payment = new();
    private string? _coupon;
    
    public PlaceOrderRequestBuilder ForCustomer(string id, string phone)
    {
        _customerId = id; _customerPhone = phone; return this;
    }
    
    public PlaceOrderRequestBuilder AddItem(int productId, int qty, decimal price)
    {
        _items.Add((productId, qty, price)); return this;
    }
    
    public PlaceOrderRequestBuilder WithCoupon(string code) { _coupon = code; return this; }
    
    public PlaceOrderRequestBuilder PayWith(PaymentDetails payment)
    {
        _payment = payment; return this;
    }
    
    public PlaceOrderRequest Build()
    {
        if (string.IsNullOrEmpty(_customerId)) throw new InvalidOperationException("Customer required");
        if (!_items.Any()) throw new InvalidOperationException("At least one item required");
        return new PlaceOrderRequest
        {
            CustomerId = _customerId,
            CustomerPhone = _customerPhone,
            Items = _items,
            PaymentDetails = _payment,
            CouponCode = _coupon
        };
    }
}
```

---

## ขั้นตอนที่ 499: DI Setup & App Bootstrap

```csharp
// Program.cs - Wire everything together with DI
var services = new ServiceCollection();

// Infrastructure
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddScoped<IMemberRepository, MemberRepository>();
services.AddScoped<ICouponRepository, CouponRepository>();

// Adapters
services.AddSingleton<TwilioClient>();
services.AddScoped<ISmsService>(sp => 
    new TwilioSmsAdapter(sp.GetRequiredService<TwilioClient>(), "+66800000000"));

// Payment - Factory pattern
services.AddSingleton(sp => new PaymentStrategyFactory()
    .Register("credit_card", () => new CreditCardStrategy())
    .Register("promptpay", () => new PromptPayStrategy())
    .Register("legacy", () => new LegacyPaymentAdapter(new LegacyPaymentSystem())));

// Discount - Composite Strategy
services.AddScoped<IDiscountStrategy>(sp => new CompositeDiscountStrategy(new IDiscountStrategy[]
{
    new MemberDiscountStrategy(sp.GetRequiredService<IMemberRepository>()),
    new CouponDiscountStrategy(sp.GetRequiredService<ICouponRepository>()),
}));

// Events - Observer
services.AddScoped<IEventPublisher, EventPublisher>();
services.AddScoped<IEventHandler<OrderCreatedEvent>, SendEmailOnOrderCreated>();
services.AddScoped<IEventHandler<OrderStatusChangedEvent>, UpdateStockOnOrderConfirmed>();

// Facade
services.AddScoped<OrderFacade>();

var provider = services.BuildServiceProvider();

// Usage
using var scope = provider.CreateScope();
var facade = scope.ServiceProvider.GetRequiredService<OrderFacade>();
var payFactory = scope.ServiceProvider.GetRequiredService<PaymentStrategyFactory>();

var request = new PlaceOrderRequestBuilder()
    .ForCustomer("CUST001", "0812345678")
    .AddItem(productId: 1, qty: 2, price: 15000)
    .AddItem(productId: 5, qty: 1, price: 8000)
    .WithCoupon("SAVE10")
    .PayWith(new PaymentDetails { Method = "credit_card", CardNumber = "4111111111111111" })
    .Build();

var result = await facade.PlaceOrderAsync(request);
Console.WriteLine(result.IsSuccess 
    ? $"✅ คำสั่งซื้อ #{result.Order!.Id} สำเร็จ (TXN: {result.TransactionId})"
    : $"❌ ผิดพลาด: {result.ErrorMessage}");
```

---

## ขั้นตอนที่ 500: สรุปหลักสูตร Design Patterns

```
Design Patterns ที่เรียนมา:
┌───────────────┬────────────────────────────────────────────────┐
│ Category      │ Patterns                                        │
├───────────────┼────────────────────────────────────────────────┤
│ Creational    │ Singleton, Factory Method, Abstract Factory,    │
│               │ Builder, Prototype, Object Pool                 │
├───────────────┼────────────────────────────────────────────────┤
│ Structural    │ Adapter, Decorator, Facade, Proxy,              │
│               │ Composite, Bridge                               │
├───────────────┼────────────────────────────────────────────────┤
│ Behavioral    │ Observer, Strategy, Command, Iterator,          │
│               │ State, Template Method, Chain of Responsibility │
└───────────────┴────────────────────────────────────────────────┘
```

## 📝 สรุป Part 41-50

| Part | หัวข้อ | Steps |
|------|--------|-------|
| 41 | EF Core Basics | 401-410 |
| 42 | EF Core Migrations | 411-420 |
| 43 | Unit of Work | 421-430 |
| 44 | LINQ Advanced | 431-440 |
| 45 | EF Core + WPF | 441-450 |
| 46 | SOLID Principles | 451-460 |
| 47 | Creational Patterns | 461-470 |
| 48 | Structural Patterns | 471-480 |
| 49 | Behavioral Patterns | 481-490 |
| 50 | Design Patterns Final | 491-500 |

---

**ก่อนหน้า → [Part 49: Behavioral Patterns](part49-behavioral-patterns.md)**  
**ต่อไป → [Part 51: Clean Architecture](../part51-60/part51-clean-architecture.md)**
