# Part 43: EF Core - Unit of Work & Dependency Injection
## ขั้นตอนที่ 421-430: Architecture Patterns

---

## 🎯 เป้าหมายของ Part นี้
- Unit of Work pattern
- Generic Repository
- Service Layer
- DI Registration
- DbContext lifetime scopes
- Health Checks
- Connection pooling
- โปรแกรม E-Commerce Order Processing

---

## ขั้นตอนที่ 421: Unit of Work Pattern

```csharp
// Data/IUnitOfWork.cs
public interface IUnitOfWork : IDisposable
{
    IProductRepository Products { get; }
    IOrderRepository Orders { get; }
    ICustomerRepository Customers { get; }
    
    Task<int> SaveChangesAsync(CancellationToken ct = default);
    Task BeginTransactionAsync();
    Task CommitAsync();
    Task RollbackAsync();
}

// Data/UnitOfWork.cs
public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _ctx;
    private IDbContextTransaction? _transaction;
    
    // Lazy initialization of repositories
    private IProductRepository? _products;
    private IOrderRepository? _orders;
    private ICustomerRepository? _customers;
    
    public UnitOfWork(AppDbContext ctx) => _ctx = ctx;
    
    public IProductRepository Products 
        => _products ??= new ProductRepository(_ctx);
    public IOrderRepository Orders 
        => _orders ??= new OrderRepository(_ctx);
    public ICustomerRepository Customers 
        => _customers ??= new CustomerRepository(_ctx);
    
    public async Task<int> SaveChangesAsync(CancellationToken ct = default)
        => await _ctx.SaveChangesAsync(ct);
    
    public async Task BeginTransactionAsync()
        => _transaction = await _ctx.Database.BeginTransactionAsync();
    
    public async Task CommitAsync()
    {
        if (_transaction != null)
        {
            await _transaction.CommitAsync();
            await _transaction.DisposeAsync();
            _transaction = null;
        }
    }
    
    public async Task RollbackAsync()
    {
        if (_transaction != null)
        {
            await _transaction.RollbackAsync();
            await _transaction.DisposeAsync();
            _transaction = null;
        }
    }
    
    public void Dispose()
    {
        _transaction?.Dispose();
        _ctx.Dispose();
    }
}
```

---

## ขั้นตอนที่ 422: Generic Repository

```csharp
// Data/IGenericRepository.cs
public interface IGenericRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
    Task<IReadOnlyList<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct = default);
    Task<T?> FirstOrDefaultAsync(Expression<Func<T, bool>> predicate, CancellationToken ct = default);
    Task<bool> AnyAsync(Expression<Func<T, bool>> predicate, CancellationToken ct = default);
    Task<int> CountAsync(Expression<Func<T, bool>>? predicate = null, CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    Task AddRangeAsync(IEnumerable<T> entities, CancellationToken ct = default);
    void Update(T entity);
    void Remove(T entity);
    void RemoveRange(IEnumerable<T> entities);
}

// Data/GenericRepository.cs
public class GenericRepository<T> : IGenericRepository<T> where T : class
{
    protected readonly AppDbContext _ctx;
    protected readonly DbSet<T> _dbSet;
    
    public GenericRepository(AppDbContext ctx)
    {
        _ctx = ctx;
        _dbSet = ctx.Set<T>();
    }
    
    public async Task<T?> GetByIdAsync(int id, CancellationToken ct = default)
        => await _dbSet.FindAsync(new object[] { id }, ct);
    
    public async Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default)
        => await _dbSet.AsNoTracking().ToListAsync(ct);
    
    public async Task<IReadOnlyList<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct = default)
        => await _dbSet.AsNoTracking().Where(predicate).ToListAsync(ct);
    
    public async Task<T?> FirstOrDefaultAsync(Expression<Func<T, bool>> predicate, CancellationToken ct = default)
        => await _dbSet.AsNoTracking().FirstOrDefaultAsync(predicate, ct);
    
    public async Task<bool> AnyAsync(Expression<Func<T, bool>> predicate, CancellationToken ct = default)
        => await _dbSet.AnyAsync(predicate, ct);
    
    public async Task<int> CountAsync(Expression<Func<T, bool>>? predicate = null, CancellationToken ct = default)
        => predicate == null 
            ? await _dbSet.CountAsync(ct) 
            : await _dbSet.CountAsync(predicate, ct);
    
    public async Task AddAsync(T entity, CancellationToken ct = default)
        => await _dbSet.AddAsync(entity, ct);
    
    public async Task AddRangeAsync(IEnumerable<T> entities, CancellationToken ct = default)
        => await _dbSet.AddRangeAsync(entities, ct);
    
    public void Update(T entity) => _dbSet.Update(entity);
    
    public void Remove(T entity) => _dbSet.Remove(entity);
    
    public void RemoveRange(IEnumerable<T> entities) => _dbSet.RemoveRange(entities);
}
```

---

## ขั้นตอนที่ 423: Service Layer

```csharp
// Services/OrderService.cs
public class OrderService
{
    private readonly IUnitOfWork _uow;
    private readonly ILogger<OrderService> _logger;
    
    public OrderService(IUnitOfWork uow, ILogger<OrderService> logger)
    {
        _uow = uow;
        _logger = logger;
    }
    
    public async Task<OrderResult> CreateOrderAsync(CreateOrderRequest request, CancellationToken ct = default)
    {
        // Validate customer
        var customer = await _uow.Customers.GetByIdAsync(request.CustomerId, ct);
        if (customer == null)
            return OrderResult.Fail($"ไม่พบลูกค้า ID {request.CustomerId}");
        
        // Validate products and stock
        var orderItems = new List<OrderItem>();
        foreach (var item in request.Items)
        {
            var product = await _uow.Products.GetByIdAsync(item.ProductId, ct);
            if (product == null)
                return OrderResult.Fail($"ไม่พบสินค้า ID {item.ProductId}");
            
            if (product.Stock < item.Quantity)
                return OrderResult.Fail($"สินค้า '{product.Name}' มีในสต็อกไม่เพียงพอ (มี {product.Stock} ต้องการ {item.Quantity})");
            
            orderItems.Add(new OrderItem
            {
                ProductId = product.Id,
                ProductName = product.Name,
                UnitPrice = product.SellPrice,
                Quantity = item.Quantity
            });
        }
        
        // Begin transaction
        await _uow.BeginTransactionAsync();
        try
        {
            // Create order
            var order = new Order
            {
                CustomerId = request.CustomerId,
                OrderDate = DateTime.UtcNow,
                Status = OrderStatus.Pending,
                Items = orderItems,
                TotalAmount = orderItems.Sum(i => i.UnitPrice * i.Quantity)
            };
            await _uow.Orders.AddAsync(order, ct);
            
            // Deduct stock
            foreach (var item in request.Items)
            {
                var product = await _uow.Products.GetByIdAsync(item.ProductId, ct);
                product!.Stock -= item.Quantity;
                _uow.Products.Update(product);
            }
            
            await _uow.SaveChangesAsync(ct);
            await _uow.CommitAsync();
            
            _logger.LogInformation("สร้างคำสั่งซื้อ {OrderId} สำเร็จ รวม {Total:N0} บาท",
                order.Id, order.TotalAmount);
            
            return OrderResult.Success(order);
        }
        catch (Exception ex)
        {
            await _uow.RollbackAsync();
            _logger.LogError(ex, "ไม่สามารถสร้างคำสั่งซื้อ");
            return OrderResult.Fail("เกิดข้อผิดพลาดในการสร้างคำสั่งซื้อ");
        }
    }
    
    public async Task<PagedResult<OrderDto>> GetOrdersAsync(int page, int pageSize, CancellationToken ct = default)
    {
        var total = await _uow.Orders.CountAsync(null, ct);
        var orders = await _uow.Orders.GetPagedAsync(page, pageSize, ct);
        
        return new PagedResult<OrderDto>
        {
            Items = orders.Select(MapToDto).ToList(),
            TotalCount = total,
            Page = page,
            PageSize = pageSize
        };
    }
    
    private static OrderDto MapToDto(Order o) => new()
    {
        Id = o.Id,
        CustomerName = o.Customer.Name,
        OrderDate = o.OrderDate,
        TotalAmount = o.TotalAmount,
        Status = o.Status.ToString(),
        ItemCount = o.Items.Count
    };
}

// Result types
public class OrderResult
{
    public bool IsSuccess { get; private set; }
    public string? ErrorMessage { get; private set; }
    public Order? Order { get; private set; }
    
    public static OrderResult Success(Order order) => new() { IsSuccess = true, Order = order };
    public static OrderResult Fail(string message) => new() { IsSuccess = false, ErrorMessage = message };
}

public class PagedResult<T>
{
    public List<T> Items { get; set; } = new();
    public int TotalCount { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasNextPage => Page < TotalPages;
    public bool HasPreviousPage => Page > 1;
}
```

---

## ขั้นตอนที่ 424: DI Registration

```csharp
// Program.cs / App.xaml.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.EntityFrameworkCore;

var services = new ServiceCollection();

// DbContext - Scoped lifetime (default for web, need factory for desktop)
services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlite("Data Source=ecommerce.db")
       .EnableSensitiveDataLogging(isDevelopment)
       .LogTo(Console.WriteLine, LogLevel.Information));

// Desktop apps: DbContext Factory pattern (avoid shared context)
services.AddDbContextFactory<AppDbContext>(opt =>
    opt.UseSqlite("Data Source=ecommerce.db"));

// Repositories
services.AddScoped<IProductRepository, ProductRepository>();
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddScoped<ICustomerRepository, CustomerRepository>();

// Unit of Work
services.AddScoped<IUnitOfWork, UnitOfWork>();

// Services
services.AddScoped<OrderService>();
services.AddScoped<ProductService>();

// Logging
services.AddLogging(b => b.AddConsole().SetMinimumLevel(LogLevel.Debug));

var provider = services.BuildServiceProvider();

// Usage in desktop:
using var scope = provider.CreateScope();
var orderService = scope.ServiceProvider.GetRequiredService<OrderService>();
```

---

## ขั้นตอนที่ 425-430: Order Processing Console App

```csharp
// Program.cs - Complete Order Processing Demo
using Microsoft.Extensions.DependencyInjection;

var provider = BuildServiceProvider();

// Ensure DB is created
using (var scope = provider.CreateScope())
{
    var ctx = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await ctx.Database.EnsureCreatedAsync();
    await SeedData(ctx);
}

// Place an order
using (var scope = provider.CreateScope())
{
    var orderService = scope.ServiceProvider.GetRequiredService<OrderService>();
    
    Console.WriteLine("=== สร้างคำสั่งซื้อ ===");
    var result = await orderService.CreateOrderAsync(new CreateOrderRequest
    {
        CustomerId = 1,
        Items = new[]
        {
            new OrderItemRequest { ProductId = 1, Quantity = 2 },
            new OrderItemRequest { ProductId = 3, Quantity = 1 },
        }
    });
    
    if (result.IsSuccess)
        Console.WriteLine($"✅ สำเร็จ! คำสั่งซื้อ #{result.Order!.Id} รวม ฿{result.Order.TotalAmount:N0}");
    else
        Console.WriteLine($"❌ ผิดพลาด: {result.ErrorMessage}");
}

// View orders
using (var scope = provider.CreateScope())
{
    var orderService = scope.ServiceProvider.GetRequiredService<OrderService>();
    var orders = await orderService.GetOrdersAsync(page: 1, pageSize: 10);
    
    Console.WriteLine($"\n=== คำสั่งซื้อ ({orders.TotalCount} รายการ) ===");
    foreach (var o in orders.Items)
        Console.WriteLine($"#{o.Id} {o.CustomerName}: ฿{o.TotalAmount:N0} ({o.Status})");
}

static ServiceProvider BuildServiceProvider()
{
    var s = new ServiceCollection();
    s.AddDbContext<AppDbContext>(o => o.UseSqlite("Data Source=ecommerce.db"));
    s.AddScoped<IUnitOfWork, UnitOfWork>();
    s.AddScoped<OrderService>();
    s.AddLogging(b => b.AddConsole());
    return s.BuildServiceProvider();
}

static async Task SeedData(AppDbContext ctx)
{
    if (await ctx.Customers.AnyAsync()) return;
    
    ctx.Customers.AddRange(
        new Customer { Name = "สมชาย ลูกค้าดี", Email = "somchai@email.com" },
        new Customer { Name = "สมหญิง ใจดี", Email = "somying@email.com" }
    );
    ctx.Products.AddRange(
        new Product { Name = "MacBook Pro", Price = 65000, Stock = 10, Code = "MAC001" },
        new Product { Name = "iPad Air", Price = 22000, Stock = 25, Code = "IPD001" },
        new Product { Name = "AirPods", Price = 8000, Stock = 50, Code = "APD001" }
    );
    await ctx.SaveChangesAsync();
}
```

---

## 📝 สรุป Part 43

| Pattern | ประโยชน์ |
|---------|---------|
| Unit of Work | Shares DbContext, atomic SaveChanges |
| Generic Repository | Reuse CRUD across entities |
| Service Layer | Business logic + transactions |
| Scoped DbContext | One context per request/scope |
| Factory Pattern | Desktop apps: controlled lifetime |
| DI Container | Loose coupling, testability |

**DbContext Lifetime Scopes:**
- `AddDbContext` → Scoped (new context per DI scope)
- `AddDbContextFactory` → Used to create scoped contexts on demand
- **ห้าม** ใช้ Singleton DbContext (thread safety issues)

---

**ก่อนหน้า → [Part 42: EF Core Migrations](part42-ef-migrations.md)**  
**ต่อไป → [Part 44: LINQ Advanced](part44-linq-advanced.md)**
