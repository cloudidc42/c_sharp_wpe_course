# Part 92: Best Practices & Anti-Patterns
## ขั้นตอนที่ 911-920: แนวทางปฏิบัติที่ดี

> **หลักสูตร C# จากผู้เริ่มต้นสู่มืออาชีพ** | ส่วนที่ 92 จาก 100

---

## บทนำ

ในส่วนนี้เราจะเรียนรู้เกี่ยวกับแนวทางปฏิบัติที่ดี (Best Practices) และรูปแบบที่ควรหลีกเลี่ยง (Anti-Patterns) ในการพัฒนา C# การเข้าใจสิ่งเหล่านี้จะช่วยให้โค้ดของคุณมีคุณภาพสูง บำรุงรักษาได้ง่าย และมีประสิทธิภาพดี

---

## ขั้นตอนที่ 911: Anti-Patterns ที่ควรหลีกเลี่ยงใน C#

### ปัญหา Mutable Statics (Static ที่เปลี่ยนค่าได้)

**อธิบาย:** Mutable static fields เป็นหนึ่งในปัญหาที่พบบ่อยที่สุด เพราะทำให้โค้ดทดสอบได้ยากและเกิด race condition ใน multi-threaded environment

```csharp
// ❌ Anti-Pattern: Mutable Static
public class BadCounter
{
    // ปัญหา: shared state ระหว่าง instances และ threads
    public static int Count = 0;
    public static string LastMessage = string.Empty;

    public void Increment(string message)
    {
        Count++;
        LastMessage = message; // race condition ใน concurrent code
    }
}

// ✅ Best Practice: Instance-based State
public class GoodCounter
{
    private int _count = 0;
    private string _lastMessage = string.Empty;
    private readonly object _lock = new object();

    public int Count => _count;
    public string LastMessage => _lastMessage;

    public void Increment(string message)
    {
        lock (_lock)
        {
            _count++;
            _lastMessage = message;
        }
    }
}

// ✅ หรือใช้ Interlocked สำหรับ atomic operations
public class ThreadSafeCounter
{
    private int _count = 0;

    public int Count => _count;

    public void Increment()
    {
        Interlocked.Increment(ref _count);
    }
}
```

### ปัญหา Service Locator

**อธิบาย:** Service Locator pattern ซ่อน dependencies ทำให้ไม่รู้ว่า class ต้องการอะไร และยากต่อการ mock ใน unit tests

```csharp
// ❌ Anti-Pattern: Service Locator
public class ServiceLocator
{
    private static readonly Dictionary<Type, object> _services 
        = new Dictionary<Type, object>();

    public static void Register<T>(T service) where T : class
    {
        _services[typeof(T)] = service;
    }

    public static T Resolve<T>() where T : class
    {
        if (_services.TryGetValue(typeof(T), out var service))
            return (T)service;
        throw new InvalidOperationException($"Service {typeof(T).Name} not found");
    }
}

// การใช้งาน Service Locator (ปัญหา)
public class OrderServiceBad
{
    public void ProcessOrder(Order order)
    {
        // ❌ Dependencies ซ่อนอยู่ภายใน method
        var repo = ServiceLocator.Resolve<IOrderRepository>();
        var emailService = ServiceLocator.Resolve<IEmailService>();
        var logger = ServiceLocator.Resolve<ILogger>();

        repo.Save(order);
        emailService.SendConfirmation(order);
        logger.Log("Order processed");
    }
}

// ✅ Best Practice: Constructor Injection
public class OrderServiceGood
{
    private readonly IOrderRepository _repository;
    private readonly IEmailService _emailService;
    private readonly ILogger<OrderServiceGood> _logger;

    // Dependencies ชัดเจน ทดสอบง่าย
    public OrderServiceGood(
        IOrderRepository repository,
        IEmailService emailService,
        ILogger<OrderServiceGood> logger)
    {
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
        _emailService = emailService ?? throw new ArgumentNullException(nameof(emailService));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    public async Task ProcessOrderAsync(Order order)
    {
        await _repository.SaveAsync(order);
        await _emailService.SendConfirmationAsync(order);
        _logger.LogInformation("Order {OrderId} processed successfully", order.Id);
    }
}
```

### ปัญหา God Class

**อธิบาย:** God Class คือ class ที่รู้ทุกอย่างและทำทุกอย่าง ละเมิดหลัก Single Responsibility Principle

```csharp
// ❌ Anti-Pattern: God Class
public class ApplicationManager
{
    // class นี้จัดการทุกอย่าง!
    public void CreateUser(string name, string email) { /* ... */ }
    public void DeleteUser(int userId) { /* ... */ }
    public void SendEmail(string to, string subject) { /* ... */ }
    public void GenerateReport() { /* ... */ }
    public void ProcessPayment(decimal amount) { /* ... */ }
    public void UpdateInventory(int productId, int quantity) { /* ... */ }
    public void LogActivity(string message) { /* ... */ }
    public void ValidateInput(string input) { /* ... */ }
    public void ConnectToDatabase() { /* ... */ }
    public void CacheData(string key, object value) { /* ... */ }
    // ... อีก 50 methods!
}

// ✅ Best Practice: แยก responsibilities ออกเป็น classes ที่เฉพาะเจาะจง
public class UserService
{
    private readonly IUserRepository _repository;

    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }

    public async Task<User> CreateUserAsync(CreateUserRequest request)
    {
        var user = new User(request.Name, request.Email);
        return await _repository.CreateAsync(user);
    }

    public async Task DeleteUserAsync(int userId)
    {
        await _repository.DeleteAsync(userId);
    }
}

public class EmailService
{
    private readonly IEmailProvider _provider;

    public EmailService(IEmailProvider provider)
    {
        _provider = provider;
    }

    public async Task SendAsync(EmailMessage message)
    {
        await _provider.SendAsync(message);
    }
}

public class PaymentService
{
    private readonly IPaymentGateway _gateway;

    public PaymentService(IPaymentGateway gateway)
    {
        _gateway = gateway;
    }

    public async Task<PaymentResult> ProcessAsync(PaymentRequest request)
    {
        return await _gateway.ChargeAsync(request);
    }
}
```

### ปัญหา Magic Strings และ Magic Numbers

```csharp
// ❌ Anti-Pattern: Magic Strings และ Magic Numbers
public class BadOrderProcessor
{
    public bool CanDiscount(string customerType, decimal orderAmount)
    {
        // ❌ Magic strings และ numbers - ไม่รู้ความหมาย
        if (customerType == "premium" && orderAmount > 1000)
        {
            return true;
        }

        if (customerType == "vip" && orderAmount > 500)
        {
            return true;
        }

        return false;
    }

    public decimal CalculateDiscount(string customerType, decimal amount)
    {
        if (customerType == "premium")
            return amount * 0.15m; // 15% แต่ไม่ชัดเจน
        if (customerType == "vip")
            return amount * 0.25m; // 25%
        return 0;
    }
}

// ✅ Best Practice: ใช้ constants, enums, และ named values
public enum CustomerType
{
    Regular,
    Premium,
    Vip
}

public static class DiscountPolicy
{
    public const decimal PremiumMinOrderAmount = 1000m;
    public const decimal VipMinOrderAmount = 500m;
    public const decimal PremiumDiscountRate = 0.15m;
    public const decimal VipDiscountRate = 0.25m;
}

public class GoodOrderProcessor
{
    public bool CanDiscount(CustomerType customerType, decimal orderAmount)
    {
        return customerType switch
        {
            CustomerType.Premium => orderAmount > DiscountPolicy.PremiumMinOrderAmount,
            CustomerType.Vip => orderAmount > DiscountPolicy.VipMinOrderAmount,
            _ => false
        };
    }

    public decimal CalculateDiscount(CustomerType customerType, decimal amount)
    {
        return customerType switch
        {
            CustomerType.Premium => amount * DiscountPolicy.PremiumDiscountRate,
            CustomerType.Vip => amount * DiscountPolicy.VipDiscountRate,
            _ => 0m
        };
    }
}
```

---

## ขั้นตอนที่ 912: Naming Conventions และการจัดระเบียบโค้ด

### การตั้งชื่อที่ดี

```csharp
// ❌ ชื่อที่ไม่ดี
public class Mgr
{
    private int n;
    private List<object> lst;
    private bool flg;

    public void DoStuff(int x, string s)
    {
        for (int i = 0; i < n; i++)
        {
            // ทำอะไรสักอย่าง
        }
    }

    public object Get(int id) => null;
}

// ✅ ชื่อที่ดี
public class OrderManager
{
    private int _maximumOrdersPerDay;
    private List<Order> _pendingOrders;
    private bool _isProcessingEnabled;

    // ชื่อ method บอกว่าทำอะไร
    public async Task ProcessPendingOrdersAsync(
        int batchSize, 
        CancellationToken cancellationToken)
    {
        for (int orderIndex = 0; orderIndex < _pendingOrders.Count; orderIndex++)
        {
            // process each order
        }
    }

    // ชื่อ return type ชัดเจน
    public async Task<Order?> FindOrderByIdAsync(int orderId) 
        => await Task.FromResult<Order?>(null);
}
```

### C# Naming Conventions มาตรฐาน

```csharp
// ✅ C# Naming Conventions ที่ถูกต้อง

// Namespaces: PascalCase
namespace MyCompany.OrderManagement.Services
{
    // Classes, Structs, Enums: PascalCase
    public class CustomerOrderService
    {
        // Constants: PascalCase (ไม่ใช่ SCREAMING_SNAKE_CASE)
        public const int MaxRetryAttempts = 3;
        public const string DefaultCurrency = "THB";

        // Private fields: _camelCase (underscore prefix)
        private readonly IOrderRepository _orderRepository;
        private int _currentRetryCount;

        // Properties: PascalCase
        public string ServiceName { get; private set; } = "OrderService";
        public bool IsInitialized { get; private set; }

        // Public methods: PascalCase
        public async Task<OrderResult> CreateOrderAsync(CreateOrderRequest request)
        {
            // Local variables: camelCase
            var validationResult = ValidateRequest(request);
            int retryCount = 0;
            bool isSuccessful = false;

            return new OrderResult();
        }

        // Private methods: PascalCase (ไม่ใช่ _camelCase)
        private ValidationResult ValidateRequest(CreateOrderRequest request)
        {
            return new ValidationResult();
        }

        // Interfaces: IPascalCase (I prefix)
        // public interface IOrderService { }

        // Generic type parameters: T หรือ TPascalCase
        public T ConvertTo<TResult>(object source) where TResult : class
        {
            return default!;
        }
    }

    // Enums: PascalCase, values: PascalCase
    public enum OrderStatus
    {
        Pending,
        Processing,
        Completed,
        Cancelled,
        Failed
    }

    // Records: PascalCase
    public record CreateOrderRequest(
        int CustomerId,
        List<OrderItem> Items,
        string Currency = "THB"
    );

    public record OrderResult(
        bool IsSuccess = false,
        string? ErrorMessage = null,
        Order? Order = null
    );
}
```

### การจัดระเบียบ Class

```csharp
// ✅ การจัดลำดับ members ใน class ที่ดี
public class WellOrganizedClass
{
    // 1. Constants
    private const int DefaultTimeout = 30;
    public const string Version = "1.0";

    // 2. Static fields
    private static readonly object _syncRoot = new object();

    // 3. Instance fields (private อยู่บน)
    private readonly ILogger<WellOrganizedClass> _logger;
    private readonly IConfiguration _configuration;
    private int _instanceCount;

    // 4. Constructors
    public WellOrganizedClass(
        ILogger<WellOrganizedClass> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }

    // 5. Properties (public ก่อน, private หลัง)
    public string Name { get; set; } = string.Empty;
    public bool IsActive { get; private set; }
    private int InternalState { get; set; }

    // 6. Public methods
    public async Task InitializeAsync()
    {
        IsActive = true;
        _logger.LogInformation("Initialized {ClassName}", nameof(WellOrganizedClass));
    }

    public async Task<Result> ProcessAsync(Request request)
    {
        ValidateRequest(request);
        return await ExecuteProcessAsync(request);
    }

    // 7. Protected methods
    protected virtual void OnProcessCompleted(Result result)
    {
        // hook สำหรับ subclasses
    }

    // 8. Private methods
    private void ValidateRequest(Request request)
    {
        ArgumentNullException.ThrowIfNull(request);
    }

    private async Task<Result> ExecuteProcessAsync(Request request)
    {
        return await Task.FromResult(new Result());
    }
}

// Placeholder types
public class Request { }
public class Result { }
```

---

## ขั้นตอนที่ 913: Dependency Injection Best Practices

### หลีกเลี่ยง Service Locator ใน DI Container

```csharp
// ❌ Anti-Pattern: ใช้ IServiceProvider โดยตรง (Service Locator)
public class BadService
{
    private readonly IServiceProvider _serviceProvider;

    public BadService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public async Task DoWorkAsync()
    {
        // ❌ ซ่อน dependencies, testability แย่
        var repo = _serviceProvider.GetRequiredService<IUserRepository>();
        var cache = _serviceProvider.GetRequiredService<ICache>();
        var logger = _serviceProvider.GetRequiredService<ILogger<BadService>>();

        var users = await repo.GetAllAsync();
    }
}

// ✅ Best Practice: Constructor Injection อย่างชัดเจน
public class GoodService
{
    private readonly IUserRepository _userRepository;
    private readonly ICache _cache;
    private readonly ILogger<GoodService> _logger;

    public GoodService(
        IUserRepository userRepository,
        ICache cache,
        ILogger<GoodService> logger)
    {
        _userRepository = userRepository;
        _cache = cache;
        _logger = logger;
    }

    public async Task DoWorkAsync()
    {
        var users = await _userRepository.GetAllAsync();
        _logger.LogInformation("Retrieved {Count} users", users.Count());
    }
}
```

### Scope Mistakes - ระวังการ Inject Service ผิด Lifetime

```csharp
// Program.cs / Startup.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddApplicationServices(
        this IServiceCollection services)
    {
        // ❌ ปัญหา Captive Dependency:
        // Singleton ไม่ควร depend on Scoped service!
        // services.AddSingleton<IBadSingletonService, BadSingletonService>();
        // services.AddScoped<IScopedService, ScopedService>();

        // ✅ Lifetime ที่ถูกต้อง:

        // Singleton: สร้างครั้งเดียว, ใช้ตลอด application lifetime
        // เหมาะสำหรับ: configuration, caches, stateless services
        services.AddSingleton<IConfigurationService, ConfigurationService>();
        services.AddSingleton<IMemoryCache, MemoryCache>();

        // Scoped: สร้างใหม่ต่อ HTTP request
        // เหมาะสำหรับ: DbContext, per-request state
        services.AddScoped<IOrderService, OrderService>();
        services.AddScoped<IUserContext, UserContext>();

        // Transient: สร้างใหม่ทุกครั้งที่ inject
        // เหมาะสำหรับ: lightweight, stateless operations
        services.AddTransient<IEmailBuilder, EmailBuilder>();
        services.AddTransient<IValidator<Order>, OrderValidator>();

        return services;
    }
}

// ✅ ถ้าต้องการ Singleton ที่ใช้ Scoped service ให้ใช้ Factory
public class SingletonServiceWithScopedDependency
{
    private readonly IServiceScopeFactory _scopeFactory;

    public SingletonServiceWithScopedDependency(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task DoWorkAsync()
    {
        // สร้าง scope ใหม่เพื่อใช้ scoped service
        using var scope = _scopeFactory.CreateScope();
        var scopedService = scope.ServiceProvider.GetRequiredService<IScopedService>();
        await scopedService.ProcessAsync();
    }
}
```

### การ Register Services แบบ Options Pattern

```csharp
// Models/Settings
public class DatabaseSettings
{
    public const string SectionName = "Database";

    public string ConnectionString { get; set; } = string.Empty;
    public int CommandTimeout { get; set; } = 30;
    public int MaxRetryCount { get; set; } = 3;
    public bool EnableSensitiveDataLogging { get; set; } = false;
}

public class EmailSettings
{
    public const string SectionName = "Email";

    public string SmtpHost { get; set; } = string.Empty;
    public int SmtpPort { get; set; } = 587;
    public string Username { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public bool UseSsl { get; set; } = true;
}

// Registration
public static class ServiceRegistration
{
    public static IServiceCollection AddConfiguredServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // ✅ Options Pattern
        services.Configure<DatabaseSettings>(
            configuration.GetSection(DatabaseSettings.SectionName));
        
        services.Configure<EmailSettings>(
            configuration.GetSection(EmailSettings.SectionName));

        // ✅ Validate options on startup
        services.AddOptions<DatabaseSettings>()
            .BindConfiguration(DatabaseSettings.SectionName)
            .ValidateDataAnnotations()
            .ValidateOnStart();

        return services;
    }
}

// ✅ ใช้ IOptions<T> ใน service
public class DatabaseService
{
    private readonly DatabaseSettings _settings;

    public DatabaseService(IOptions<DatabaseSettings> options)
    {
        _settings = options.Value;
    }

    public async Task<IDbConnection> CreateConnectionAsync()
    {
        // ใช้ _settings.ConnectionString
        return await Task.FromResult<IDbConnection>(null!);
    }
}
```

---

## ขั้นตอนที่ 914: Exception Handling Best Practices

### เมื่อไหร่ควร Catch Exception

```csharp
// ❌ Anti-Pattern: Catch ทุก exception โดยไม่จัดการ
public class BadExceptionHandling
{
    public async Task<User?> GetUserAsync(int userId)
    {
        try
        {
            return await _repository.FindByIdAsync(userId);
        }
        catch (Exception ex)
        {
            // ❌ กลืน exception! ไม่รู้ว่าเกิดอะไรขึ้น
            return null;
        }
    }

    public void ProcessData(string data)
    {
        try
        {
            // process data
        }
        catch
        {
            // ❌ catch ทุกอย่างโดยไม่ log
        }
    }
}

// ✅ Best Practice: Catch เฉพาะ exceptions ที่รู้วิธีจัดการ
public class GoodExceptionHandling
{
    private readonly IUserRepository _repository;
    private readonly ILogger<GoodExceptionHandling> _logger;

    public GoodExceptionHandling(
        IUserRepository repository, 
        ILogger<GoodExceptionHandling> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<User?> GetUserAsync(int userId)
    {
        try
        {
            return await _repository.FindByIdAsync(userId);
        }
        catch (NotFoundException)
        {
            // ✅ Known case - user doesn't exist, return null
            return null;
        }
        // ✅ ไม่ catch exceptions อื่น - ให้ propagate ขึ้นไป
    }

    public async Task<bool> TryUpdateUserAsync(int userId, UpdateUserRequest request)
    {
        try
        {
            await _repository.UpdateAsync(userId, request);
            return true;
        }
        catch (ConcurrencyException ex)
        {
            // ✅ Catch specific exception, log it, return false
            _logger.LogWarning(ex, "Concurrency conflict updating user {UserId}", userId);
            return false;
        }
        catch (ValidationException ex)
        {
            // ✅ Validation error - expected, handle gracefully
            _logger.LogInformation(
                "Validation failed for user {UserId}: {Error}", 
                userId, ex.Message);
            throw; // re-throw ให้ caller จัดการ
        }
    }
}
```

### Custom Exceptions

```csharp
// ✅ Custom Exception hierarchy
public abstract class DomainException : Exception
{
    public string ErrorCode { get; }

    protected DomainException(string errorCode, string message) 
        : base(message)
    {
        ErrorCode = errorCode;
    }

    protected DomainException(string errorCode, string message, Exception innerException) 
        : base(message, innerException)
    {
        ErrorCode = errorCode;
    }
}

public class NotFoundException : DomainException
{
    public NotFoundException(string entityName, object key)
        : base("NOT_FOUND", $"{entityName} with key '{key}' was not found.")
    {
    }
}

public class ValidationException : DomainException
{
    public IReadOnlyList<string> Errors { get; }

    public ValidationException(IEnumerable<string> errors)
        : base("VALIDATION_FAILED", "One or more validation errors occurred.")
    {
        Errors = errors.ToList();
    }
}

public class BusinessRuleException : DomainException
{
    public BusinessRuleException(string ruleName, string message)
        : base($"BUSINESS_RULE_{ruleName.ToUpper()}", message)
    {
    }
}

public class UnauthorizedException : DomainException
{
    public UnauthorizedException(string resource, string action)
        : base("UNAUTHORIZED", 
            $"You are not authorized to {action} on {resource}.")
    {
    }
}

// ✅ Global Exception Handler (ASP.NET Core)
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var (statusCode, message) = exception switch
        {
            NotFoundException ex => (StatusCodes.Status404NotFound, ex.Message),
            ValidationException ex => (StatusCodes.Status400BadRequest, ex.Message),
            UnauthorizedException ex => (StatusCodes.Status403Forbidden, ex.Message),
            BusinessRuleException ex => (StatusCodes.Status422UnprocessableEntity, ex.Message),
            _ => (StatusCodes.Status500InternalServerError, "An unexpected error occurred.")
        };

        if (statusCode >= 500)
        {
            _logger.LogError(exception, "Unhandled exception occurred");
        }
        else
        {
            _logger.LogWarning(exception, "Business exception: {Message}", exception.Message);
        }

        httpContext.Response.StatusCode = statusCode;
        await httpContext.Response.WriteAsJsonAsync(
            new { error = message, code = (exception as DomainException)?.ErrorCode },
            cancellationToken);

        return true;
    }
}
```

### Re-throw Patterns

```csharp
public class ExceptionRethrowExamples
{
    private readonly ILogger<ExceptionRethrowExamples> _logger;

    public ExceptionRethrowExamples(ILogger<ExceptionRethrowExamples> logger)
    {
        _logger = logger;
    }

    // ✅ ถูกต้อง: ใช้ throw; (ไม่ใช่ throw ex;) เพื่อรักษา stack trace
    public async Task GoodRethrowAsync()
    {
        try
        {
            await DoOperationAsync();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Operation failed");
            throw; // ✅ รักษา original stack trace
        }
    }

    // ❌ ผิด: throw ex; ทำลาย stack trace
    public async Task BadRethrowAsync()
    {
        try
        {
            await DoOperationAsync();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Operation failed");
            throw ex; // ❌ reset stack trace!
        }
    }

    // ✅ Wrap exception เมื่อต้องการเพิ่ม context
    public async Task WrapExceptionAsync()
    {
        try
        {
            await DoOperationAsync();
        }
        catch (DatabaseException ex)
        {
            // ✅ Wrap เพื่อเพิ่ม context แต่รักษา inner exception
            throw new DataAccessException(
                "Failed to save order to database", ex);
        }
    }

    private async Task DoOperationAsync() 
        => await Task.CompletedTask;
}

public class DatabaseException : Exception
{
    public DatabaseException(string message) : base(message) { }
}

public class DataAccessException : Exception
{
    public DataAccessException(string message, Exception innerException) 
        : base(message, innerException) { }
}
```

---

## ขั้นตอนที่ 915: Async/Await Anti-Patterns

### async void - หลีกเลี่ยง!

```csharp
// ❌ Anti-Pattern: async void
public class AsyncVoidProblems
{
    // ❌ async void: exceptions ไม่ถูก catch, ไม่รอได้
    public async void BadFireAndForget()
    {
        await Task.Delay(1000);
        throw new Exception("This exception will crash the app!"); // ❌ unhandled!
    }

    // ❌ Event handler exception ที่อาจ crash app
    public async void Button_Click(object sender, EventArgs e)
    {
        // หาก throw exception ที่นี่ app จะ crash โดยไม่มีการ catch
        await DoSomethingAsync();
    }

    // ✅ ใช้ async Task แทน (ยกเว้น event handlers)
    public async Task GoodMethodAsync()
    {
        await Task.Delay(1000);
        // exceptions จะ propagate ถูกต้อง
    }

    // ✅ Event handler: ต้องเป็น async void แต่ handle exceptions ด้วยตนเอง
    public async void Button_ClickSafe(object sender, EventArgs e)
    {
        try
        {
            await DoSomethingAsync();
        }
        catch (Exception ex)
        {
            // ✅ Handle exception explicitly
            LogError(ex);
            ShowErrorToUser("An error occurred");
        }
    }

    // ✅ Fire-and-forget อย่างปลอดภัย
    public void SafeFireAndForget(Func<Task> taskFactory, ILogger logger)
    {
        _ = Task.Run(async () =>
        {
            try
            {
                await taskFactory();
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Background task failed");
            }
        });
    }

    private async Task DoSomethingAsync() => await Task.CompletedTask;
    private void LogError(Exception ex) { }
    private void ShowErrorToUser(string message) { }
}
```

### .Result และ .Wait() - Deadlock!

```csharp
// ❌ Anti-Pattern: .Result และ .Wait() อาจทำให้เกิด Deadlock
public class DeadlockExample
{
    public string GetDataBad()
    {
        // ❌ อาจ deadlock ใน ASP.NET หรือ UI thread!
        return GetDataAsync().Result;
    }

    public void ProcessBad()
    {
        // ❌ อาจ deadlock!
        DoWorkAsync().Wait();
    }

    // ✅ ใช้ async/await ตลอดทั้ง call chain
    public async Task<string> GetDataAsync()
    {
        await Task.Delay(100);
        return "data";
    }

    // ✅ ถ้า ต้องเรียกจาก sync code (rare case)
    public string GetDataSafeSync()
    {
        // ใช้ Task.Run เพื่อหลีกเลี่ยง deadlock (แต่ระวังใช้ resources เพิ่ม)
        return Task.Run(() => GetDataAsync()).GetAwaiter().GetResult();
    }
}
```

### Cancellation Token

```csharp
// ❌ ไม่ใช้ CancellationToken
public class WithoutCancellation
{
    public async Task<List<Order>> GetOrdersAsync()
    {
        // ❌ ถ้า user cancel request นี้ยังทำงานต่อไปจนเสร็จ
        await Task.Delay(5000); // long operation
        return new List<Order>();
    }
}

// ✅ ใช้ CancellationToken อย่างถูกต้อง
public class WithCancellation
{
    private readonly IOrderRepository _repository;
    private readonly ILogger<WithCancellation> _logger;

    public WithCancellation(
        IOrderRepository repository, 
        ILogger<WithCancellation> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<List<Order>> GetOrdersAsync(
        CancellationToken cancellationToken = default)
    {
        // ✅ Pass cancellationToken ไปทุก async call
        var orders = await _repository.GetAllAsync(cancellationToken);
        
        // ✅ Check cancellation ใน loops ที่ทำงานนาน
        var results = new List<Order>();
        foreach (var order in orders)
        {
            cancellationToken.ThrowIfCancellationRequested();
            results.Add(order);
        }

        return results;
    }

    // ✅ สร้าง CancellationTokenSource พร้อม timeout
    public async Task<string> GetWithTimeoutAsync()
    {
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

        try
        {
            return await LongRunningOperationAsync(cts.Token);
        }
        catch (OperationCanceledException) when (cts.IsCancellationRequested)
        {
            _logger.LogWarning("Operation timed out after 10 seconds");
            throw new TimeoutException("Operation took too long");
        }
    }

    // ✅ Linked CancellationToken
    public async Task ProcessWithLinkedCancellationAsync(
        CancellationToken requestToken)
    {
        using var timeoutCts = new CancellationTokenSource(TimeSpan.FromMinutes(2));
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            requestToken, timeoutCts.Token);

        await LongRunningOperationAsync(linkedCts.Token);
    }

    private async Task<string> LongRunningOperationAsync(
        CancellationToken cancellationToken)
    {
        await Task.Delay(5000, cancellationToken);
        return "result";
    }
}
```

### ConfigureAwait

```csharp
// ✅ Library code ควรใช้ ConfigureAwait(false)
public class LibraryService
{
    // Library code: ใช้ ConfigureAwait(false) เพื่อป้องกัน deadlock
    // และ performance ดีขึ้น (ไม่ต้อง resume บน original context)
    public async Task<string> GetDataAsync()
    {
        var result = await FetchFromApiAsync().ConfigureAwait(false);
        var processed = await ProcessDataAsync(result).ConfigureAwait(false);
        return processed;
    }

    private async Task<string> FetchFromApiAsync()
    {
        await Task.Delay(100).ConfigureAwait(false);
        return "raw data";
    }

    private async Task<string> ProcessDataAsync(string data)
    {
        await Task.Delay(50).ConfigureAwait(false);
        return data.ToUpper();
    }
}

// ✅ Application code (ASP.NET Core) มักไม่ต้องใช้ ConfigureAwait(false)
// เพราะ ASP.NET Core ไม่มี SynchronizationContext
public class ApplicationController
{
    public async Task<IActionResult> GetAsync()
    {
        // ใน ASP.NET Core สามารถไม่ใช้ ConfigureAwait(false) ได้
        var data = await GetDataFromServiceAsync();
        return new OkObjectResult(data);
    }

    private async Task<string> GetDataFromServiceAsync()
        => await Task.FromResult("data");
}

public interface IActionResult { }
public class OkObjectResult : IActionResult
{
    public OkObjectResult(object value) { }
}
```

---

## ขั้นตอนที่ 916: LINQ Best Practices

### หลีกเลี่ยง N+1 Problem

```csharp
// ❌ Anti-Pattern: N+1 Query Problem
public class BadLinqUsage
{
    private readonly AppDbContext _context;

    public BadLinqUsage(AppDbContext context)
    {
        _context = context;
    }

    // ❌ N+1: 1 query สำหรับ orders + N queries สำหรับแต่ละ customer
    public async Task<List<OrderSummary>> GetOrderSummariesBadAsync()
    {
        var orders = await _context.Orders.ToListAsync();
        
        return orders.Select(order => new OrderSummary
        {
            OrderId = order.Id,
            // ❌ แต่ละ order ยิง query ใหม่!
            CustomerName = _context.Customers
                .First(c => c.Id == order.CustomerId).Name,
            Total = order.Total
        }).ToList();
    }
}

// ✅ Best Practice: Eager Loading และ Projections
public class GoodLinqUsage
{
    private readonly AppDbContext _context;
    private readonly ILogger<GoodLinqUsage> _logger;

    public GoodLinqUsage(AppDbContext context, ILogger<GoodLinqUsage> logger)
    {
        _context = context;
        _logger = logger;
    }

    // ✅ ใช้ Include() สำหรับ related data
    public async Task<List<OrderSummary>> GetOrderSummariesWithIncludeAsync()
    {
        return await _context.Orders
            .Include(o => o.Customer)  // ✅ Eager load
            .Select(o => new OrderSummary
            {
                OrderId = o.Id,
                CustomerName = o.Customer.Name,
                Total = o.Total
            })
            .ToListAsync();
    }

    // ✅ Projection โดยตรง (ดีที่สุดสำหรับ performance)
    public async Task<List<OrderSummary>> GetOrderSummariesWithProjectionAsync(
        CancellationToken cancellationToken = default)
    {
        return await _context.Orders
            .Join(
                _context.Customers,
                order => order.CustomerId,
                customer => customer.Id,
                (order, customer) => new OrderSummary
                {
                    OrderId = order.Id,
                    CustomerName = customer.Name,
                    Total = order.Total
                }
            )
            .ToListAsync(cancellationToken);
    }

    // ✅ ใช้ Where ก่อน ToList เสมอ (filter ที่ database)
    public async Task<List<Order>> GetPendingOrdersAsync(
        DateTime fromDate,
        CancellationToken cancellationToken = default)
    {
        // ✅ Filter ที่ database, ไม่ใช่ in-memory
        return await _context.Orders
            .Where(o => o.Status == OrderStatus.Pending)
            .Where(o => o.CreatedAt >= fromDate)
            .OrderBy(o => o.CreatedAt)
            .Take(100) // ✅ เสมอ limit results
            .ToListAsync(cancellationToken);
    }
}
```

### Deferred Execution Pitfalls

```csharp
// ✅ เข้าใจ Deferred Execution
public class DeferredExecutionExamples
{
    private List<int> _numbers = new() { 1, 2, 3, 4, 5 };

    public void ShowDeferredExecution()
    {
        // ❌ ปัญหา: query ถูก execute หลายครั้ง
        IEnumerable<int> query = _numbers.Where(n => n % 2 == 0);
        
        // query execute ครั้งที่ 1
        Console.WriteLine($"Count: {query.Count()}");
        
        // เพิ่มข้อมูลระหว่างกลาง
        _numbers.Add(6);
        
        // query execute ครั้งที่ 2 - ได้ผลลัพธ์ต่างกัน!
        Console.WriteLine($"Count again: {query.Count()}");
        
        // ✅ Materialize เมื่อต้องการ snapshot
        List<int> materializedQuery = _numbers
            .Where(n => n % 2 == 0)
            .ToList(); // execute และเก็บผล

        _numbers.Add(8); // ไม่กระทบ materializedQuery แล้ว
        Console.WriteLine($"Stable count: {materializedQuery.Count}");
    }

    // ✅ ระวัง closure ใน LINQ
    public IEnumerable<Func<int>> ClosurePitfall()
    {
        var funcs = new List<Func<int>>();
        
        // ❌ ทุก lambda จะ capture variable i เดียวกัน
        for (int i = 0; i < 5; i++)
        {
            funcs.Add(() => i); // ❌ capture by reference!
        }
        // funcs.Select(f => f()) จะให้ [5, 5, 5, 5, 5] ไม่ใช่ [0, 1, 2, 3, 4]

        // ✅ Copy value ก่อน capture
        funcs.Clear();
        for (int i = 0; i < 5; i++)
        {
            int copy = i; // ✅ copy value
            funcs.Add(() => copy);
        }
        // ตอนนี้ได้ [0, 1, 2, 3, 4]
        
        return funcs;
    }

    // ✅ LINQ Best Practices
    public void LinqBestPractices()
    {
        var data = Enumerable.Range(1, 1000).ToList();

        // ✅ Chain operations ที่ IEnumerable (lazy evaluation)
        var result = data
            .Where(x => x % 2 == 0)    // filter ก่อน
            .Select(x => x * x)          // transform
            .Take(10)                    // limit
            .ToList();                   // materialize เมื่อพร้อม

        // ✅ ใช้ Any() แทน Count() > 0 (เร็วกว่า)
        bool hasEvens = data.Any(x => x % 2 == 0);    // ✅
        bool hasEvensOld = data.Count(x => x % 2 == 0) > 0; // ❌ slow

        // ✅ ใช้ FirstOrDefault() แทน Where().First()
        int? first = data.FirstOrDefault(x => x > 500); // ✅
        // int firstBad = data.Where(x => x > 500).First(); // ❌ less clear
    }
}
```

---

## ขั้นตอนที่ 917: Memory Management Best Practices

### IDisposable และ Using Statements

```csharp
// ✅ Implement IDisposable อย่างถูกต้อง
public class ResourceManager : IDisposable
{
    private readonly FileStream _fileStream;
    private readonly SqlConnection _dbConnection;
    private bool _disposed = false;

    public ResourceManager(string filePath, string connectionString)
    {
        _fileStream = new FileStream(filePath, FileMode.OpenOrCreate);
        _dbConnection = new SqlConnection(connectionString);
    }

    public async Task DoWorkAsync()
    {
        if (_disposed) throw new ObjectDisposedException(nameof(ResourceManager));
        
        await _dbConnection.OpenAsync();
        // do work...
    }

    // ✅ IDisposable pattern ที่ถูกต้อง
    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                // Managed resources
                _fileStream?.Dispose();
                _dbConnection?.Dispose();
            }
            // Unmanaged resources ถ้ามี

            _disposed = true;
        }
    }

    // Finalizer (ถ้ามี unmanaged resources)
    ~ResourceManager()
    {
        Dispose(disposing: false);
    }
}

// ✅ IAsyncDisposable สำหรับ async cleanup
public class AsyncResourceManager : IAsyncDisposable
{
    private readonly HttpClient _httpClient;
    private bool _disposed = false;

    public AsyncResourceManager()
    {
        _httpClient = new HttpClient();
    }

    public async ValueTask DisposeAsync()
    {
        if (!_disposed)
        {
            _httpClient.Dispose();
            _disposed = true;
        }
        
        GC.SuppressFinalize(this);
        await ValueTask.CompletedTask;
    }
}

// ✅ Using statements อย่างถูกต้อง
public class UsingExamples
{
    public async Task ProcessFileAsync(string path)
    {
        // ✅ using statement - dispose เมื่อออกจาก scope
        using var fileStream = new FileStream(path, FileMode.Open);
        using var reader = new StreamReader(fileStream);
        
        string content = await reader.ReadToEndAsync();
        // fileStream และ reader จะถูก dispose อัตโนมัติ

        // ✅ await using สำหรับ IAsyncDisposable
        await using var asyncResource = new AsyncResourceManager();
        // async cleanup เมื่อออกจาก scope
    }

    // ✅ using สำหรับ multiple resources
    public async Task ProcessMultipleResourcesAsync()
    {
        using var connection = new SqlConnection("...");
        using var command = connection.CreateCommand();
        await using var transaction = await connection.BeginTransactionAsync();
        
        try
        {
            command.CommandText = "INSERT INTO Orders VALUES (...)";
            await command.ExecuteNonQueryAsync();
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

### GC Considerations

```csharp
// ✅ Memory-efficient patterns
public class MemoryEfficiency
{
    // ✅ ใช้ Span<T> และ Memory<T> สำหรับ array operations
    public static int SumArray(int[] numbers)
    {
        Span<int> span = numbers;
        int sum = 0;
        foreach (int n in span)
        {
            sum += n;
        }
        return sum;
    }

    // ✅ ใช้ StringBuilder สำหรับ string concatenation
    public static string BuildLargeString(IEnumerable<string> parts)
    {
        var builder = new System.Text.StringBuilder();
        foreach (var part in parts)
        {
            builder.Append(part);
            builder.Append(", ");
        }
        
        if (builder.Length > 2)
            builder.Length -= 2; // ลบ ", " ท้าย
            
        return builder.ToString();
    }

    // ❌ ไม่ดี: string concatenation ใน loop
    public static string BuildLargeStringBad(IEnumerable<string> parts)
    {
        string result = "";
        foreach (var part in parts)
        {
            result += part + ", "; // ❌ สร้าง string ใหม่ทุก iteration!
        }
        return result;
    }

    // ✅ ArrayPool สำหรับ temporary arrays
    public static int ProcessTemporaryData(byte[] input)
    {
        // ยืม buffer จาก pool แทนการสร้างใหม่
        byte[] buffer = System.Buffers.ArrayPool<byte>.Shared.Rent(input.Length);
        
        try
        {
            Array.Copy(input, buffer, input.Length);
            // process buffer...
            return buffer.Length;
        }
        finally
        {
            // คืน buffer กลับ pool เสมอ!
            System.Buffers.ArrayPool<byte>.Shared.Return(buffer);
        }
    }
}
```

---

## ขั้นตอนที่ 918: Logging Best Practices

### Structured Logging

```csharp
// ❌ Anti-Pattern: String Interpolation ใน Log
public class BadLogging
{
    private readonly ILogger<BadLogging> _logger;

    public BadLogging(ILogger<BadLogging> logger)
    {
        _logger = logger;
    }

    public async Task ProcessOrderAsync(int orderId, decimal amount)
    {
        // ❌ string interpolation: ไม่ structured, ไม่ searchable
        _logger.LogInformation($"Processing order {orderId} with amount {amount}");
        
        // ❌ ใช้ LogError สำหรับ non-error
        _logger.LogError($"User {orderId} logged in"); // ❌ wrong log level
        
        await Task.CompletedTask;
    }
}

// ✅ Best Practice: Structured Logging with message templates
public class GoodLogging
{
    private readonly ILogger<GoodLogging> _logger;

    public GoodLogging(ILogger<GoodLogging> logger)
    {
        _logger = logger;
    }

    public async Task ProcessOrderAsync(
        int orderId, 
        decimal amount,
        CancellationToken cancellationToken = default)
    {
        // ✅ Message template: properties ถูก log เป็น structured data
        _logger.LogInformation(
            "Processing order {OrderId} with amount {Amount:C}", 
            orderId, amount);

        var stopwatch = System.Diagnostics.Stopwatch.StartNew();
        
        try
        {
            // ... process order
            await Task.Delay(100, cancellationToken);
            
            stopwatch.Stop();
            _logger.LogInformation(
                "Order {OrderId} processed successfully in {ElapsedMs}ms",
                orderId, stopwatch.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            _logger.LogError(
                ex, 
                "Failed to process order {OrderId} after {ElapsedMs}ms",
                orderId, stopwatch.ElapsedMilliseconds);
            throw;
        }
    }
}
```

### Log Levels - เลือกใช้ให้เหมาะสม

```csharp
public class LogLevelExamples
{
    private readonly ILogger<LogLevelExamples> _logger;

    public LogLevelExamples(ILogger<LogLevelExamples> logger)
    {
        _logger = logger;
    }

    public async Task DemonstrateLogLevelsAsync(
        int orderId, 
        string userId,
        CancellationToken cancellationToken = default)
    {
        // Trace: ข้อมูลละเอียดมากสำหรับ debugging (ปิดใน production)
        _logger.LogTrace(
            "Entering ProcessOrder with OrderId={OrderId}, UserId={UserId}", 
            orderId, userId);

        // Debug: ข้อมูล debug ที่ developer ต้องการ
        _logger.LogDebug(
            "Looking up customer for OrderId={OrderId}", orderId);

        // Information: เหตุการณ์ปกติที่สำคัญ (user login, order created)
        _logger.LogInformation(
            "Processing order {OrderId} for user {UserId}", orderId, userId);

        try
        {
            await ProcessAsync(orderId, cancellationToken);
        }
        catch (ValidationException ex)
        {
            // Warning: สิ่งที่ไม่คาดหวัง แต่ไม่ทำให้ระบบล้มเหลว
            _logger.LogWarning(
                "Validation failed for order {OrderId}: {Errors}", 
                orderId, string.Join(", ", ex.Errors));
        }
        catch (DatabaseException ex)
        {
            // Error: ปัญหาที่ทำให้ operation ล้มเหลว แต่ app ยังทำงานได้
            _logger.LogError(
                ex, 
                "Database error processing order {OrderId}", orderId);
        }
        catch (Exception ex)
        {
            // Critical: ปัญหาร้ายแรงที่อาจทำให้ app หยุดทำงาน
            _logger.LogCritical(
                ex, 
                "Unexpected critical error in order processing {OrderId}", orderId);
            throw;
        }
    }

    private async Task ProcessAsync(int orderId, CancellationToken cancellationToken)
    {
        await Task.Delay(100, cancellationToken);
    }
}
```

### สิ่งที่ควรและไม่ควร Log

```csharp
// ✅ สิ่งที่ควร Log
public class LoggingGuidelines
{
    private readonly ILogger<LoggingGuidelines> _logger;

    public LoggingGuidelines(ILogger<LoggingGuidelines> logger)
    {
        _logger = logger;
    }

    public async Task ProcessPaymentAsync(PaymentRequest request)
    {
        // ✅ Log business events สำคัญ
        _logger.LogInformation(
            "Payment initiated for Order={OrderId}, Amount={Amount:C}, Currency={Currency}",
            request.OrderId, request.Amount, request.Currency);

        // ✅ Log security events
        _logger.LogInformation(
            "User {UserId} initiated payment from IP {ClientIp}",
            request.UserId, request.ClientIp);

        // ❌ ห้าม Log ข้อมูล sensitive!
        // _logger.LogInformation("Card number: {CardNumber}", request.CardNumber); // ❌
        // _logger.LogInformation("Password: {Password}", user.Password); // ❌
        // _logger.LogInformation("SSN: {SSN}", customer.SSN); // ❌

        // ✅ Log แค่ข้อมูลที่จำเป็น (masked)
        string maskedCard = $"****-****-****-{request.CardLastFour}";
        _logger.LogInformation(
            "Payment processed with card {MaskedCard}", maskedCard);

        await Task.CompletedTask;
    }

    // ✅ Performance logging
    public async Task<List<Product>> GetProductsWithTimingAsync(
        string category,
        CancellationToken cancellationToken = default)
    {
        using var activity = System.Diagnostics.Activity.Current;
        var sw = System.Diagnostics.Stopwatch.StartNew();
        
        try
        {
            var products = new List<Product>(); // mock result
            sw.Stop();

            // ✅ Log slow queries
            if (sw.ElapsedMilliseconds > 1000)
            {
                _logger.LogWarning(
                    "Slow query: GetProducts for Category={Category} took {ElapsedMs}ms",
                    category, sw.ElapsedMilliseconds);
            }
            else
            {
                _logger.LogDebug(
                    "GetProducts for Category={Category} completed in {ElapsedMs}ms",
                    category, sw.ElapsedMilliseconds);
            }

            return products;
        }
        catch (Exception ex)
        {
            sw.Stop();
            _logger.LogError(
                ex, 
                "GetProducts failed for Category={Category} after {ElapsedMs}ms",
                category, sw.ElapsedMilliseconds);
            throw;
        }
    }
}

// Placeholder models
public class PaymentRequest 
{ 
    public int OrderId { get; set; }
    public decimal Amount { get; set; }
    public string Currency { get; set; } = "THB";
    public string UserId { get; set; } = "";
    public string ClientIp { get; set; } = "";
    public string CardLastFour { get; set; } = "";
}

public class Product { }
```

---

## ขั้นตอนที่ 919: Configuration Best Practices

### Environment-Specific Configuration

```csharp
// appsettings.json (base, default values)
/*
{
  "App": {
    "Name": "MyApplication",
    "MaxRetryAttempts": 3
  },
  "Database": {
    "CommandTimeout": 30
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}

// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft": "Information"
    }
  },
  "Database": {
    "EnableSensitiveDataLogging": true
  }
}

// appsettings.Production.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  }
}
*/

// ✅ Strongly-typed configuration with validation
public class AppSettings
{
    [Required]
    public string Name { get; set; } = string.Empty;

    [Range(1, 10)]
    public int MaxRetryAttempts { get; set; } = 3;

    [Required]
    public DatabaseSettings Database { get; set; } = new();

    [Required]
    public SecuritySettings Security { get; set; } = new();
}

public class SecuritySettings
{
    [Required]
    [MinLength(32)]
    public string JwtSecret { get; set; } = string.Empty;

    [Range(1, 1440)]
    public int TokenExpirationMinutes { get; set; } = 60;

    public bool RequireHttps { get; set; } = true;
}

// ✅ Registration ใน Program.cs
public static class ConfigurationExtensions
{
    public static IServiceCollection AddValidatedConfiguration(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddOptions<AppSettings>()
            .BindConfiguration("App")
            .ValidateDataAnnotations()
            .ValidateOnStart(); // ✅ Fail fast ถ้า config ผิด

        services.AddOptions<DatabaseSettings>()
            .BindConfiguration(DatabaseSettings.SectionName)
            .ValidateDataAnnotations()
            .Validate(settings =>
            {
                // ✅ Custom validation logic
                if (string.IsNullOrEmpty(settings.ConnectionString))
                    return false;
                
                if (settings.CommandTimeout < 5 || settings.CommandTimeout > 300)
                    return false;
                    
                return true;
            }, "Database configuration is invalid")
            .ValidateOnStart();

        return services;
    }
}
```

### Secrets Management

```csharp
// ✅ ห้าม hardcode secrets ใน code หรือ appsettings.json
// ❌ Bad:
// public const string ConnectionString = "Server=prod-db;Password=SuperSecret123!";

// ✅ ใช้ Environment Variables สำหรับ secrets
// Environment variable: DATABASE__CONNECTIONSTRING=...

// ✅ ใช้ User Secrets ใน development
// dotnet user-secrets set "Database:ConnectionString" "your-connection-string"

// ✅ Secret Provider ที่ปลอดภัย
public static class SecretConfiguration
{
    public static IConfigurationBuilder AddSecrets(
        this IConfigurationBuilder builder,
        IWebHostEnvironment environment,
        string? keyVaultUri = null)
    {
        if (environment.IsDevelopment())
        {
            // ✅ User Secrets สำหรับ Development
            // builder.AddUserSecrets<Program>();
        }
        else if (!string.IsNullOrEmpty(keyVaultUri))
        {
            // ✅ Azure Key Vault สำหรับ Production
            // builder.AddAzureKeyVault(new Uri(keyVaultUri), ...);
        }

        // ✅ Environment variables override ทุกอย่าง
        builder.AddEnvironmentVariables();

        return builder;
    }
}

// ✅ Configuration Service ที่ตรวจสอบ secrets
public class SecureConfigurationService
{
    private readonly AppSettings _settings;
    private readonly ILogger<SecureConfigurationService> _logger;

    public SecureConfigurationService(
        IOptions<AppSettings> options,
        ILogger<SecureConfigurationService> logger)
    {
        _settings = options.Value;
        _logger = logger;
    }

    public void ValidateSecurityConfiguration()
    {
        var warnings = new List<string>();

        if (_settings.Security.JwtSecret.Length < 64)
        {
            warnings.Add("JWT secret should be at least 64 characters for security");
        }

        if (!_settings.Security.RequireHttps)
        {
            warnings.Add("HTTPS is not required - this is insecure for production");
        }

        foreach (var warning in warnings)
        {
            _logger.LogWarning("Security warning: {Warning}", warning);
        }
    }
}
```

---

## ขั้นตอนที่ 920: Code Review Checklist สำหรับ .NET Projects

### Checklist ที่ครอบคลุม

```csharp
/*
=====================================================
CODE REVIEW CHECKLIST สำหรับ .NET Projects
=====================================================

## 1. CORRECTNESS (ความถูกต้อง)
□ Logic ทำงานถูกต้องตาม requirements
□ Edge cases ถูกจัดการ (null, empty, boundary values)
□ Error handling ครอบคลุมทุก failure paths
□ Concurrency issues ไม่มี (race conditions, deadlocks)
□ Security vulnerabilities ไม่มี (SQL injection, XSS, etc.)

## 2. CODE QUALITY
□ Single Responsibility Principle ถูกปฏิบัติ
□ DRY - ไม่มีโค้ดซ้ำที่ไม่จำเป็น
□ ชื่อ variables, methods, classes สื่อความหมายชัดเจน
□ ไม่มี magic strings/numbers
□ ไม่มี commented-out code (ลบออกหรืออธิบายว่าทำไมยังอยู่)
□ Methods ไม่ยาวเกิน (แนะนำ < 30 lines)
□ Complexity ต่ำ (cyclomatic complexity < 10)

## 3. ASYNC/AWAIT
□ async void ไม่มี (ยกเว้น event handlers ที่จัดการ exceptions แล้ว)
□ .Result / .Wait() ไม่มี (อาจ deadlock)
□ CancellationToken ถูก pass และใช้งาน
□ ConfigureAwait(false) ใช้ใน library code
□ Async methods มี Async suffix

## 4. EXCEPTION HANDLING
□ Catch specific exceptions (ไม่ catch Exception ทุกอย่าง)
□ Exceptions ไม่ถูก swallowed โดยไม่ log
□ throw; ใช้แทน throw ex; เมื่อ rethrow
□ Custom exceptions มี constructors ครบ
□ finally blocks ไม่มี exception ที่ซ่อน

## 5. DEPENDENCY INJECTION
□ ไม่ใช้ Service Locator (IServiceProvider.GetService ใน business logic)
□ Lifetimes ถูกต้อง (ไม่มี captive dependencies)
□ Constructor ไม่มี logic ซับซ้อน
□ Dependencies inject ผ่าน constructor (ไม่ใช่ property/method)
□ Interfaces ถูกใช้ (ไม่ inject concrete implementations)

## 6. PERFORMANCE
□ LINQ queries ไม่มี N+1 problem
□ Pagination ใช้กับ large datasets
□ Caching ถูกใช้ที่เหมาะสม
□ String concatenation ใช้ StringBuilder ใน loops
□ IDisposable objects ถูก dispose (using statements)
□ Database queries มี indices ที่เหมาะสม
□ Lazy loading ไม่ถูกใช้แทน explicit loading ใน performance-critical paths

## 7. SECURITY
□ Input validation ครอบคลุม
□ SQL Parameterization ใช้ (ไม่ใช่ string concatenation)
□ Sensitive data ไม่ถูก log
□ Authentication/Authorization ถูกต้อง
□ Secrets ไม่ hardcode ใน code
□ HTTPS enforced

## 8. TESTING
□ Unit tests ครอบคลุม happy path
□ Unit tests ครอบคลุม edge cases
□ Test names บอกว่าทดสอบอะไร
□ Tests ไม่ depend กันเอง (independent)
□ Mocks ใช้อย่างเหมาะสม
□ Integration tests มีสำหรับ critical paths

## 9. LOGGING
□ Log levels เหมาะสม
□ Structured logging ใช้ (message templates, ไม่ใช่ interpolation)
□ Sensitive data ไม่อยู่ใน log messages
□ สำคัญ business events ถูก log
□ Performance-critical operations มี timing logs

## 10. DOCUMENTATION & MAINTAINABILITY
□ Public APIs มี XML documentation
□ Complex logic มี comments อธิบาย "ทำไม" (ไม่ใช่ "ทำอะไร")
□ TODO comments มี ticket/issue reference
□ Breaking changes มี migration guide
□ Configuration options มี documentation

=====================================================
*/

// ✅ ตัวอย่าง Code ที่ผ่าน Review Checklist
public class ExemplaryService
{
    private readonly IOrderRepository _orderRepository;
    private readonly IEmailService _emailService;
    private readonly ILogger<ExemplaryService> _logger;
    private readonly IOptions<OrderSettings> _settings;

    /// <summary>
    /// Creates a new order processing service.
    /// </summary>
    public ExemplaryService(
        IOrderRepository orderRepository,
        IEmailService emailService,
        ILogger<ExemplaryService> logger,
        IOptions<OrderSettings> settings)
    {
        _orderRepository = orderRepository 
            ?? throw new ArgumentNullException(nameof(orderRepository));
        _emailService = emailService 
            ?? throw new ArgumentNullException(nameof(emailService));
        _logger = logger 
            ?? throw new ArgumentNullException(nameof(logger));
        _settings = settings 
            ?? throw new ArgumentNullException(nameof(settings));
    }

    /// <summary>
    /// Processes an order asynchronously.
    /// </summary>
    /// <param name="request">The order creation request.</param>
    /// <param name="cancellationToken">Cancellation token for the operation.</param>
    /// <returns>The created order.</returns>
    /// <exception cref="ValidationException">Thrown when request validation fails.</exception>
    /// <exception cref="BusinessRuleException">Thrown when a business rule is violated.</exception>
    public async Task<Order> ProcessOrderAsync(
        CreateOrderRequest request,
        CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(request);

        _logger.LogInformation(
            "Processing order for customer {CustomerId} with {ItemCount} items",
            request.CustomerId, request.Items?.Count ?? 0);

        // Validate request
        ValidateRequest(request);

        // Check business rules
        await EnsureCustomerCanOrderAsync(request.CustomerId, cancellationToken);

        Order order;
        try
        {
            order = await _orderRepository.CreateAsync(request, cancellationToken);
        }
        catch (DatabaseException ex)
        {
            _logger.LogError(
                ex, 
                "Failed to create order for customer {CustomerId}",
                request.CustomerId);
            throw new DataAccessException("Failed to create order", ex);
        }

        // Fire-and-forget email notification (ไม่ต้องรอ)
        _ = SendConfirmationEmailAsync(order, cancellationToken);

        _logger.LogInformation(
            "Order {OrderId} created successfully for customer {CustomerId}",
            order.Id, request.CustomerId);

        return order;
    }

    private static void ValidateRequest(CreateOrderRequest request)
    {
        var errors = new List<string>();

        if (request.CustomerId <= 0)
            errors.Add("CustomerId must be positive");

        if (request.Items == null || !request.Items.Any())
            errors.Add("Order must contain at least one item");

        if (errors.Any())
            throw new ValidationException(errors);
    }

    private async Task EnsureCustomerCanOrderAsync(
        int customerId, 
        CancellationToken cancellationToken)
    {
        var maxOrders = _settings.Value.MaxOrdersPerCustomerPerDay;
        var todayOrderCount = await _orderRepository
            .GetTodayOrderCountAsync(customerId, cancellationToken);

        if (todayOrderCount >= maxOrders)
        {
            throw new BusinessRuleException(
                "MAX_DAILY_ORDERS",
                $"Customer {customerId} has reached the daily order limit of {maxOrders}");
        }
    }

    private async Task SendConfirmationEmailAsync(
        Order order, 
        CancellationToken cancellationToken)
    {
        try
        {
            await _emailService.SendOrderConfirmationAsync(order, cancellationToken);
        }
        catch (Exception ex)
        {
            // Log แต่ไม่ throw - email failure ไม่ควร fail the order
            _logger.LogWarning(
                ex, 
                "Failed to send confirmation email for order {OrderId}", 
                order.Id);
        }
    }
}

// Supporting models and interfaces
public class OrderSettings
{
    public const string SectionName = "Orders";
    public int MaxOrdersPerCustomerPerDay { get; set; } = 10;
}

public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public List<OrderItem> Items { get; set; } = new();
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}

public class OrderItem
{
    public int ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

public interface IOrderRepository
{
    Task<Order> CreateAsync(CreateOrderRequest request, CancellationToken cancellationToken);
    Task<int> GetTodayOrderCountAsync(int customerId, CancellationToken cancellationToken);
    Task<List<Order>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<Order?> FindByIdAsync(int id);
    Task UpdateAsync(int id, UpdateUserRequest request);
    Task<IEnumerable<Order>> GetAllByStatusAsync(OrderStatus status);
    Task SaveAsync(Order order);
    Task DeleteAsync(int id);
}

public interface IEmailService
{
    Task SendOrderConfirmationAsync(Order order, CancellationToken cancellationToken);
    Task SendAsync(EmailMessage message);
}

public class EmailMessage
{
    public string To { get; set; } = string.Empty;
    public string Subject { get; set; } = string.Empty;
    public string Body { get; set; } = string.Empty;
}

public interface IScopedService
{
    Task ProcessAsync();
}

public interface ICache
{
    Task<T?> GetAsync<T>(string key);
    Task SetAsync<T>(string key, T value, TimeSpan? expiry = null);
}

public interface IUserRepository
{
    Task<IEnumerable<User>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<User?> FindByIdAsync(int id);
    Task<User> CreateAsync(User user);
    Task UpdateAsync(int id, UpdateUserRequest request);
    Task DeleteAsync(int id);
}

public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;

    public User(string name, string email)
    {
        Name = name;
        Email = email;
    }
}

public class UpdateUserRequest
{
    public string? Name { get; set; }
    public string? Email { get; set; }
}

public interface IUserContext { }
public interface IValidator<T> { }
public interface IEmailProvider
{
    Task SendAsync(EmailMessage message);
}

public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(PaymentRequest request);
}

public class PaymentResult
{
    public bool IsSuccess { get; set; }
    public string? TransactionId { get; set; }
}

public interface IConfigurationService { }
public interface IMemoryCache { }

public class AppDbContext
{
    public IQueryable<Order> Orders { get; set; } = null!;
    public IQueryable<Customer> Customers { get; set; } = null!;
}

public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
}

public class OrderSummary
{
    public int OrderId { get; set; }
    public string CustomerName { get; set; } = string.Empty;
    public decimal Total { get; set; }
}

// Extension method stubs
public static class QueryableExtensions
{
    public static Task<List<T>> ToListAsync<T>(
        this IQueryable<T> query, 
        CancellationToken cancellationToken = default)
    {
        return Task.FromResult(query.ToList());
    }

    public static IQueryable<T> Include<T, TProperty>(
        this IQueryable<T> query, 
        System.Linq.Expressions.Expression<Func<T, TProperty>> navigationPropertyPath)
        where T : class
    {
        return query;
    }

    public static Task<T> FirstAsync<T>(
        this IQueryable<T> query,
        System.Linq.Expressions.Expression<Func<T, bool>> predicate)
    {
        return Task.FromResult(query.First(predicate.Compile()));
    }
}

public class SqlConnection : IDisposable
{
    public SqlConnection(string connectionString) { }
    public SqlCommand CreateCommand() => new SqlCommand();
    public Task OpenAsync() => Task.CompletedTask;
    public Task<SqlTransaction> BeginTransactionAsync()
        => Task.FromResult(new SqlTransaction());
    public void Dispose() { }
}

public class SqlCommand : IDisposable
{
    public string CommandText { get; set; } = string.Empty;
    public Task<int> ExecuteNonQueryAsync() => Task.FromResult(0);
    public void Dispose() { }
}

public class SqlTransaction : IAsyncDisposable
{
    public Task CommitAsync() => Task.CompletedTask;
    public Task RollbackAsync() => Task.CompletedTask;
    public ValueTask DisposeAsync() => ValueTask.CompletedTask;
}

public interface IDbConnection { }
```

---

## สรุปบทเรียน

ในส่วนนี้เราได้เรียนรู้:

| ขั้นตอน | หัวข้อ | สาระสำคัญ |
|---------|--------|-----------|
| 911 | Anti-Patterns | หลีกเลี่ยง Mutable Statics, Service Locator, God Class, Magic Strings |
| 912 | Naming Conventions | PascalCase, camelCase, _underscore prefix สำหรับ private fields |
| 913 | Dependency Injection | Constructor Injection, Lifetimes ที่ถูกต้อง, Options Pattern |
| 914 | Exception Handling | Catch เฉพาะที่รู้วิธีจัดการ, Custom Exceptions, throw; ไม่ใช่ throw ex; |
| 915 | Async/Await | หลีกเลี่ยง async void, .Result/.Wait(), ใช้ CancellationToken |
| 916 | LINQ | หลีกเลี่ยง N+1, ใช้ Projections, เข้าใจ Deferred Execution |
| 917 | Memory Management | IDisposable Pattern, using statements, ArrayPool, StringBuilder |
| 918 | Logging | Structured Logging, Log Levels ที่เหมาะสม, ไม่ log sensitive data |
| 919 | Configuration | Strongly-typed config, Validation on startup, Secrets Management |
| 920 | Code Review | Checklist ครอบคลุม Correctness, Quality, Security, Performance, Testing |

---

## แนวทางการนำไปใช้

1. **เริ่มจาก Code Review Checklist** - ใช้ checklist ใน Step 920 ทุกครั้งที่ review code
2. **ตั้งค่า Static Analysis** - ใช้ Roslyn analyzers, SonarQube, หรือ Roslynator
3. **สร้าง Coding Standards Document** - บันทึก conventions ของทีม
4. **ทำ Code Review อย่างสม่ำเสมอ** - จัดให้มี peer review ทุก PR
5. **เรียนรู้จาก Anti-Patterns** - เมื่อพบปัญหา ให้บันทึกไว้เพื่อแบ่งปันกับทีม

---

## แหล่งข้อมูลเพิ่มเติม

- [Microsoft C# Coding Conventions](https://docs.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions)
- [ASP.NET Core Performance Best Practices](https://docs.microsoft.com/en-us/aspnet/core/performance/performance-best-practices)
- [.NET Async Guidance](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios)
- [Serilog Structured Logging](https://serilog.net/)

---

## การนำทาง

← [Part 91: Microservices Architecture](part91-microservices.md) | [Part 93: Design Patterns Advanced](part93-design-patterns-advanced.md) →

---

*ส่วนที่ 92 จาก 100 | หลักสูตร C# จากผู้เริ่มต้นสู่มืออาชีพ*
