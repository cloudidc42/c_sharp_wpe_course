# Part 46: SOLID Principles
## ขั้นตอนที่ 451-460: หลักการออกแบบซอฟต์แวร์

---

## 🎯 เป้าหมายของ Part นี้
- S: Single Responsibility Principle
- O: Open/Closed Principle
- L: Liskov Substitution Principle
- I: Interface Segregation Principle
- D: Dependency Inversion Principle
- ตัวอย่างจริงในแต่ละหลักการ
- Refactoring ก่อน/หลัง

---

## ขั้นตอนที่ 451: S - Single Responsibility Principle

```
"A class should have only one reason to change"
คลาสควรมีหน้าที่เดียว มีเหตุผลเดียวที่จะต้องเปลี่ยน
```

```csharp
// ❌ BAD: UserService ทำงานหลายอย่างเกินไป
public class UserService
{
    private List<User> _users = new();
    
    public void Register(string email, string password)
    {
        // 1. Validate
        if (!email.Contains("@")) throw new Exception("Invalid email");
        if (password.Length < 8) throw new Exception("Password too short");
        
        // 2. Hash password
        var hash = Convert.ToBase64String(SHA256.HashData(Encoding.UTF8.GetBytes(password)));
        
        // 3. Save to DB
        var sql = $"INSERT INTO Users VALUES ('{email}', '{hash}')";
        // Execute SQL...
        
        // 4. Send email
        var smtp = new SmtpClient("smtp.gmail.com");
        smtp.Send("noreply@app.com", email, "ยืนยันการสมัคร", "...");
        
        // 5. Log
        File.AppendAllText("log.txt", $"User registered: {email}");
    }
}
```

```csharp
// ✅ GOOD: แยกแต่ละหน้าที่ออกมา
public class PasswordHasher
{
    public string Hash(string password)
        => Convert.ToBase64String(SHA256.HashData(Encoding.UTF8.GetBytes(password)));
    
    public bool Verify(string password, string hash) => Hash(password) == hash;
}

public class UserValidator
{
    public void ValidateRegistration(string email, string password)
    {
        if (!email.Contains('@')) throw new ValidationException("Invalid email");
        if (password.Length < 8) throw new ValidationException("Password too short");
    }
}

public class UserRepository
{
    private readonly DbContext _ctx;
    public UserRepository(DbContext ctx) => _ctx = ctx;
    
    public Task SaveAsync(User user) => /* ... */;
}

public class EmailService
{
    public Task SendWelcomeEmailAsync(string email) => /* ... */;
}

public class UserLogger
{
    private readonly ILogger _logger;
    public void LogRegistration(string email) => _logger.LogInformation("User registered: {Email}", email);
}

// UserService now only orchestrates
public class UserService
{
    private readonly UserValidator _validator;
    private readonly PasswordHasher _hasher;
    private readonly UserRepository _repo;
    private readonly EmailService _email;
    private readonly UserLogger _logger;
    
    public UserService(UserValidator v, PasswordHasher h, UserRepository r, 
                       EmailService e, UserLogger l)
    { _validator = v; _hasher = h; _repo = r; _email = e; _logger = l; }
    
    public async Task RegisterAsync(string email, string password)
    {
        _validator.ValidateRegistration(email, password);
        var user = new User { Email = email, PasswordHash = _hasher.Hash(password) };
        await _repo.SaveAsync(user);
        await _email.SendWelcomeEmailAsync(email);
        _logger.LogRegistration(email);
    }
}
```

---

## ขั้นตอนที่ 452: O - Open/Closed Principle

```
"Open for extension, closed for modification"
เพิ่มฟีเจอร์ใหม่โดยไม่ต้องแก้ code เดิม
```

```csharp
// ❌ BAD: ต้องแก้ Calculate() ทุกครั้งที่เพิ่ม discount type ใหม่
public class DiscountCalculator
{
    public decimal Calculate(Order order, string discountType)
    {
        if (discountType == "Percentage10") return order.Total * 0.9m;
        if (discountType == "Fixed50") return order.Total - 50;
        if (discountType == "VIP") return order.Total * 0.8m;
        // ต้องแก้ไขไฟล์นี้ทุกครั้ง!
        return order.Total;
    }
}
```

```csharp
// ✅ GOOD: Strategy pattern - เพิ่ม discount type ใหม่โดยไม่แก้ code เดิม
public interface IDiscountStrategy
{
    decimal Apply(decimal price);
    string Description { get; }
}

public class PercentageDiscount : IDiscountStrategy
{
    private readonly decimal _percent;
    public PercentageDiscount(decimal percent) => _percent = percent;
    public decimal Apply(decimal price) => price * (1 - _percent / 100);
    public string Description => $"ลด {_percent}%";
}

public class FixedDiscount : IDiscountStrategy
{
    private readonly decimal _amount;
    public FixedDiscount(decimal amount) => _amount = amount;
    public decimal Apply(decimal price) => Math.Max(0, price - _amount);
    public string Description => $"ลด ฿{_amount:N0}";
}

public class BuyXGetYDiscount : IDiscountStrategy
{
    private readonly int _x, _y;
    public BuyXGetYDiscount(int buyX, int getFreeY) { _x = buyX; _y = getFreeY; }
    public decimal Apply(decimal price) => price; // Complex logic based on quantity
    public string Description => $"ซื้อ {_x} แถม {_y}";
}

// DiscountCalculator ไม่ต้องแก้เมื่อเพิ่ม strategy ใหม่
public class DiscountCalculator
{
    public decimal Calculate(decimal price, IDiscountStrategy strategy)
        => strategy.Apply(price);
    
    public decimal CalculateAll(decimal price, IEnumerable<IDiscountStrategy> strategies)
        => strategies.Aggregate(price, (p, s) => s.Apply(p));
}

// Adding new discount type = new class only, no changes to existing code
public class SeasonalDiscount : IDiscountStrategy
{
    private readonly Dictionary<int, decimal> _monthDiscounts = new()
    {
        { 1, 20 }, { 7, 15 }, { 12, 25 } // January, July, December
    };
    
    public decimal Apply(decimal price)
    {
        if (_monthDiscounts.TryGetValue(DateTime.Now.Month, out var pct))
            return price * (1 - pct / 100);
        return price;
    }
    
    public string Description => "Seasonal sale";
}
```

---

## ขั้นตอนที่ 453: L - Liskov Substitution Principle

```
"Subclass must be substitutable for its superclass"
subclass ต้องทำงานแทน base class ได้โดยไม่เกิดปัญหา
```

```csharp
// ❌ BAD: Rectangle/Square problem
public class Rectangle
{
    public virtual double Width { get; set; }
    public virtual double Height { get; set; }
    public double Area => Width * Height;
}

public class Square : Rectangle
{
    public override double Width
    {
        get => base.Width;
        set { base.Width = value; base.Height = value; } // ✗ violates LSP!
    }
    public override double Height
    {
        get => base.Height;
        set { base.Height = value; base.Width = value; } // ✗ violates LSP!
    }
}

// This test fails for Square!
public void TestArea(Rectangle r)
{
    r.Width = 4; r.Height = 5;
    Debug.Assert(r.Area == 20); // ✗ Square gives 25!
}
```

```csharp
// ✅ GOOD: Shape hierarchy that respects LSP
public abstract class Shape
{
    public abstract double Area { get; }
    public abstract double Perimeter { get; }
    public virtual bool Contains(double x, double y) => false;
}

public class Rectangle : Shape
{
    public double Width { get; }
    public double Height { get; }
    public Rectangle(double w, double h) { Width = w; Height = h; }
    public override double Area => Width * Height;
    public override double Perimeter => 2 * (Width + Height);
}

public class Square : Shape
{
    public double Side { get; }
    public Square(double s) => Side = s;
    public override double Area => Side * Side;
    public override double Perimeter => 4 * Side;
}

// Both work correctly as Shape
public double TotalArea(IEnumerable<Shape> shapes)
    => shapes.Sum(s => s.Area); // Works for any Shape

// Another LSP violation to avoid: exceptions in overrides
public abstract class Storage
{
    public abstract void Save(string data);
    public abstract string Load();
    public virtual bool CanSave => true; // Pre-condition
}

// ✗ BadReadOnlyStorage: throws on Save() - violates base class contract
public class BadReadOnlyStorage : Storage
{
    public override void Save(string data) => throw new NotSupportedException();
    public override string Load() => "data";
}

// ✓ Better: don't extend Storage; use separate interface
public interface IReadable { string Load(); }
public interface IWritable { void Save(string data); }
public class ReadOnlyStorage : IReadable { public string Load() => "data"; }
```

---

## ขั้นตอนที่ 454: I - Interface Segregation Principle

```
"Clients should not be forced to depend on interfaces they don't use"
Interface ควรเล็ก เฉพาะเจาะจง ไม่บังคับ implement methods ที่ไม่ต้องการ
```

```csharp
// ❌ BAD: Fat interface
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
    void GetPaid();
    void CallInSick();
}

// Robot doesn't eat/sleep/get sick, forced to throw NotImplementedException
public class Robot : IWorker
{
    public void Work() => Console.WriteLine("bzz...");
    public void Eat() => throw new NotImplementedException();
    public void Sleep() => throw new NotImplementedException();
    public void GetPaid() => throw new NotImplementedException();
    public void CallInSick() => throw new NotImplementedException();
}
```

```csharp
// ✅ GOOD: Small, focused interfaces
public interface IWorkable { void Work(); }
public interface IFeedable { void Eat(); }
public interface ISleepable { void Sleep(); }
public interface IPayable { void GetPaid(decimal amount); }
public interface ISickLeave { void CallInSick(string reason); }

// Human implements what's relevant
public class HumanEmployee : IWorkable, IFeedable, ISleepable, IPayable, ISickLeave
{
    public void Work() => Console.WriteLine("working...");
    public void Eat() => Console.WriteLine("eating...");
    public void Sleep() => Console.WriteLine("sleeping...");
    public void GetPaid(decimal amount) => Console.WriteLine($"💰 {amount:N0}");
    public void CallInSick(string reason) => Console.WriteLine($"sick: {reason}");
}

// Robot only implements what it can do
public class Robot : IWorkable
{
    public void Work() => Console.WriteLine("bzz...");
}

// Another example: Repository interfaces
// ❌ BAD: forces readonly repos to implement write methods
public interface IRepository<T>
{
    T GetById(int id);
    List<T> GetAll();
    void Add(T entity);
    void Update(T entity);
    void Delete(int id);
}

// ✅ GOOD: Split by capability
public interface IReadRepository<T>
{
    Task<T?> GetByIdAsync(int id);
    Task<List<T>> GetAllAsync();
    Task<bool> ExistsAsync(int id);
}

public interface IWriteRepository<T>
{
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
}

public interface IRepository<T> : IReadRepository<T>, IWriteRepository<T> { }

// Report service only needs reads
public class ReportService
{
    private readonly IReadRepository<Order> _orders;  // only reads!
    public ReportService(IReadRepository<Order> orders) => _orders = orders;
}
```

---

## ขั้นตอนที่ 455: D - Dependency Inversion Principle

```
"Depend on abstractions, not concretions"
High-level modules ไม่ควร depend on Low-level modules
ทั้งคู่ควร depend on Abstractions (interfaces)
```

```csharp
// ❌ BAD: High-level OrderService depends on concrete SqlServer
public class OrderService
{
    private readonly SqlServerRepository _repo; // concrete!
    private readonly SmtpEmailSender _email;    // concrete!
    
    public OrderService()
    {
        _repo = new SqlServerRepository("connection_string");  // tightly coupled
        _email = new SmtpEmailSender("smtp.gmail.com");
    }
    
    public void PlaceOrder(Order order)
    {
        _repo.Save(order);
        _email.Send(order.CustomerEmail, "Your order is confirmed");
    }
}
```

```csharp
// ✅ GOOD: Depend on abstractions
public interface IOrderRepository
{
    Task SaveAsync(Order order);
    Task<Order?> GetByIdAsync(int id);
}

public interface INotificationService
{
    Task SendOrderConfirmationAsync(string email, int orderId);
}

// High-level module: depends on interfaces
public class OrderService
{
    private readonly IOrderRepository _repo;
    private readonly INotificationService _notification;
    private readonly ILogger<OrderService> _logger;
    
    // Dependencies injected (not created here)
    public OrderService(
        IOrderRepository repo,
        INotificationService notification,
        ILogger<OrderService> logger)
    {
        _repo = repo;
        _notification = notification;
        _logger = logger;
    }
    
    public async Task PlaceOrderAsync(Order order)
    {
        await _repo.SaveAsync(order);
        await _notification.SendOrderConfirmationAsync(order.CustomerEmail, order.Id);
        _logger.LogInformation("Order {OrderId} placed", order.Id);
    }
}

// Low-level modules: implement the abstractions
public class EfOrderRepository : IOrderRepository
{
    private readonly AppDbContext _ctx;
    public EfOrderRepository(AppDbContext ctx) => _ctx = ctx;
    public async Task SaveAsync(Order order) { _ctx.Orders.Add(order); await _ctx.SaveChangesAsync(); }
    public async Task<Order?> GetByIdAsync(int id) => await _ctx.Orders.FindAsync(id);
}

public class EmailNotificationService : INotificationService
{
    private readonly IEmailClient _client;
    public EmailNotificationService(IEmailClient client) => _client = client;
    public async Task SendOrderConfirmationAsync(string email, int orderId)
        => await _client.SendAsync(email, $"Order #{orderId} confirmed!");
}

// Testing: easy to swap with mocks
public class InMemoryOrderRepository : IOrderRepository
{
    private readonly Dictionary<int, Order> _store = new();
    private int _nextId = 1;
    
    public Task SaveAsync(Order order) { order.Id = _nextId++; _store[order.Id] = order; return Task.CompletedTask; }
    public Task<Order?> GetByIdAsync(int id) => Task.FromResult(_store.GetValueOrDefault(id));
}

// DI configuration
services.AddScoped<IOrderRepository, EfOrderRepository>();
services.AddScoped<INotificationService, EmailNotificationService>();
services.AddScoped<OrderService>();
```

---

## ขั้นตอนที่ 456-460: Putting It All Together

```csharp
// SOLID-compliant Order Processing System
// S: Each class has one responsibility
// O: New payment methods don't require changing existing code
// L: All payment methods work wherever IPaymentProcessor is expected  
// I: PaymentProcessor interface is focused
// D: OrderService depends on abstractions

public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessAsync(PaymentRequest request);
    string ProviderName { get; }
}

public class StripePaymentProcessor : IPaymentProcessor
{
    public string ProviderName => "Stripe";
    public async Task<PaymentResult> ProcessAsync(PaymentRequest request)
    {
        // Stripe API call
        await Task.Delay(100);
        return new PaymentResult { Success = true, TransactionId = $"STR_{Guid.NewGuid():N}" };
    }
}

public class PayPalPaymentProcessor : IPaymentProcessor
{
    public string ProviderName => "PayPal";
    public async Task<PaymentResult> ProcessAsync(PaymentRequest request)
    {
        await Task.Delay(100);
        return new PaymentResult { Success = true, TransactionId = $"PP_{Guid.NewGuid():N}" };
    }
}

public class PaymentService
{
    private readonly IEnumerable<IPaymentProcessor> _processors;
    
    public PaymentService(IEnumerable<IPaymentProcessor> processors) => _processors = processors;
    
    public async Task<PaymentResult> ProcessAsync(PaymentRequest request, string provider)
    {
        var processor = _processors.FirstOrDefault(p => 
            p.ProviderName.Equals(provider, StringComparison.OrdinalIgnoreCase))
            ?? throw new ArgumentException($"Provider '{provider}' not supported");
        
        return await processor.ProcessAsync(request);
    }
}

// Register all payment processors
services.AddScoped<IPaymentProcessor, StripePaymentProcessor>();
services.AddScoped<IPaymentProcessor, PayPalPaymentProcessor>();
// Adding QR Pay = just add QRPayProcessor class + register it. Zero changes to existing!
```

---

## 📝 สรุป Part 46

| Principle | หลักการ | ประโยชน์ |
|-----------|---------|---------|
| S - SRP | หนึ่งคลาส หนึ่งหน้าที่ | แก้ไขง่าย ทดสอบง่าย |
| O - OCP | เปิดสำหรับ extend ปิดสำหรับ modify | เพิ่มฟีเจอร์ไม่ต้องแก้ code เดิม |
| L - LSP | Subclass ทำงานแทน base ได้ | Polymorphism ที่ถูกต้อง |
| I - ISP | Interface เล็กเฉพาะเจาะจง | ไม่ต้อง implement method ที่ไม่ต้องการ |
| D - DIP | Depend บน Abstraction | Testable, swappable dependencies |

---

**ก่อนหน้า → [Part 45: EF Core + WPF](part45-ef-wpf-integration.md)**  
**ต่อไป → [Part 47: Design Patterns - Creational](part47-creational-patterns.md)**
