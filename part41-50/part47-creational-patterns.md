# Part 47: Design Patterns - Creational
## ขั้นตอนที่ 461-470: Patterns การสร้าง Objects

---

## 🎯 เป้าหมายของ Part นี้
- Singleton
- Factory Method
- Abstract Factory
- Builder
- Prototype
- Object Pool

---

## ขั้นตอนที่ 461: Singleton

```csharp
// Thread-safe Singleton with lazy initialization
public sealed class ConfigurationManager
{
    private static readonly Lazy<ConfigurationManager> _instance = 
        new(() => new ConfigurationManager());
    
    private readonly Dictionary<string, string> _settings;
    
    private ConfigurationManager()
    {
        _settings = new Dictionary<string, string>
        {
            { "AppName", "My Application" },
            { "Version", "1.0.0" },
            { "MaxRetries", "3" }
        };
        LoadFromFile();
    }
    
    public static ConfigurationManager Instance => _instance.Value;
    
    public string Get(string key, string defaultValue = "") 
        => _settings.GetValueOrDefault(key, defaultValue);
    
    public T Get<T>(string key, T defaultValue = default!) where T : IParsable<T>
    {
        if (_settings.TryGetValue(key, out var str) && T.TryParse(str, null, out var val))
            return val;
        return defaultValue;
    }
    
    private void LoadFromFile()
    {
        var path = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "appsettings.json");
        if (!File.Exists(path)) return;
        // Load from JSON file...
    }
}

// Usage
var maxRetries = ConfigurationManager.Instance.Get<int>("MaxRetries", 3);
```

---

## ขั้นตอนที่ 462: Factory Method

```csharp
// Abstract creator
public abstract class NotificationSender
{
    // Factory method - subclass decides concrete object
    protected abstract INotification CreateNotification(string message, string recipient);
    
    // Template method - uses the factory method
    public async Task SendAsync(string message, string recipient)
    {
        var notification = CreateNotification(message, recipient);
        await BeforeSendAsync(notification);
        await notification.SendAsync();
        await AfterSendAsync(notification);
    }
    
    protected virtual Task BeforeSendAsync(INotification notification) => Task.CompletedTask;
    protected virtual Task AfterSendAsync(INotification notification) => Task.CompletedTask;
}

public interface INotification
{
    Task SendAsync();
}

// Concrete creators
public class EmailSender : NotificationSender
{
    protected override INotification CreateNotification(string message, string recipient)
        => new EmailNotification(message, recipient);
    
    protected override async Task AfterSendAsync(INotification n)
        => await LogAsync($"Email sent to {n}");
}

public class SmsSender : NotificationSender
{
    protected override INotification CreateNotification(string message, string recipient)
        => new SmsNotification(message, recipient);
}

// Concrete products
public class EmailNotification : INotification
{
    private readonly string _message, _email;
    public EmailNotification(string msg, string email) { _message = msg; _email = email; }
    public async Task SendAsync() { /* Send email */ await Task.Delay(100); }
}

public class SmsNotification : INotification
{
    private readonly string _message, _phone;
    public SmsNotification(string msg, string phone) { _message = msg; _phone = phone; }
    public async Task SendAsync() { /* Send SMS */ await Task.Delay(50); }
}

// Simpler factory (static factory method pattern)
public static class NotificationFactory
{
    public static INotification Create(string type, string message, string recipient)
        => type.ToLower() switch
        {
            "email" => new EmailNotification(message, recipient),
            "sms" => new SmsNotification(message, recipient),
            _ => throw new ArgumentException($"Unknown type: {type}")
        };
}
```

---

## ขั้นตอนที่ 463: Abstract Factory

```csharp
// Abstract Factory สำหรับ UI themes
public interface IUIFactory
{
    IButton CreateButton(string text);
    ITextBox CreateTextBox(string placeholder);
    IDialog CreateDialog(string title, string message);
}

public interface IButton { void Render(); void SetEnabled(bool enabled); }
public interface ITextBox { string GetText(); void SetText(string text); }
public interface IDialog { void Show(); }

// Light theme factory
public class LightThemeFactory : IUIFactory
{
    public IButton CreateButton(string text) => new LightButton(text);
    public ITextBox CreateTextBox(string placeholder) => new LightTextBox(placeholder);
    public IDialog CreateDialog(string title, string message) => new LightDialog(title, message);
}

// Dark theme factory
public class DarkThemeFactory : IUIFactory
{
    public IButton CreateButton(string text) => new DarkButton(text);
    public ITextBox CreateTextBox(string placeholder) => new DarkTextBox(placeholder);
    public IDialog CreateDialog(string title, string message) => new DarkDialog(title, message);
}

// Client code - doesn't know which theme it's using
public class LoginForm
{
    private readonly IButton _loginBtn;
    private readonly ITextBox _emailInput;
    private readonly ITextBox _passwordInput;
    
    public LoginForm(IUIFactory factory)
    {
        _loginBtn = factory.CreateButton("เข้าสู่ระบบ");
        _emailInput = factory.CreateTextBox("อีเมล");
        _passwordInput = factory.CreateTextBox("รหัสผ่าน");
    }
    
    public void Render()
    {
        _emailInput.SetText("");
        _passwordInput.SetText("");
        _loginBtn.SetEnabled(true);
        _loginBtn.Render();
    }
}

// Select factory based on settings
IUIFactory factory = isDarkMode ? new DarkThemeFactory() : new LightThemeFactory();
var form = new LoginForm(factory);
```

---

## ขั้นตอนที่ 464: Builder Pattern

```csharp
// Builder สำหรับ complex object (Email)
public class Email
{
    public string From { get; private set; } = "";
    public List<string> To { get; private set; } = new();
    public List<string> Cc { get; private set; } = new();
    public string Subject { get; private set; } = "";
    public string Body { get; private set; } = "";
    public bool IsHtml { get; private set; }
    public List<string> Attachments { get; private set; } = new();
    public string? ReplyTo { get; private set; }
    public DateTime? ScheduledAt { get; private set; }
    
    private Email() { }
    
    public class Builder
    {
        private readonly Email _email = new();
        
        public Builder From(string from) { _email.From = from; return this; }
        public Builder To(params string[] recipients) { _email.To.AddRange(recipients); return this; }
        public Builder Cc(params string[] recipients) { _email.Cc.AddRange(recipients); return this; }
        public Builder Subject(string subject) { _email.Subject = subject; return this; }
        public Builder Body(string body, bool isHtml = false) { _email.Body = body; _email.IsHtml = isHtml; return this; }
        public Builder Attach(string filePath) { _email.Attachments.Add(filePath); return this; }
        public Builder ReplyTo(string email) { _email.ReplyTo = email; return this; }
        public Builder ScheduleAt(DateTime dt) { _email.ScheduledAt = dt; return this; }
        
        public Email Build()
        {
            if (string.IsNullOrEmpty(_email.From)) throw new InvalidOperationException("From is required");
            if (!_email.To.Any()) throw new InvalidOperationException("At least one recipient required");
            if (string.IsNullOrEmpty(_email.Subject)) throw new InvalidOperationException("Subject is required");
            return _email;
        }
    }
}

// Fluent usage
var email = new Email.Builder()
    .From("noreply@myapp.com")
    .To("customer@example.com", "manager@example.com")
    .Cc("support@myapp.com")
    .Subject("ยืนยันคำสั่งซื้อ #12345")
    .Body("<h1>ขอบคุณสำหรับการสั่งซื้อ</h1>", isHtml: true)
    .Attach("/reports/invoice_12345.pdf")
    .Build();

// Builder for Query - used in ORMs
public class QueryBuilder<T>
{
    private Expression<Func<T, bool>>? _where;
    private Expression<Func<T, object>>? _orderBy;
    private bool _ascending = true;
    private int _skip = 0;
    private int _take = int.MaxValue;
    private List<Expression<Func<T, object>>> _includes = new();
    
    public QueryBuilder<T> Where(Expression<Func<T, bool>> predicate)
    {
        _where = predicate;
        return this;
    }
    
    public QueryBuilder<T> OrderBy(Expression<Func<T, object>> selector, bool ascending = true)
    {
        _orderBy = selector;
        _ascending = ascending;
        return this;
    }
    
    public QueryBuilder<T> Include(Expression<Func<T, object>> navigation)
    {
        _includes.Add(navigation);
        return this;
    }
    
    public QueryBuilder<T> Page(int page, int pageSize)
    {
        _skip = (page - 1) * pageSize;
        _take = pageSize;
        return this;
    }
    
    public IQueryable<T> Build(IQueryable<T> source)
    {
        var q = source;
        if (_where != null) q = q.Where(_where);
        foreach (var inc in _includes) q = q.Include(inc);
        if (_orderBy != null) q = _ascending ? q.OrderBy(_orderBy) : q.OrderByDescending(_orderBy);
        q = q.Skip(_skip).Take(_take);
        return q;
    }
}

// Usage:
var query = new QueryBuilder<Product>()
    .Where(p => p.IsActive && p.Price > 1000)
    .Include(p => p.Category)
    .OrderBy(p => p.Price, ascending: false)
    .Page(page: 2, pageSize: 20)
    .Build(context.Products);
```

---

## ขั้นตอนที่ 465: Prototype & Object Pool

```csharp
// Prototype - clone objects
public interface ICloneable<T>
{
    T Clone();
}

public class OrderTemplate : ICloneable<OrderTemplate>
{
    public string ShippingAddress { get; set; } = "";
    public string BillingAddress { get; set; } = "";
    public PaymentMethod PaymentMethod { get; set; }
    public List<int> DefaultProductIds { get; set; } = new();
    
    public OrderTemplate Clone()
    {
        return new OrderTemplate
        {
            ShippingAddress = ShippingAddress,
            BillingAddress = BillingAddress,
            PaymentMethod = PaymentMethod,
            DefaultProductIds = new List<int>(DefaultProductIds) // deep copy
        };
    }
}

// Object Pool - reuse expensive objects
public class DatabaseConnectionPool
{
    private readonly ConcurrentBag<DatabaseConnection> _pool = new();
    private readonly SemaphoreSlim _semaphore;
    private readonly int _maxConnections;
    private int _currentCount;
    private readonly string _connectionString;
    
    public DatabaseConnectionPool(string connStr, int maxConnections = 10)
    {
        _connectionString = connStr;
        _maxConnections = maxConnections;
        _semaphore = new SemaphoreSlim(maxConnections, maxConnections);
    }
    
    public async Task<PooledConnection> AcquireAsync(CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        
        if (!_pool.TryTake(out var conn))
        {
            conn = new DatabaseConnection(_connectionString);
            await conn.OpenAsync();
            Interlocked.Increment(ref _currentCount);
        }
        
        return new PooledConnection(conn, this);
    }
    
    internal void Release(DatabaseConnection conn)
    {
        _pool.Add(conn);
        _semaphore.Release();
    }
    
    public int Available => _pool.Count;
    public int Total => _currentCount;
}

public class PooledConnection : IAsyncDisposable
{
    private readonly DatabaseConnection _conn;
    private readonly DatabaseConnectionPool _pool;
    
    public PooledConnection(DatabaseConnection conn, DatabaseConnectionPool pool)
    { _conn = conn; _pool = pool; }
    
    public Task<List<T>> QueryAsync<T>(string sql) => _conn.QueryAsync<T>(sql);
    
    public ValueTask DisposeAsync()
    {
        _pool.Release(_conn); // Return to pool, not close!
        return ValueTask.CompletedTask;
    }
}

// Usage
var pool = new DatabaseConnectionPool("Data Source=app.db", maxConnections: 5);

await using var conn = await pool.AcquireAsync();
var results = await conn.QueryAsync<Order>("SELECT * FROM Orders");
// Auto-released back to pool when using block ends
```

---

## ขั้นตอนที่ 466-470: Pattern Comparison

```
Pattern Selection Guide:
┌─────────────────────────────────────────────────────────────┐
│ Problem: Create a single shared instance                     │
│ → Use: Singleton                                            │
├─────────────────────────────────────────────────────────────┤
│ Problem: Create objects without specifying exact class       │
│ → Use: Factory Method                                       │
├─────────────────────────────────────────────────────────────┤
│ Problem: Create families of related objects                  │
│ → Use: Abstract Factory                                     │
├─────────────────────────────────────────────────────────────┤
│ Problem: Complex object with many optional parts             │
│ → Use: Builder                                              │
├─────────────────────────────────────────────────────────────┤
│ Problem: Copy an existing object                            │
│ → Use: Prototype                                            │
├─────────────────────────────────────────────────────────────┤
│ Problem: Expensive object creation, reuse is possible        │
│ → Use: Object Pool                                          │
└─────────────────────────────────────────────────────────────┘
```

```csharp
// Real-world example combining patterns
// Abstract Factory + Builder for Report Generation
public interface IReportFactory
{
    IReportBuilder CreateBuilder();
    IReportRenderer CreateRenderer();
}

public class PdfReportFactory : IReportFactory
{
    public IReportBuilder CreateBuilder() => new PdfReportBuilder();
    public IReportRenderer CreateRenderer() => new PdfRenderer();
}

public class ExcelReportFactory : IReportFactory
{
    public IReportBuilder CreateBuilder() => new ExcelReportBuilder();
    public IReportRenderer CreateRenderer() => new ExcelRenderer();
}

// Usage
IReportFactory factory = format == "pdf" ? new PdfReportFactory() : new ExcelReportFactory();
var report = factory.CreateBuilder()
    .AddTitle("ยอดขายรายเดือน")
    .AddDataTable(salesData)
    .AddChart(chartData)
    .Build();

var renderer = factory.CreateRenderer();
var bytes = await renderer.RenderAsync(report);
await File.WriteAllBytesAsync("report.pdf", bytes);
```

---

## 📝 สรุป Part 47

| Pattern | Use When |
|---------|---------|
| Singleton | Need exactly one instance globally |
| Factory Method | Subclass decides what to create |
| Abstract Factory | Family of related objects |
| Builder | Complex object with many options |
| Prototype | Clone expensive objects |
| Object Pool | Reuse expensive objects |

---

**ก่อนหน้า → [Part 46: SOLID Principles](part46-solid-principles.md)**  
**ต่อไป → [Part 48: Design Patterns - Structural](part48-structural-patterns.md)**
