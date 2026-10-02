# Part 80: Architecture Review & Best Practices
## ขั้นตอนที่ 791-800: สรุปสถาปัตยกรรมและ Best Practices

---

## 🎯 เป้าหมายของ Part นี้
- สรุปสถาปัตยกรรมทั้งหมดที่เรียนมา
- เลือก architecture ที่เหมาะสมกับ project
- Monolith vs Microservices trade-offs
- API design principles
- Error handling strategy
- Logging & monitoring strategy
- Database design best practices
- Security checklist

---

## ขั้นตอนที่ 791: Architecture Selection Guide

```
การเลือก Architecture ที่เหมาะสม
═══════════════════════════════════════════════════════════════

Team Size / Domain Complexity Matrix:
                     │  Simple    │  Moderate  │  Complex
─────────────────────┼────────────┼────────────┼─────────────
1-5 developers       │  Monolith  │  Modular   │  Modular
                     │  Clean Arch│  Monolith  │  Monolith
─────────────────────┼────────────┼────────────┼─────────────
5-20 developers      │  Modular   │  Modular   │  Microservices
                     │  Monolith  │  Monolith  │  (careful!)
─────────────────────┼────────────┼────────────┼─────────────
20+ developers       │  Modular   │  Microservices  │  Microservices
                     │  Monolith  │             │  + DDD
```

```csharp
// Guiding questions before choosing architecture:
// 1. How many developers? (< 10 → start with monolith)
// 2. Domain complexity? (multiple bounded contexts → consider microservices later)
// 3. Scale requirements? (> 1M users or specific components need independent scaling)
// 4. Team topology? (Conway's Law: architecture follows team structure)
// 5. Deployment frequency? (microservices help if different services deploy differently)

// Evolutionary architecture: start simple, evolve as needed
// Step 1: Clean Architecture Monolith
// Step 2: Modular Monolith (clear boundaries, separate assemblies)
// Step 3: Microservices (extract when boundaries are proven stable)
```

---

## ขั้นตอนที่ 792: Clean Architecture Review

```
Clean Architecture Layers:
════════════════════════════════════

         ┌─────────────────────────┐
         │    Presentation         │  WPF / Blazor / API
         │   (UI, Controllers)     │
         ├─────────────────────────┤
         │    Infrastructure       │  EF Core, Redis, Email
         │  (DB, External APIs)    │
         ├─────────────────────────┤
         │    Application          │  Use Cases, CQRS, DTOs
         │  (Commands, Queries)    │
         ├─────────────────────────┤
         │       Domain            │  Entities, Value Objects
         │  (Business Rules)       │  Domain Events, Repos
         └─────────────────────────┘
         
         Dependency Rule: outer → inner ONLY
         Domain knows nothing about Infrastructure
```

```csharp
// Dependency directions:
// Presentation → Application → Domain ← Infrastructure (via interfaces)

// Domain layer: no dependencies on framework or infrastructure
namespace MyApp.Domain.Entities;
public class Order : AggregateRoot
{
    // Pure business logic, no EF attributes, no external references
    public Money TotalAmount => OrderLines.Sum(l => l.LineTotal);
    
    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Can only confirm pending orders");
        Status = OrderStatus.Confirmed;
        AddDomainEvent(new OrderConfirmed(Id, DateTime.UtcNow));
    }
}

// Application: orchestrates domain, defines interfaces
namespace MyApp.Application.Orders.Commands;
public record ConfirmOrderCommand(Guid OrderId) : IRequest<Result>;

public class ConfirmOrderHandler : IRequestHandler<ConfirmOrderCommand, Result>
{
    private readonly IOrderRepository _orders;
    private readonly IUnitOfWork _uow;
    
    public async Task<Result> Handle(ConfirmOrderCommand cmd, CancellationToken ct)
    {
        var order = await _orders.GetByIdAsync(cmd.OrderId, ct);
        if (order == null) return Result.Failure("Order not found");
        
        order.Confirm(); // domain method
        await _uow.SaveChangesAsync(ct);
        return Result.Success();
    }
}

// Infrastructure: implements interfaces from Application/Domain
namespace MyApp.Infrastructure.Repositories;
public class OrderRepository : IOrderRepository  // implements Domain interface
{
    private readonly AppDbContext _db;
    // EF Core operations here - Domain doesn't know about this
}
```

---

## ขั้นตอนที่ 793: API Design Best Practices

```csharp
// RESTful API design principles

// 1. Resource-based URLs (nouns, not verbs)
// ✅ GET /api/orders/{id}
// ❌ GET /api/getOrder?id=123

// 2. HTTP verbs semantics
// GET    → read (idempotent, safe)
// POST   → create (not idempotent)
// PUT    → replace completely (idempotent)
// PATCH  → partial update (not necessarily idempotent)
// DELETE → delete (idempotent)

// 3. Versioning strategy
// URL:    GET /api/v1/orders  (easiest for clients)
// Header: Api-Version: 1.0
// Query:  GET /api/orders?version=1.0

// 4. Consistent error responses
public record ApiError(string Code, string Message, Dictionary<string, string[]>? Errors = null);

// 400 Bad Request
{ "code": "VALIDATION_ERROR", "message": "Invalid input", "errors": { "email": ["Email is required"] } }
// 404 Not Found
{ "code": "NOT_FOUND", "message": "Order #123 not found" }
// 500 Internal Server Error
{ "code": "INTERNAL_ERROR", "message": "An unexpected error occurred" }

// 5. Pagination
public record PagedResult<T>(IEnumerable<T> Items, int TotalCount, int Page, int PageSize)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasNext => Page < TotalPages;
    public bool HasPrev => Page > 1;
    
    // Link header (RFC 5988)
    public string BuildLinkHeader(string baseUrl) => 
        $"<{baseUrl}?page={Page-1}>; rel=\"prev\", <{baseUrl}?page={Page+1}>; rel=\"next\"";
}

// 6. HATEOAS (optional but good for discoverability)
public record OrderDto(
    Guid Id, 
    string Status, 
    decimal Amount,
    IEnumerable<Link> Links);  // self, cancel, track

public record Link(string Rel, string Href, string Method);
```

---

## ขั้นตอนที่ 794: Error Handling Strategy

```csharp
// Structured error handling approach

// 1. Domain errors (business rule violations)
public abstract class DomainException : Exception
{
    public string Code { get; }
    protected DomainException(string code, string message) : base(message) => Code = code;
}
public class InsufficientStockException : DomainException
{
    public InsufficientStockException(int productId, int requested, int available) 
        : base("INSUFFICIENT_STOCK", $"Product {productId}: requested {requested}, available {available}")
    { }
}

// 2. Result<T> for expected failures (no exceptions for control flow)
public class Result<T>
{
    public T? Value { get; }
    public string? Error { get; }
    public bool IsSuccess => Error == null;
    
    private Result(T value) => Value = value;
    private Result(string error) => Error = error;
    
    public static Result<T> Ok(T value) => new(value);
    public static Result<T> Fail(string error) => new(error);
    
    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<string, TResult> onFailure)
        => IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}

// 3. Global exception handler
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;
    
    public async Task InvokeAsync(HttpContext ctx)
    {
        try
        {
            await _next(ctx);
        }
        catch (DomainException ex)
        {
            _logger.LogWarning("Domain exception: {Code} - {Message}", ex.Code, ex.Message);
            ctx.Response.StatusCode = 400;
            await ctx.Response.WriteAsJsonAsync(new ApiError(ex.Code, ex.Message));
        }
        catch (NotFoundException ex)
        {
            ctx.Response.StatusCode = 404;
            await ctx.Response.WriteAsJsonAsync(new ApiError("NOT_FOUND", ex.Message));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            ctx.Response.StatusCode = 500;
            await ctx.Response.WriteAsJsonAsync(new ApiError("INTERNAL_ERROR", "An unexpected error occurred"));
        }
    }
}

// 4. Validation at boundaries
public class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Items).NotEmpty().WithMessage("Order must have at least one item");
        RuleForEach(x => x.Items).SetValidator(new OrderItemValidator());
    }
}
```

---

## ขั้นตอนที่ 795: Database Best Practices

```csharp
// Database design principles

// 1. Always use strong typed IDs (no int/Guid mix-up)
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public static implicit operator Guid(OrderId id) => id.Value;
}

// 2. Soft delete pattern (never delete data)
public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
}

// Global query filter
modelBuilder.Entity<Order>().HasQueryFilter(o => !o.IsDeleted);

// 3. Concurrency tokens
public class Order
{
    [Timestamp]
    public byte[] RowVersion { get; set; } = null!;
    // EF throws DbUpdateConcurrencyException on conflict
}

// 4. Auditing
public interface IAuditable
{
    DateTime CreatedAt { get; set; }
    string CreatedBy { get; set; }
    DateTime? UpdatedAt { get; set; }
    string? UpdatedBy { get; set; }
}

// 5. Query optimization
// Always project to DTOs - don't load entire entities for read operations
var orders = await db.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .Select(o => new OrderSummaryDto(o.Id, o.CustomerName, o.TotalAmount))  // ✅ projection
    .AsNoTracking()  // ✅ no change tracking for read-only queries
    .ToListAsync();

// Use compiled queries for hot paths
private static readonly Func<AppDbContext, int, Task<Product?>> GetProductById =
    EF.CompileAsyncQuery((AppDbContext db, int id) =>
        db.Products.FirstOrDefault(p => p.Id == id));

// 6. Migration strategy
// - Never modify applied migrations
// - Always create new migration for changes
// - Include rollback script
// - Test migrations in staging before production
```

---

## ขั้นตอนที่ 796: Security Checklist

```csharp
// Production security checklist

// Authentication & Authorization
// ✅ JWT with short expiry (15min) + refresh tokens
// ✅ Refresh token rotation (invalidate old on use)
// ✅ Role-based + policy-based authorization
// ✅ Resource-based authorization (user can only access their own resources)

// Data Protection
// ✅ HTTPS everywhere (HSTS)
// ✅ Password hashing (Argon2id or BCrypt, never MD5/SHA1)
// ✅ Sensitive data encrypted at rest (AES-256-GCM)
// ✅ No secrets in code/config files (use environment variables/Key Vault)
// ✅ Parameterized queries (never string concatenation)

// API Security
// ✅ Rate limiting (prevent brute force)
// ✅ Input validation (server-side, always)
// ✅ Output encoding (prevent XSS)
// ✅ CORS properly configured (not *)
// ✅ Security headers (CSP, X-Frame-Options, etc.)

// Infrastructure
// ✅ Dependencies up to date (dotnet list package --vulnerable)
// ✅ Container runs as non-root user
// ✅ Minimal attack surface (only expose needed ports)
// ✅ Audit logging for sensitive operations

// Example: dotnet package vulnerability check
// dotnet list package --vulnerable --include-transitive
```

---

## ขั้นตอนที่ 797-800: Complete Architecture Blueprint

```csharp
// Full application structure for a production .NET app

// Solution structure:
// MyApp.sln
// ├── src/
// │   ├── MyApp.Domain/           -- Entities, VOs, Domain Events, Interfaces
// │   ├── MyApp.Application/      -- Use Cases, CQRS, Validators, Mappings
// │   ├── MyApp.Infrastructure/   -- EF Core, Redis, Email, File Storage
// │   ├── MyApp.Api/              -- Controllers, Middleware, DTOs
// │   └── MyApp.Shared/           -- Contracts, Shared types
// ├── tests/
// │   ├── MyApp.Domain.Tests/
// │   ├── MyApp.Application.Tests/
// │   ├── MyApp.Integration.Tests/
// │   └── MyApp.Architecture.Tests/
// └── tools/
//     └── MyApp.Migrations/       -- EF Migration project (separate)

// Key decisions for production readiness:
// 1. Database: PostgreSQL (reliability, JSONB, full-text search)
// 2. Cache: Redis (distributed, Pub/Sub, sessions)
// 3. Message Bus: RabbitMQ (simple) or Azure Service Bus (enterprise)
// 4. Logging: Serilog → Seq (dev) / Azure Monitor (prod)
// 5. Tracing: OpenTelemetry → Jaeger (dev) / Application Insights (prod)
// 6. Secrets: dotnet user-secrets (dev) / Azure Key Vault (prod)
// 7. Health checks: /health/live, /health/ready
// 8. Container: Docker multi-stage build, non-root user
// 9. CI/CD: GitHub Actions (build/test/push/deploy)
// 10. Monitoring: Prometheus + Grafana dashboards

// Deployment target checklist:
public static class ProductionReadinessChecklist
{
    public static Dictionary<string, bool> Check(IConfiguration cfg) => new()
    {
        ["HTTPS configured"] = cfg.GetValue<bool>("UseHttps"),
        ["HSTS enabled"] = cfg.GetValue<bool>("UseHsts"),
        ["Health checks registered"] = true,
        ["Structured logging"] = cfg["Serilog"] != null,
        ["Rate limiting"] = cfg["RateLimiting"] != null,
        ["Database connection string set"] = !string.IsNullOrEmpty(cfg.GetConnectionString("Default")),
        ["JWT secret set"] = !string.IsNullOrEmpty(cfg["Jwt:Secret"]),
        ["CORS configured"] = cfg["Cors:Origins"] != null,
    };
}
```

---

## 📝 สรุป Parts 71-80

| Part | หัวข้อ | Steps |
|------|--------|-------|
| 71 | Observability & Monitoring | 701-710 |
| 72 | Resilience Patterns | 711-720 |
| 73 | Advanced Testing | 721-730 |
| 74 | Message Bus | 731-740 |
| 75 | Background Services | 741-750 |
| 76 | Functional C# | 751-760 |
| 77 | WPF Advanced | 761-770 |
| 78 | WinForms Advanced | 771-780 |
| 79 | Code Quality | 781-790 |
| 80 | Architecture Review | 791-800 |

ถึงจุดนี้เราได้เรียนรู้ครบ 800 ขั้นตอน ครอบคลุม:
- C# ตั้งแต่พื้นฐานถึงขั้นสูง
- WPF, WinForms, Blazor, MAUI
- Clean Architecture, DDD, CQRS, Event Sourcing
- Microservices, gRPC, GraphQL, SignalR
- Testing, Security, Performance, DevOps

---

**ก่อนหน้า → [Part 79: Code Quality](part79-code-quality.md)**  
**ต่อไป → [Part 81: Capstone Project - E-Commerce Platform](../part81-100/part81-capstone-ecommerce.md)**
