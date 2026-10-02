# Part 97: Database Advanced Patterns
## ขั้นตอนที่ 961-970: Database ขั้นสูง

ในส่วนนี้เราจะศึกษา Pattern และเทคนิคขั้นสูงสำหรับการจัดการฐานข้อมูลด้วย C# และ Entity Framework Core ครอบคลุมตั้งแต่ Multi-tenancy, Sharding, Read Replicas, Temporal Tables, JSON Columns, Full-text Search, Change Data Capture, Database Testing, Interceptors ไปจนถึงกลยุทธ์การ Migrate ข้อมูลแบบ Zero-downtime

---

## ขั้นตอนที่ 961: Multi-tenancy Patterns กับ EF Core

### แนวคิด Multi-tenancy

Multi-tenancy คือการออกแบบระบบให้รองรับผู้เช่า (tenant) หลายรายบนโครงสร้างพื้นฐานเดียวกัน มีรูปแบบหลัก 3 แบบ:

1. **Row-level** - ข้อมูลของทุก tenant อยู่ในตารางเดียวกัน แยกด้วย `TenantId`
2. **Schema-level** - แต่ละ tenant มี schema แยกกัน แต่อยู่ใน database เดียวกัน
3. **Database-level** - แต่ละ tenant มี database แยกกันโดยสมบูรณ์

### 1. Row-level Multi-tenancy

```csharp
// TenantContext.cs - บริบท Tenant
public interface ITenantContext
{
    string TenantId { get; }
}

public class TenantContext : ITenantContext
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public TenantContext(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public string TenantId =>
        _httpContextAccessor.HttpContext?.User?.FindFirst("tenant_id")?.Value
        ?? throw new InvalidOperationException("TenantId not found in claims");
}

// Base Entity สำหรับ Row-level tenancy
public abstract class TenantEntity
{
    public int Id { get; set; }
    public string TenantId { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}

// Product Entity
public class Product : TenantEntity
{
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Stock { get; set; }
}

// MultiTenantDbContext.cs
public class MultiTenantDbContext : DbContext
{
    private readonly ITenantContext _tenantContext;

    public MultiTenantDbContext(
        DbContextOptions<MultiTenantDbContext> options,
        ITenantContext tenantContext) : base(options)
    {
        _tenantContext = tenantContext;
    }

    public DbSet<Product> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Global query filter สำหรับ Row-level security
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => p.TenantId == _tenantContext.TenantId);

        // Index บน TenantId เพื่อ performance
        modelBuilder.Entity<Product>()
            .HasIndex(p => p.TenantId)
            .HasDatabaseName("IX_Products_TenantId");

        // Composite index
        modelBuilder.Entity<Product>()
            .HasIndex(p => new { p.TenantId, p.Name })
            .HasDatabaseName("IX_Products_TenantId_Name");
    }

    public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // Auto-set TenantId เมื่อ insert
        foreach (var entry in ChangeTracker.Entries<TenantEntity>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.TenantId = _tenantContext.TenantId;
            }
        }
        return base.SaveChangesAsync(cancellationToken);
    }
}
```

### 2. Schema-level Multi-tenancy

```csharp
// SchemaMultiTenantDbContext.cs
public class SchemaMultiTenantDbContext : DbContext
{
    private readonly string _schema;

    public SchemaMultiTenantDbContext(string connectionString, string tenantId)
    {
        _schema = $"tenant_{tenantId}";
        // สร้าง connection ด้วย connection string ที่กำหนด
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // ตั้ง schema สำหรับทุก entity
        modelBuilder.HasDefaultSchema(_schema);

        modelBuilder.Entity<Product>(entity =>
        {
            entity.ToTable("Products", _schema);
        });
    }
}

// TenantSchemaManager.cs - จัดการการสร้าง schema
public class TenantSchemaManager
{
    private readonly IDbConnectionFactory _connectionFactory;
    private readonly ILogger<TenantSchemaManager> _logger;

    public TenantSchemaManager(
        IDbConnectionFactory connectionFactory,
        ILogger<TenantSchemaManager> logger)
    {
        _connectionFactory = connectionFactory;
        _logger = logger;
    }

    public async Task ProvisionTenantAsync(string tenantId)
    {
        var schema = $"tenant_{tenantId}";

        using var connection = _connectionFactory.CreateConnection();
        await connection.OpenAsync();

        // สร้าง schema ใหม่
        var createSchema = $"CREATE SCHEMA IF NOT EXISTS {schema}";
        await ExecuteNonQueryAsync(connection, createSchema);

        // สร้างตารางใน schema ใหม่
        var createTables = $@"
            CREATE TABLE IF NOT EXISTS {schema}.Products (
                Id SERIAL PRIMARY KEY,
                Name VARCHAR(255) NOT NULL,
                Price DECIMAL(18,2) NOT NULL,
                Stock INT NOT NULL DEFAULT 0,
                CreatedAt TIMESTAMPTZ DEFAULT NOW()
            );
            CREATE INDEX IF NOT EXISTS IX_{schema}_Products_Name
                ON {schema}.Products(Name);
        ";
        await ExecuteNonQueryAsync(connection, createTables);

        _logger.LogInformation("Provisioned schema {Schema} for tenant {TenantId}",
            schema, tenantId);
    }

    private async Task ExecuteNonQueryAsync(IDbConnection connection, string sql)
    {
        using var command = connection.CreateCommand();
        command.CommandText = sql;
        await ((DbCommand)command).ExecuteNonQueryAsync();
    }
}
```

### 3. Database-level Multi-tenancy

```csharp
// TenantDatabaseResolver.cs
public interface ITenantDatabaseResolver
{
    string GetConnectionString(string tenantId);
}

public class TenantDatabaseResolver : ITenantDatabaseResolver
{
    private readonly ITenantRepository _tenantRepository;
    private readonly IMemoryCache _cache;

    public TenantDatabaseResolver(
        ITenantRepository tenantRepository,
        IMemoryCache cache)
    {
        _tenantRepository = tenantRepository;
        _cache = cache;
    }

    public string GetConnectionString(string tenantId)
    {
        var cacheKey = $"tenant_connstring_{tenantId}";

        if (_cache.TryGetValue(cacheKey, out string? cached))
            return cached!;

        var tenant = _tenantRepository.GetByIdAsync(tenantId).GetAwaiter().GetResult();
        if (tenant == null)
            throw new TenantNotFoundException(tenantId);

        var connectionString = tenant.DatabaseConnectionString;

        _cache.Set(cacheKey, connectionString,
            TimeSpan.FromMinutes(5));

        return connectionString;
    }
}

// DatabaseLevelDbContextFactory.cs
public class DatabaseLevelDbContextFactory
{
    private readonly ITenantDatabaseResolver _resolver;
    private readonly ILoggerFactory _loggerFactory;

    public DatabaseLevelDbContextFactory(
        ITenantDatabaseResolver resolver,
        ILoggerFactory loggerFactory)
    {
        _resolver = resolver;
        _loggerFactory = loggerFactory;
    }

    public AppDbContext CreateForTenant(string tenantId)
    {
        var connectionString = _resolver.GetConnectionString(tenantId);

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(connectionString)
            .UseLoggerFactory(_loggerFactory)
            .Options;

        return new AppDbContext(options);
    }
}

// TenantNotFoundException.cs
public class TenantNotFoundException : Exception
{
    public string TenantId { get; }

    public TenantNotFoundException(string tenantId)
        : base($"Tenant '{tenantId}' not found")
    {
        TenantId = tenantId;
    }
}
```

---

## ขั้นตอนที่ 962: Database Sharding แนวคิดและการ Implement

### แนวคิด Database Sharding

Sharding คือการแบ่ง (partition) ข้อมูลออกเป็นหลาย shard (ฐานข้อมูลย่อย) เพื่อกระจาย load และขยาย capacity

```csharp
// ShardingStrategy.cs - กลยุทธ์การ shard
public interface IShardingStrategy
{
    int GetShardId(string shardKey, int totalShards);
}

// Hash-based sharding
public class HashShardingStrategy : IShardingStrategy
{
    public int GetShardId(string shardKey, int totalShards)
    {
        var hash = Math.Abs(shardKey.GetHashCode());
        return hash % totalShards;
    }
}

// Range-based sharding
public class RangeShardingStrategy : IShardingStrategy
{
    private readonly List<(long Min, long Max, int ShardId)> _ranges;

    public RangeShardingStrategy(List<(long Min, long Max, int ShardId)> ranges)
    {
        _ranges = ranges.OrderBy(r => r.Min).ToList();
    }

    public int GetShardId(string shardKey, int totalShards)
    {
        if (long.TryParse(shardKey, out var numericKey))
        {
            var range = _ranges.FirstOrDefault(r =>
                numericKey >= r.Min && numericKey <= r.Max);

            if (range != default)
                return range.ShardId;
        }

        // Fallback to hash
        return Math.Abs(shardKey.GetHashCode()) % totalShards;
    }
}

// ShardRouter.cs
public class ShardRouter
{
    private readonly IShardingStrategy _strategy;
    private readonly List<string> _connectionStrings;

    public ShardRouter(
        IShardingStrategy strategy,
        List<string> connectionStrings)
    {
        _strategy = strategy;
        _connectionStrings = connectionStrings;
    }

    public string GetConnectionString(string shardKey)
    {
        var shardId = _strategy.GetShardId(shardKey, _connectionStrings.Count);
        return _connectionStrings[shardId];
    }
}

// ShardedOrderDbContext.cs
public class ShardedOrderDbContext : DbContext
{
    public ShardedOrderDbContext(DbContextOptions<ShardedOrderDbContext> options)
        : base(options) { }

    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }
}

// ShardedOrderRepository.cs
public class ShardedOrderRepository
{
    private readonly ShardRouter _shardRouter;
    private readonly ILoggerFactory _loggerFactory;

    public ShardedOrderRepository(
        ShardRouter shardRouter,
        ILoggerFactory loggerFactory)
    {
        _shardRouter = shardRouter;
        _loggerFactory = loggerFactory;
    }

    private ShardedOrderDbContext CreateContext(string customerId)
    {
        var connectionString = _shardRouter.GetConnectionString(customerId);

        var options = new DbContextOptionsBuilder<ShardedOrderDbContext>()
            .UseNpgsql(connectionString)
            .UseLoggerFactory(_loggerFactory)
            .Options;

        return new ShardedOrderDbContext(options);
    }

    public async Task<Order> CreateOrderAsync(Order order)
    {
        using var context = CreateContext(order.CustomerId.ToString());
        context.Orders.Add(order);
        await context.SaveChangesAsync();
        return order;
    }

    public async Task<List<Order>> GetOrdersByCustomerAsync(Guid customerId)
    {
        using var context = CreateContext(customerId.ToString());
        return await context.Orders
            .Where(o => o.CustomerId == customerId)
            .Include(o => o.Items)
            .ToListAsync();
    }

    // Cross-shard query - ต้องระวังเรื่อง performance
    public async Task<List<Order>> GetAllPendingOrdersAsync()
    {
        var allOrders = new List<Order>();
        var tasks = _shardRouter.GetAllConnectionStrings()
            .Select(async connStr =>
            {
                var options = new DbContextOptionsBuilder<ShardedOrderDbContext>()
                    .UseNpgsql(connStr)
                    .Options;

                using var ctx = new ShardedOrderDbContext(options);
                return await ctx.Orders
                    .Where(o => o.Status == OrderStatus.Pending)
                    .ToListAsync();
            });

        var results = await Task.WhenAll(tasks);
        return results.SelectMany(r => r).ToList();
    }
}

// Order Models
public class Order
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public OrderStatus Status { get; set; }
    public decimal TotalAmount { get; set; }
    public DateTime CreatedAt { get; set; }
    public List<OrderItem> Items { get; set; } = new();
}

public class OrderItem
{
    public Guid Id { get; set; }
    public Guid OrderId { get; set; }
    public string ProductName { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

public enum OrderStatus { Pending, Processing, Shipped, Delivered, Cancelled }
```

---

## ขั้นตอนที่ 963: Read Replicas กับ Connection String Routing ใน EF Core

### แนวคิด Read Replicas

Read Replicas ช่วยกระจาย load โดยส่ง query แบบ read-only ไปยัง replica แทน primary database

```csharp
// ReadWriteDbContext.cs
public class ReadWriteDbContext : DbContext
{
    private readonly IDatabaseSelector _databaseSelector;

    public ReadWriteDbContext(
        DbContextOptions<ReadWriteDbContext> options,
        IDatabaseSelector databaseSelector) : base(options)
    {
        _databaseSelector = databaseSelector;
    }

    public DbSet<Customer> Customers { get; set; }
    public DbSet<Product> Products { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured)
        {
            // กำหนด connection string ตาม read/write mode
            var connectionString = _databaseSelector.GetConnectionString();
            optionsBuilder.UseNpgsql(connectionString);
        }
    }
}

// IDatabaseSelector.cs
public interface IDatabaseSelector
{
    string GetConnectionString();
    void UseReadReplica();
    void UsePrimary();
}

// RoundRobinDatabaseSelector.cs
public class RoundRobinDatabaseSelector : IDatabaseSelector
{
    private readonly string _primaryConnectionString;
    private readonly List<string> _replicaConnectionStrings;
    private int _currentReplicaIndex = 0;
    private bool _useReplica = false;
    private readonly object _lock = new();

    public RoundRobinDatabaseSelector(
        string primaryConnectionString,
        List<string> replicaConnectionStrings)
    {
        _primaryConnectionString = primaryConnectionString;
        _replicaConnectionStrings = replicaConnectionStrings;
    }

    public string GetConnectionString()
    {
        if (!_useReplica || !_replicaConnectionStrings.Any())
            return _primaryConnectionString;

        lock (_lock)
        {
            var connStr = _replicaConnectionStrings[_currentReplicaIndex];
            _currentReplicaIndex =
                (_currentReplicaIndex + 1) % _replicaConnectionStrings.Count;
            return connStr;
        }
    }

    public void UseReadReplica() => _useReplica = true;
    public void UsePrimary() => _useReplica = false;
}

// ReadOnlyRepository.cs - Repository ที่ใช้ Read Replica
public class CustomerReadRepository
{
    private readonly IDbContextFactory<ReadWriteDbContext> _contextFactory;
    private readonly IDatabaseSelector _databaseSelector;

    public CustomerReadRepository(
        IDbContextFactory<ReadWriteDbContext> contextFactory,
        IDatabaseSelector databaseSelector)
    {
        _contextFactory = contextFactory;
        _databaseSelector = databaseSelector;
    }

    public async Task<Customer?> GetByIdAsync(Guid id)
    {
        _databaseSelector.UseReadReplica();
        try
        {
            await using var context = await _contextFactory.CreateDbContextAsync();
            return await context.Customers
                .AsNoTracking()
                .FirstOrDefaultAsync(c => c.Id == id);
        }
        finally
        {
            _databaseSelector.UsePrimary();
        }
    }

    public async Task<List<Customer>> SearchAsync(string searchTerm, int page, int pageSize)
    {
        _databaseSelector.UseReadReplica();
        try
        {
            await using var context = await _contextFactory.CreateDbContextAsync();
            return await context.Customers
                .AsNoTracking()
                .Where(c => c.Name.Contains(searchTerm) ||
                            c.Email.Contains(searchTerm))
                .OrderBy(c => c.Name)
                .Skip((page - 1) * pageSize)
                .Take(pageSize)
                .ToListAsync();
        }
        finally
        {
            _databaseSelector.UsePrimary();
        }
    }
}

// Service Registration
public static class ReadWriteServiceExtensions
{
    public static IServiceCollection AddReadWriteDatabase(
        this IServiceCollection services,
        string primaryConnection,
        List<string> replicaConnections)
    {
        services.AddSingleton<IDatabaseSelector>(
            new RoundRobinDatabaseSelector(primaryConnection, replicaConnections));

        services.AddDbContextFactory<ReadWriteDbContext>((sp, options) =>
        {
            var selector = sp.GetRequiredService<IDatabaseSelector>();
            options.UseNpgsql(selector.GetConnectionString());
        });

        services.AddScoped<CustomerReadRepository>();

        return services;
    }
}
```

---

## ขั้นตอนที่ 964: Temporal Tables (SQL Server / PostgreSQL Time Travel)

### Temporal Tables ใน SQL Server

Temporal Tables ช่วยเก็บประวัติการเปลี่ยนแปลงข้อมูลแบบอัตโนมัติ

```csharp
// Employee.cs - Entity สำหรับ Temporal Table
public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Department { get; set; } = string.Empty;
    public decimal Salary { get; set; }
    public string Position { get; set; } = string.Empty;

    // Temporal period columns (จัดการโดย SQL Server)
    public DateTime PeriodStart { get; set; }
    public DateTime PeriodEnd { get; set; }
}

// TemporalDbContext.cs
public class TemporalDbContext : DbContext
{
    public TemporalDbContext(DbContextOptions<TemporalDbContext> options)
        : base(options) { }

    public DbSet<Employee> Employees { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Employee>(entity =>
        {
            entity.ToTable("Employees", tb =>
            {
                // กำหนดให้ใช้ Temporal Table
                tb.IsTemporal(temporal =>
                {
                    temporal.UseHistoryTable("EmployeesHistory");
                    temporal.HasPeriodStart("PeriodStart")
                        .HasColumnName("ValidFrom");
                    temporal.HasPeriodEnd("PeriodEnd")
                        .HasColumnName("ValidTo");
                });
            });
        });
    }
}

// TemporalQueryService.cs - Query ด้วย Time Travel
public class EmployeeTemporalService
{
    private readonly TemporalDbContext _context;

    public EmployeeTemporalService(TemporalDbContext context)
    {
        _context = context;
    }

    // ดูข้อมูล ณ เวลาที่กำหนด (AS OF)
    public async Task<Employee?> GetEmployeeAsOfAsync(int id, DateTime pointInTime)
    {
        return await _context.Employees
            .TemporalAsOf(pointInTime)
            .FirstOrDefaultAsync(e => e.Id == id);
    }

    // ดูประวัติทั้งหมดของ employee
    public async Task<List<Employee>> GetEmployeeHistoryAsync(int id)
    {
        return await _context.Employees
            .TemporalAll()
            .Where(e => e.Id == id)
            .OrderBy(e => e.PeriodStart)
            .ToListAsync();
    }

    // ดูข้อมูลที่ active ในช่วงเวลาที่กำหนด (BETWEEN)
    public async Task<List<Employee>> GetEmployeesFromToAsync(
        DateTime from, DateTime to)
    {
        return await _context.Employees
            .TemporalFromTo(from, to)
            .Where(e => e.Department == "Engineering")
            .ToListAsync();
    }

    // ดูข้อมูลที่ถูก delete ไปแล้ว
    public async Task<List<Employee>> GetDeletedEmployeesAsync()
    {
        var currentIds = await _context.Employees
            .Select(e => e.Id)
            .ToListAsync();

        var allHistoricalIds = await _context.Employees
            .TemporalAll()
            .Select(e => e.Id)
            .Distinct()
            .ToListAsync();

        var deletedIds = allHistoricalIds.Except(currentIds).ToList();

        return await _context.Employees
            .TemporalAll()
            .Where(e => deletedIds.Contains(e.Id))
            .GroupBy(e => e.Id)
            .Select(g => g.OrderByDescending(e => e.PeriodEnd).First())
            .ToListAsync();
    }

    // Restore ข้อมูลที่ถูก delete
    public async Task RestoreEmployeeAsync(int id, DateTime restoreFrom)
    {
        var historicalEmployee = await _context.Employees
            .TemporalAsOf(restoreFrom)
            .FirstOrDefaultAsync(e => e.Id == id);

        if (historicalEmployee == null)
            throw new InvalidOperationException(
                $"Employee {id} not found at {restoreFrom}");

        var restored = new Employee
        {
            Id = historicalEmployee.Id,
            Name = historicalEmployee.Name,
            Department = historicalEmployee.Department,
            Salary = historicalEmployee.Salary,
            Position = historicalEmployee.Position
        };

        _context.Employees.Add(restored);
        await _context.SaveChangesAsync();
    }
}
```

### Temporal Tables ใน PostgreSQL ด้วย Audit History

```sql
-- Migration สำหรับ PostgreSQL temporal pattern
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    department VARCHAR(100) NOT NULL,
    salary DECIMAL(18,2),
    valid_from TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    valid_to TIMESTAMPTZ
);

-- History table
CREATE TABLE employees_history (
    LIKE employees INCLUDING ALL,
    operation CHAR(1) NOT NULL
);

-- Trigger function
CREATE OR REPLACE FUNCTION archive_employee_history()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        INSERT INTO employees_history
        SELECT OLD.*, 'U';
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO employees_history
        SELECT OLD.*, 'D';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER employees_history_trigger
BEFORE UPDATE OR DELETE ON employees
FOR EACH ROW EXECUTE FUNCTION archive_employee_history();
```

---

## ขั้นตอนที่ 965: JSON Columns ใน EF Core (PostgreSQL JSONB)

### การใช้ JSONB ใน PostgreSQL กับ EF Core

```csharp
// Models
public class ProductCatalog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string SKU { get; set; } = string.Empty;

    // JSON column
    public ProductAttributes Attributes { get; set; } = new();
    public List<ProductVariant> Variants { get; set; } = new();
    public Dictionary<string, string> Metadata { get; set; } = new();
}

public class ProductAttributes
{
    public string Color { get; set; } = string.Empty;
    public string Size { get; set; } = string.Empty;
    public double Weight { get; set; }
    public string Material { get; set; } = string.Empty;
    public List<string> Tags { get; set; } = new();
}

public class ProductVariant
{
    public string VariantId { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Stock { get; set; }
}

// JsonDbContext.cs
public class JsonDbContext : DbContext
{
    public JsonDbContext(DbContextOptions<JsonDbContext> options)
        : base(options) { }

    public DbSet<ProductCatalog> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<ProductCatalog>(entity =>
        {
            // Owned JSON entity (EF Core 7+)
            entity.OwnsOne(p => p.Attributes, attrs =>
            {
                attrs.ToJson();
            });

            // Collection ใน JSON
            entity.OwnsMany(p => p.Variants, variants =>
            {
                variants.ToJson();
            });

            // Dictionary เป็น JSON
            entity.Property(p => p.Metadata)
                .HasColumnType("jsonb")
                .HasConversion(
                    v => JsonSerializer.Serialize(v, JsonSerializerOptions.Default),
                    v => JsonSerializer.Deserialize<Dictionary<string, string>>(v,
                        JsonSerializerOptions.Default) ?? new());
        });
    }
}

// JsonQueryService.cs
public class ProductJsonQueryService
{
    private readonly JsonDbContext _context;

    public ProductJsonQueryService(JsonDbContext context)
    {
        _context = context;
    }

    // Query ด้วย JSON property
    public async Task<List<ProductCatalog>> GetByColorAsync(string color)
    {
        return await _context.Products
            .Where(p => p.Attributes.Color == color)
            .ToListAsync();
    }

    // Query กับ JSON array
    public async Task<List<ProductCatalog>> GetByTagAsync(string tag)
    {
        return await _context.Products
            .Where(p => p.Attributes.Tags.Contains(tag))
            .ToListAsync();
    }

    // Query กับ nested JSON
    public async Task<List<ProductCatalog>> GetByPriceRangeAsync(
        decimal minPrice, decimal maxPrice)
    {
        return await _context.Products
            .Where(p => p.Variants.Any(v =>
                v.Price >= minPrice && v.Price <= maxPrice))
            .ToListAsync();
    }

    // Raw SQL สำหรับ JSONB operator ขั้นสูง
    public async Task<List<ProductCatalog>> SearchByMetadataAsync(
        string key, string value)
    {
        return await _context.Products
            .FromSqlRaw(
                "SELECT * FROM \"Products\" WHERE \"Metadata\"->>{0} = {1}",
                key, value)
            .ToListAsync();
    }

    // JSONB containment query
    public async Task<List<ProductCatalog>> GetProductsContainingAsync(
        object jsonFilter)
    {
        var jsonString = JsonSerializer.Serialize(jsonFilter);
        return await _context.Products
            .FromSqlRaw(
                $"SELECT * FROM \"Products\" WHERE \"Attributes\" @> '{jsonString}'::jsonb")
            .ToListAsync();
    }

    // Upsert product ด้วย JSON
    public async Task UpsertProductAsync(ProductCatalog product)
    {
        var existing = await _context.Products
            .FirstOrDefaultAsync(p => p.SKU == product.SKU);

        if (existing == null)
        {
            _context.Products.Add(product);
        }
        else
        {
            existing.Name = product.Name;
            existing.Attributes = product.Attributes;
            existing.Variants = product.Variants;
            existing.Metadata = product.Metadata;
            _context.Products.Update(existing);
        }

        await _context.SaveChangesAsync();
    }
}
```

---

## ขั้นตอนที่ 966: Full-text Search ใน EF Core (PostgreSQL tsvector)

### การ Implement Full-text Search

```csharp
// Article.cs
public class Article
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;
    public string Author { get; set; } = string.Empty;
    public List<string> Tags { get; set; } = new();
    public DateTime PublishedAt { get; set; }

    // tsvector column สำหรับ full-text search
    public NpgsqlTsVector? SearchVector { get; set; }
}

// FullTextDbContext.cs
public class FullTextDbContext : DbContext
{
    public FullTextDbContext(DbContextOptions<FullTextDbContext> options)
        : base(options) { }

    public DbSet<Article> Articles { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Article>(entity =>
        {
            // กำหนด tsvector column
            entity.Property(a => a.SearchVector)
                .HasColumnType("tsvector")
                .HasComputedColumnSql(
                    "to_tsvector('english', coalesce(\"Title\", '') || ' ' || " +
                    "coalesce(\"Content\", ''))",
                    stored: true);

            // GIN index สำหรับ full-text search
            entity.HasIndex(a => a.SearchVector)
                .HasMethod("GIN");

            // Index สำหรับ ranking
            entity.HasIndex(a => a.PublishedAt);
        });
    }
}

// FullTextSearchService.cs
public class ArticleSearchService
{
    private readonly FullTextDbContext _context;

    public ArticleSearchService(FullTextDbContext context)
    {
        _context = context;
    }

    // Basic full-text search
    public async Task<List<ArticleSearchResult>> SearchAsync(
        string searchQuery, int page = 1, int pageSize = 10)
    {
        var tsQuery = EF.Functions.ToTsQuery("english", searchQuery);

        var query = _context.Articles
            .Where(a => a.SearchVector!.Matches(tsQuery))
            .Select(a => new ArticleSearchResult
            {
                Id = a.Id,
                Title = a.Title,
                Author = a.Author,
                PublishedAt = a.PublishedAt,
                Rank = a.SearchVector!.RankCoverDensity(tsQuery),
                Headline = EF.Functions.ToTsHeadline(
                    "english",
                    a.Content,
                    tsQuery,
                    "MaxWords=50, MinWords=25, ShortWord=3, HighlightAll=false")
            })
            .OrderByDescending(r => r.Rank)
            .Skip((page - 1) * pageSize)
            .Take(pageSize);

        return await query.ToListAsync();
    }

    // Phrase search
    public async Task<List<Article>> SearchByPhraseAsync(string phrase)
    {
        var tsQuery = EF.Functions.ToTsQuery(
            "english",
            $"'{phrase}'");

        return await _context.Articles
            .Where(a => a.SearchVector!.Matches(tsQuery))
            .OrderByDescending(a =>
                a.SearchVector!.RankCoverDensity(tsQuery))
            .ToListAsync();
    }

    // Fuzzy search ด้วย trigram
    public async Task<List<Article>> FuzzySearchAsync(string searchTerm)
    {
        return await _context.Articles
            .Where(a =>
                EF.Functions.TrigramsSimilarity(a.Title, searchTerm) > 0.3 ||
                EF.Functions.TrigramsSimilarity(a.Content, searchTerm) > 0.2)
            .OrderByDescending(a =>
                Math.Max(
                    EF.Functions.TrigramsSimilarity(a.Title, searchTerm),
                    EF.Functions.TrigramsSimilarity(a.Content, searchTerm)))
            .Take(20)
            .ToListAsync();
    }
}

public class ArticleSearchResult
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Author { get; set; } = string.Empty;
    public DateTime PublishedAt { get; set; }
    public float Rank { get; set; }
    public string Headline { get; set; } = string.Empty;
}
```

### Migration สำหรับ Full-text Search

```csharp
// FullTextSearchMigration.cs
public partial class AddFullTextSearch : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // เพิ่ม tsvector column
        migrationBuilder.AddColumn<NpgsqlTsVector>(
            name: "SearchVector",
            table: "Articles",
            type: "tsvector",
            nullable: true,
            computedColumnSql:
                "to_tsvector('english', coalesce(\"Title\", '') || ' ' || " +
                "coalesce(\"Content\", ''))",
            stored: true);

        // สร้าง GIN index
        migrationBuilder.Sql(
            "CREATE INDEX \"IX_Articles_SearchVector\" ON \"Articles\" " +
            "USING GIN (\"SearchVector\")");

        // Enable pg_trgm extension สำหรับ fuzzy search
        migrationBuilder.Sql("CREATE EXTENSION IF NOT EXISTS pg_trgm");

        // Index สำหรับ trigram search
        migrationBuilder.Sql(
            "CREATE INDEX \"IX_Articles_Title_Trgm\" ON \"Articles\" " +
            "USING GIN (\"Title\" gin_trgm_ops)");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("DROP INDEX IF EXISTS \"IX_Articles_Title_Trgm\"");
        migrationBuilder.Sql("DROP INDEX IF EXISTS \"IX_Articles_SearchVector\"");
        migrationBuilder.DropColumn("SearchVector", "Articles");
    }
}
```

---

## ขั้นตอนที่ 967: Change Data Capture (CDC) กับ Debezium + PostgreSQL

### แนวคิด CDC

Change Data Capture (CDC) คือการ capture การเปลี่ยนแปลงของข้อมูลใน database แบบ real-time เพื่อส่งต่อไปยังระบบอื่น

```csharp
// Debezium Event Models
public record DebeziumEvent<T>
{
    public DebeziumPayload<T> Payload { get; init; } = new();
    public string Schema { get; init; } = string.Empty;
}

public record DebeziumPayload<T>
{
    public T? Before { get; init; }
    public T? After { get; init; }
    public string Op { get; init; } = string.Empty; // c=create, u=update, d=delete, r=read
    public long TsMs { get; init; }
    public DebeziumSource Source { get; init; } = new();
}

public record DebeziumSource
{
    public string Connector { get; init; } = string.Empty;
    public string Table { get; init; } = string.Empty;
    public string Schema { get; init; } = string.Empty;
    public long TsMs { get; init; }
    public long Lsn { get; init; }
}

// CustomerCdcEvent.cs
public record CustomerCdcEvent
{
    public Guid Id { get; init; }
    public string Name { get; init; } = string.Empty;
    public string Email { get; init; } = string.Empty;
    public DateTime UpdatedAt { get; init; }
}

// CdcConsumerService.cs - Kafka Consumer สำหรับ CDC events
public class CustomerCdcConsumerService : BackgroundService
{
    private readonly IConsumer<string, string> _consumer;
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<CustomerCdcConsumerService> _logger;

    public CustomerCdcConsumerService(
        IConsumer<string, string> consumer,
        IServiceProvider serviceProvider,
        ILogger<CustomerCdcConsumerService> logger)
    {
        _consumer = consumer;
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _consumer.Subscribe("postgres.public.customers");

        _logger.LogInformation("Started consuming CDC events from Debezium");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var consumeResult = _consumer.Consume(
                    TimeSpan.FromMilliseconds(100));

                if (consumeResult?.Message == null) continue;

                await ProcessCdcEventAsync(consumeResult.Message.Value, stoppingToken);

                _consumer.Commit(consumeResult);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing CDC event");
                await Task.Delay(1000, stoppingToken);
            }
        }

        _consumer.Close();
    }

    private async Task ProcessCdcEventAsync(string eventJson, CancellationToken ct)
    {
        var debeziumEvent = JsonSerializer.Deserialize<
            DebeziumEvent<CustomerCdcEvent>>(eventJson);

        if (debeziumEvent == null) return;

        var payload = debeziumEvent.Payload;

        using var scope = _serviceProvider.CreateScope();
        var handler = scope.ServiceProvider
            .GetRequiredService<ICdcEventHandler<CustomerCdcEvent>>();

        switch (payload.Op)
        {
            case "c": // Create
                await handler.HandleCreateAsync(payload.After!, ct);
                break;
            case "u": // Update
                await handler.HandleUpdateAsync(payload.Before!, payload.After!, ct);
                break;
            case "d": // Delete
                await handler.HandleDeleteAsync(payload.Before!, ct);
                break;
            case "r": // Read (snapshot)
                await handler.HandleSnapshotAsync(payload.After!, ct);
                break;
        }

        _logger.LogDebug(
            "Processed CDC event: Op={Op}, Table={Table}",
            payload.Op, payload.Source.Table);
    }
}

// ICdcEventHandler.cs
public interface ICdcEventHandler<T>
{
    Task HandleCreateAsync(T entity, CancellationToken ct = default);
    Task HandleUpdateAsync(T before, T after, CancellationToken ct = default);
    Task HandleDeleteAsync(T entity, CancellationToken ct = default);
    Task HandleSnapshotAsync(T entity, CancellationToken ct = default);
}

// CustomerSearchIndexHandler.cs - Sync to Elasticsearch
public class CustomerSearchIndexHandler : ICdcEventHandler<CustomerCdcEvent>
{
    private readonly IElasticClient _elasticClient;
    private readonly ILogger<CustomerSearchIndexHandler> _logger;

    public CustomerSearchIndexHandler(
        IElasticClient elasticClient,
        ILogger<CustomerSearchIndexHandler> logger)
    {
        _elasticClient = elasticClient;
        _logger = logger;
    }

    public async Task HandleCreateAsync(CustomerCdcEvent entity, CancellationToken ct)
    {
        await _elasticClient.IndexDocumentAsync(entity);
        _logger.LogInformation("Indexed new customer {Id}", entity.Id);
    }

    public async Task HandleUpdateAsync(
        CustomerCdcEvent before, CustomerCdcEvent after, CancellationToken ct)
    {
        await _elasticClient.UpdateAsync<CustomerCdcEvent>(
            after.Id,
            u => u.Doc(after));
        _logger.LogInformation("Updated customer index {Id}", after.Id);
    }

    public async Task HandleDeleteAsync(CustomerCdcEvent entity, CancellationToken ct)
    {
        await _elasticClient.DeleteAsync<CustomerCdcEvent>(entity.Id);
        _logger.LogInformation("Deleted customer index {Id}", entity.Id);
    }

    public async Task HandleSnapshotAsync(CustomerCdcEvent entity, CancellationToken ct)
    {
        await HandleCreateAsync(entity, ct);
    }
}
```

### Debezium Configuration

```json
{
  "name": "postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.dbname": "myapp",
    "database.server.name": "postgres",
    "table.include.list": "public.customers,public.orders",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_slot",
    "publication.name": "debezium_pub",
    "snapshot.mode": "initial",
    "transforms": "route",
    "transforms.route.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter"
  }
}
```

---

## ขั้นตอนที่ 968: Database Testing Strategies

### 1. TestContainers สำหรับ Integration Tests

```csharp
// DatabaseTestFixture.cs
public class PostgreSqlTestFixture : IAsyncLifetime
{
    private PostgreSqlContainer _container = null!;
    public string ConnectionString { get; private set; } = string.Empty;

    public async Task InitializeAsync()
    {
        _container = new PostgreSqlBuilder()
            .WithDatabase("testdb")
            .WithUsername("testuser")
            .WithPassword("testpass")
            .WithImage("postgres:15-alpine")
            .Build();

        await _container.StartAsync();
        ConnectionString = _container.GetConnectionString();

        // Run migrations
        await RunMigrationsAsync();
    }

    private async Task RunMigrationsAsync()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(ConnectionString)
            .Options;

        await using var context = new AppDbContext(options);
        await context.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        await _container.DisposeAsync();
    }
}

// ProductRepositoryTests.cs
public class ProductRepositoryTests : IClassFixture<PostgreSqlTestFixture>
{
    private readonly PostgreSqlTestFixture _fixture;

    public ProductRepositoryTests(PostgreSqlTestFixture fixture)
    {
        _fixture = fixture;
    }

    private AppDbContext CreateContext()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(_fixture.ConnectionString)
            .Options;
        return new AppDbContext(options);
    }

    [Fact]
    public async Task CreateProduct_ShouldPersistToDatabase()
    {
        // Arrange
        await using var context = CreateContext();
        var repository = new ProductRepository(context);

        var product = new Product
        {
            Name = "Test Product",
            Price = 99.99m,
            Stock = 100
        };

        // Act
        var created = await repository.CreateAsync(product);

        // Assert
        await using var verifyContext = CreateContext();
        var found = await verifyContext.Products
            .FirstOrDefaultAsync(p => p.Id == created.Id);

        Assert.NotNull(found);
        Assert.Equal("Test Product", found.Name);
        Assert.Equal(99.99m, found.Price);
    }

    [Fact]
    public async Task SearchProducts_WithFullTextSearch_ShouldReturnResults()
    {
        // Arrange
        await using var context = CreateContext();

        context.Products.AddRange(
            new Product { Name = "Apple iPhone 15", Price = 999m, Stock = 50 },
            new Product { Name = "Samsung Galaxy S23", Price = 899m, Stock = 30 },
            new Product { Name = "Apple MacBook Pro", Price = 2499m, Stock = 20 }
        );
        await context.SaveChangesAsync();

        // Act
        var results = await context.Products
            .Where(p => EF.Functions.Like(p.Name, "%Apple%"))
            .ToListAsync();

        // Assert
        Assert.Equal(2, results.Count);
        Assert.All(results, r => Assert.Contains("Apple", r.Name));
    }
}
```

### 2. In-Memory Database Testing

```csharp
// InMemoryDbTests.cs
public class OrderServiceInMemoryTests
{
    private AppDbContext CreateInMemoryContext(string dbName = "TestDb")
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: dbName)
            .ConfigureWarnings(w =>
                w.Ignore(InMemoryEventId.TransactionIgnoredWarning))
            .Options;

        return new AppDbContext(options);
    }

    [Fact]
    public async Task PlaceOrder_ShouldUpdateStock()
    {
        // Arrange
        var dbName = Guid.NewGuid().ToString();
        await using var seedContext = CreateInMemoryContext(dbName);

        var product = new Product
        {
            Id = 1,
            Name = "Widget",
            Price = 9.99m,
            Stock = 10
        };
        seedContext.Products.Add(product);
        await seedContext.SaveChangesAsync();

        await using var testContext = CreateInMemoryContext(dbName);
        var orderService = new OrderService(testContext);

        // Act
        var order = await orderService.PlaceOrderAsync(new PlaceOrderRequest
        {
            ProductId = 1,
            Quantity = 3,
            CustomerId = Guid.NewGuid()
        });

        // Assert
        await using var verifyContext = CreateInMemoryContext(dbName);
        var updatedProduct = await verifyContext.Products.FindAsync(1);

        Assert.NotNull(updatedProduct);
        Assert.Equal(7, updatedProduct.Stock); // 10 - 3 = 7
        Assert.Equal(OrderStatus.Pending, order.Status);
    }
}

// Respawn สำหรับ clean database ระหว่าง test
public class DatabaseCleanupFixture : IAsyncLifetime
{
    private Respawner _respawner = null!;
    public string ConnectionString { get; private set; } = "your-test-connection-string";

    public async Task InitializeAsync()
    {
        _respawner = await Respawner.CreateAsync(ConnectionString, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            TablesToIgnore = new Table[]
            {
                new Table("__EFMigrationsHistory"),
                new Table("seed_data")
            }
        });
    }

    public async Task ResetDatabaseAsync()
    {
        await _respawner.ResetAsync(ConnectionString);
    }

    public Task DisposeAsync() => Task.CompletedTask;
}
```

### 3. Repository Test ด้วย Real Database Pattern

```csharp
// BaseIntegrationTest.cs
public abstract class BaseIntegrationTest : IAsyncLifetime
{
    private static PostgreSqlContainer _container = null!;
    private static string _connectionString = string.Empty;
    private static bool _initialized = false;
    private static readonly SemaphoreSlim _initLock = new(1, 1);

    protected AppDbContext Context { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        await _initLock.WaitAsync();
        try
        {
            if (!_initialized)
            {
                _container = new PostgreSqlBuilder()
                    .WithImage("postgres:15-alpine")
                    .Build();
                await _container.StartAsync();
                _connectionString = _container.GetConnectionString();

                // Run migrations once
                var options = new DbContextOptionsBuilder<AppDbContext>()
                    .UseNpgsql(_connectionString)
                    .Options;
                await using var ctx = new AppDbContext(options);
                await ctx.Database.MigrateAsync();

                _initialized = true;
            }
        }
        finally
        {
            _initLock.Release();
        }

        Context = new AppDbContext(
            new DbContextOptionsBuilder<AppDbContext>()
                .UseNpgsql(_connectionString)
                .Options);

        // Clean up data before each test
        await CleanupAsync();
    }

    protected virtual async Task CleanupAsync()
    {
        // Override สำหรับ cleanup เฉพาะ test
        await Task.CompletedTask;
    }

    public async Task DisposeAsync()
    {
        await Context.DisposeAsync();
    }
}
```

---

## ขั้นตอนที่ 969: EF Core Interceptors (Audit Log, Soft Delete, Query Hints)

### 1. Audit Log Interceptor

```csharp
// AuditEntry.cs
public class AuditEntry
{
    public int Id { get; set; }
    public string EntityType { get; set; } = string.Empty;
    public string EntityId { get; set; } = string.Empty;
    public string Action { get; set; } = string.Empty; // Created, Updated, Deleted
    public string? OldValues { get; set; }
    public string? NewValues { get; set; }
    public string ChangedBy { get; set; } = string.Empty;
    public DateTime ChangedAt { get; set; } = DateTime.UtcNow;
    public string? IpAddress { get; set; }
}

// AuditInterceptor.cs
public class AuditSaveChangesInterceptor : SaveChangesInterceptor
{
    private readonly ICurrentUserService _currentUser;
    private readonly IHttpContextAccessor _httpContextAccessor;

    public AuditSaveChangesInterceptor(
        ICurrentUserService currentUser,
        IHttpContextAccessor httpContextAccessor)
    {
        _currentUser = currentUser;
        _httpContextAccessor = httpContextAccessor;
    }

    public override async ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        if (eventData.Context == null) return result;

        var auditEntries = CreateAuditEntries(eventData.Context);

        if (auditEntries.Any())
        {
            eventData.Context.Set<AuditEntry>().AddRange(auditEntries);
        }

        return result;
    }

    private List<AuditEntry> CreateAuditEntries(DbContext context)
    {
        var entries = new List<AuditEntry>();
        var ipAddress = _httpContextAccessor.HttpContext?
            .Connection.RemoteIpAddress?.ToString();

        foreach (var entry in context.ChangeTracker.Entries())
        {
            if (entry.Entity is AuditEntry) continue; // ข้าม audit ของ audit
            if (entry.State == EntityState.Detached ||
                entry.State == EntityState.Unchanged) continue;

            var entityType = entry.Entity.GetType().Name;
            var entityId = GetEntityId(entry);
            var action = entry.State switch
            {
                EntityState.Added => "Created",
                EntityState.Modified => "Updated",
                EntityState.Deleted => "Deleted",
                _ => "Unknown"
            };

            string? oldValues = null;
            string? newValues = null;

            if (entry.State == EntityState.Modified)
            {
                var oldValueDict = entry.OriginalValues.Properties
                    .ToDictionary(p => p.Name, p => entry.OriginalValues[p]);
                var newValueDict = entry.CurrentValues.Properties
                    .ToDictionary(p => p.Name, p => entry.CurrentValues[p]);

                oldValues = JsonSerializer.Serialize(oldValueDict);
                newValues = JsonSerializer.Serialize(newValueDict);
            }
            else if (entry.State == EntityState.Added)
            {
                var newValueDict = entry.CurrentValues.Properties
                    .ToDictionary(p => p.Name, p => entry.CurrentValues[p]);
                newValues = JsonSerializer.Serialize(newValueDict);
            }

            entries.Add(new AuditEntry
            {
                EntityType = entityType,
                EntityId = entityId,
                Action = action,
                OldValues = oldValues,
                NewValues = newValues,
                ChangedBy = _currentUser.UserId ?? "system",
                ChangedAt = DateTime.UtcNow,
                IpAddress = ipAddress
            });
        }

        return entries;
    }

    private static string GetEntityId(EntityEntry entry)
    {
        var keyValues = entry.Metadata.FindPrimaryKey()?.Properties
            .Select(p => entry.Property(p.Name).CurrentValue?.ToString())
            .Where(v => v != null)
            .ToArray();

        return keyValues != null ? string.Join(",", keyValues) : "unknown";
    }
}
```

### 2. Soft Delete Interceptor

```csharp
// ISoftDeletable.cs
public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
    string? DeletedBy { get; set; }
}

// SoftDeleteInterceptor.cs
public class SoftDeleteInterceptor : SaveChangesInterceptor
{
    private readonly ICurrentUserService _currentUser;

    public SoftDeleteInterceptor(ICurrentUserService currentUser)
    {
        _currentUser = currentUser;
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        if (eventData.Context == null) return new ValueTask<InterceptionResult<int>>(result);

        ProcessSoftDeletes(eventData.Context);

        return new ValueTask<InterceptionResult<int>>(result);
    }

    private void ProcessSoftDeletes(DbContext context)
    {
        var deletedEntities = context.ChangeTracker
            .Entries<ISoftDeletable>()
            .Where(e => e.State == EntityState.Deleted)
            .ToList();

        foreach (var entry in deletedEntities)
        {
            entry.State = EntityState.Modified;
            entry.Entity.IsDeleted = true;
            entry.Entity.DeletedAt = DateTime.UtcNow;
            entry.Entity.DeletedBy = _currentUser.UserId ?? "system";
        }
    }
}

// SoftDeleteDbContext.cs
public class SoftDeleteDbContext : DbContext
{
    public SoftDeleteDbContext(DbContextOptions options) : base(options) { }

    public DbSet<Customer> Customers { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Global query filter สำหรับ soft delete
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            if (typeof(ISoftDeletable).IsAssignableFrom(entityType.ClrType))
            {
                var method = typeof(SoftDeleteDbContext)
                    .GetMethod(nameof(GetSoftDeleteFilter),
                        BindingFlags.NonPublic | BindingFlags.Static)!
                    .MakeGenericMethod(entityType.ClrType);

                var filter = method.Invoke(null, null);
                modelBuilder.Entity(entityType.ClrType)
                    .HasQueryFilter((LambdaExpression)filter!);
            }
        }
    }

    private static LambdaExpression GetSoftDeleteFilter<TEntity>()
        where TEntity : class, ISoftDeletable
    {
        Expression<Func<TEntity, bool>> filter = e => !e.IsDeleted;
        return filter;
    }
}
```

### 3. Query Hints Interceptor

```csharp
// QueryHintsInterceptor.cs
public class QueryHintsInterceptor : DbCommandInterceptor
{
    private readonly IQueryHintProvider _hintProvider;

    public QueryHintsInterceptor(IQueryHintProvider hintProvider)
    {
        _hintProvider = hintProvider;
    }

    public override ValueTask<DbCommand> CommandCreatedAsync(
        CommandEndEventData eventData,
        DbCommand result,
        CancellationToken cancellationToken = default)
    {
        var hints = _hintProvider.GetCurrentHints();

        if (hints.Any())
        {
            var hintComment = $"/* {string.Join(", ", hints)} */\n";
            result.CommandText = hintComment + result.CommandText;
        }

        return new ValueTask<DbCommand>(result);
    }
}

// IQueryHintProvider.cs
public interface IQueryHintProvider
{
    IEnumerable<string> GetCurrentHints();
    void AddHint(string hint);
    void ClearHints();
}

// AsyncLocalQueryHintProvider.cs
public class AsyncLocalQueryHintProvider : IQueryHintProvider
{
    private static readonly AsyncLocal<List<string>> _hints = new();

    public IEnumerable<string> GetCurrentHints()
    {
        return _hints.Value ?? Enumerable.Empty<string>();
    }

    public void AddHint(string hint)
    {
        _hints.Value ??= new List<string>();
        _hints.Value.Add(hint);
    }

    public void ClearHints()
    {
        _hints.Value?.Clear();
    }
}

// การใช้งาน
public class ReportService
{
    private readonly AppDbContext _context;
    private readonly IQueryHintProvider _hintProvider;

    public ReportService(AppDbContext context, IQueryHintProvider hintProvider)
    {
        _context = context;
        _hintProvider = hintProvider;
    }

    public async Task<List<SalesReport>> GenerateReportAsync(DateTime from, DateTime to)
    {
        // เพิ่ม query hint สำหรับ report query
        _hintProvider.AddHint("NOLOCK");
        _hintProvider.AddHint("INDEX(orders, IX_Orders_CreatedAt)");

        try
        {
            return await _context.Orders
                .Where(o => o.CreatedAt >= from && o.CreatedAt <= to)
                .GroupBy(o => o.CustomerId)
                .Select(g => new SalesReport
                {
                    CustomerId = g.Key,
                    TotalOrders = g.Count(),
                    TotalAmount = g.Sum(o => o.TotalAmount)
                })
                .ToListAsync();
        }
        finally
        {
            _hintProvider.ClearHints();
        }
    }
}
```

### การ Register Interceptors

```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    options.UseNpgsql(connectionString)
        .AddInterceptors(
            sp.GetRequiredService<AuditSaveChangesInterceptor>(),
            sp.GetRequiredService<SoftDeleteInterceptor>(),
            sp.GetRequiredService<QueryHintsInterceptor>()
        );
});

builder.Services.AddScoped<AuditSaveChangesInterceptor>();
builder.Services.AddScoped<SoftDeleteInterceptor>();
builder.Services.AddSingleton<QueryHintsInterceptor>();
builder.Services.AddSingleton<IQueryHintProvider, AsyncLocalQueryHintProvider>();
```

---

## ขั้นตอนที่ 970: Data Migration Strategies (Expand-Contract Pattern, Zero-downtime Migrations)

### Expand-Contract Pattern

Expand-Contract (หรือ Parallel Change) เป็น pattern สำหรับการ migrate ข้อมูลโดยไม่ต้องหยุดระบบ

```
Phase 1: EXPAND - เพิ่ม column ใหม่ (backward compatible)
Phase 2: MIGRATE - ย้ายข้อมูลจาก column เก่าไปยัง column ใหม่
Phase 3: CONTRACT - ลบ column เก่าออก (หลังจาก deploy เสร็จสมบูรณ์)
```

```csharp
// ตัวอย่าง: เปลี่ยน FullName เป็น FirstName + LastName

// Phase 1: Expand Migration
public partial class ExpandAddFirstLastName : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // เพิ่ม column ใหม่ (nullable เพื่อ backward compatibility)
        migrationBuilder.AddColumn<string>(
            name: "FirstName",
            table: "Customers",
            type: "varchar(100)",
            nullable: true);

        migrationBuilder.AddColumn<string>(
            name: "LastName",
            table: "Customers",
            type: "varchar(100)",
            nullable: true);

        // สร้าง index บน column ใหม่
        migrationBuilder.CreateIndex(
            name: "IX_Customers_LastName",
            table: "Customers",
            column: "LastName");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropIndex("IX_Customers_LastName", "Customers");
        migrationBuilder.DropColumn("FirstName", "Customers");
        migrationBuilder.DropColumn("LastName", "Customers");
    }
}

// Phase 2: Migrate Data
public partial class MigrateCustomerNames : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // migrate ข้อมูลจาก FullName เป็น FirstName + LastName
        migrationBuilder.Sql(@"
            UPDATE ""Customers""
            SET
                ""FirstName"" = SPLIT_PART(""FullName"", ' ', 1),
                ""LastName"" = CASE
                    WHEN POSITION(' ' IN ""FullName"") > 0
                    THEN SUBSTRING(""FullName"" FROM POSITION(' ' IN ""FullName"") + 1)
                    ELSE ''
                END
            WHERE ""FirstName"" IS NULL;
        ");

        // ทำให้ NOT NULL หลังจาก migrate
        migrationBuilder.AlterColumn<string>(
            name: "FirstName",
            table: "Customers",
            type: "varchar(100)",
            nullable: false,
            defaultValue: "");

        migrationBuilder.AlterColumn<string>(
            name: "LastName",
            table: "Customers",
            type: "varchar(100)",
            nullable: false,
            defaultValue: "");
    }
}

// Phase 3: Contract Migration (หลังจาก deploy รอบ 2)
public partial class ContractRemoveFullName : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // ลบ column เก่า
        migrationBuilder.DropColumn("FullName", "Customers");
    }
}
```

### Zero-downtime Migration Service

```csharp
// BatchMigrationService.cs
public class BatchMigrationService
{
    private readonly IDbConnectionFactory _connectionFactory;
    private readonly ILogger<BatchMigrationService> _logger;

    private const int DefaultBatchSize = 1000;
    private const int DefaultDelayMs = 50;

    public BatchMigrationService(
        IDbConnectionFactory connectionFactory,
        ILogger<BatchMigrationService> logger)
    {
        _connectionFactory = connectionFactory;
        _logger = logger;
    }

    public async Task MigrateInBatchesAsync(
        string migrationSql,
        string countSql,
        int batchSize = DefaultBatchSize,
        int delayBetweenBatchesMs = DefaultDelayMs,
        CancellationToken ct = default)
    {
        using var connection = _connectionFactory.CreateConnection();
        await ((DbConnection)connection).OpenAsync(ct);

        // นับจำนวน records ที่ต้อง migrate
        var totalCount = await GetCountAsync(connection, countSql);
        _logger.LogInformation(
            "Starting batch migration: {Total} records, batch size: {BatchSize}",
            totalCount, batchSize);

        long processed = 0;

        while (processed < totalCount)
        {
            var batchMigrationSql = $@"
                WITH batch AS (
                    SELECT ctid FROM ({migrationSql.Replace("LIMIT_PLACEHOLDER", "")})
                    LIMIT {batchSize}
                )
                UPDATE ... WHERE ctid IN (SELECT ctid FROM batch)
            ";

            var rowsAffected = await ExecuteNonQueryAsync(connection, batchMigrationSql);

            if (rowsAffected == 0) break;

            processed += rowsAffected;

            var progressPercent = (double)processed / totalCount * 100;
            _logger.LogInformation(
                "Migration progress: {Processed}/{Total} ({Percent:F1}%)",
                processed, totalCount, progressPercent);

            // หน่วงเวลาเพื่อไม่ให้ database overloaded
            if (delayBetweenBatchesMs > 0)
                await Task.Delay(delayBetweenBatchesMs, ct);
        }

        _logger.LogInformation("Migration completed: {Processed} records processed",
            processed);
    }

    private async Task<long> GetCountAsync(IDbConnection connection, string countSql)
    {
        using var cmd = connection.CreateCommand();
        cmd.CommandText = countSql;
        var result = await ((DbCommand)cmd).ExecuteScalarAsync();
        return Convert.ToInt64(result);
    }

    private async Task<int> ExecuteNonQueryAsync(IDbConnection connection, string sql)
    {
        using var cmd = connection.CreateCommand();
        cmd.CommandText = sql;
        cmd.CommandTimeout = 30; // 30 seconds timeout per batch
        return await ((DbCommand)cmd).ExecuteNonQueryAsync();
    }
}

// ZeroDowntimeMigrationRunner.cs
public class ZeroDowntimeMigrationRunner
{
    private readonly AppDbContext _context;
    private readonly BatchMigrationService _batchService;
    private readonly ILogger<ZeroDowntimeMigrationRunner> _logger;

    public ZeroDowntimeMigrationRunner(
        AppDbContext context,
        BatchMigrationService batchService,
        ILogger<ZeroDowntimeMigrationRunner> logger)
    {
        _context = context;
        _batchService = batchService;
        _logger = logger;
    }

    public async Task RunMigrationAsync(MigrationPlan plan, CancellationToken ct = default)
    {
        _logger.LogInformation("Starting migration: {Name}", plan.Name);

        try
        {
            // ตรวจสอบว่า migration ยังไม่เคยรัน
            if (await IsMigrationCompletedAsync(plan.Name))
            {
                _logger.LogInformation("Migration {Name} already completed, skipping",
                    plan.Name);
                return;
            }

            // รัน expand phase
            if (plan.ExpandSql != null)
            {
                _logger.LogInformation("Running expand phase for {Name}", plan.Name);
                await _context.Database.ExecuteSqlRawAsync(plan.ExpandSql, ct);
            }

            // รัน batch migration
            if (plan.DataMigrationSql != null)
            {
                _logger.LogInformation("Running data migration for {Name}", plan.Name);
                await _batchService.MigrateInBatchesAsync(
                    plan.DataMigrationSql,
                    plan.CountSql ?? "SELECT COUNT(*) FROM migration_pending",
                    plan.BatchSize,
                    plan.DelayBetweenBatchesMs,
                    ct);
            }

            // บันทึกว่า migration เสร็จแล้ว
            await RecordMigrationAsync(plan.Name);

            _logger.LogInformation("Migration {Name} completed successfully", plan.Name);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Migration {Name} failed", plan.Name);
            throw;
        }
    }

    private async Task<bool> IsMigrationCompletedAsync(string migrationName)
    {
        return await _context.Database
            .SqlQuery<int>($@"
                SELECT COUNT(1)
                FROM migration_history
                WHERE migration_name = {migrationName}
                AND completed_at IS NOT NULL")
            .FirstOrDefaultAsync() > 0;
    }

    private async Task RecordMigrationAsync(string migrationName)
    {
        await _context.Database.ExecuteSqlRawAsync(
            "INSERT INTO migration_history (migration_name, completed_at) " +
            "VALUES ({0}, {1})",
            migrationName, DateTime.UtcNow);
    }
}

// MigrationPlan.cs
public class MigrationPlan
{
    public string Name { get; set; } = string.Empty;
    public string? ExpandSql { get; set; }
    public string? DataMigrationSql { get; set; }
    public string? CountSql { get; set; }
    public string? ContractSql { get; set; }
    public int BatchSize { get; set; } = 1000;
    public int DelayBetweenBatchesMs { get; set; } = 50;
}

// ตัวอย่างการใช้งาน Migration Plan
public class CustomerNameMigrationPlan
{
    public static MigrationPlan Create() => new MigrationPlan
    {
        Name = "split-customer-full-name-v2",
        ExpandSql = @"
            ALTER TABLE ""Customers""
            ADD COLUMN IF NOT EXISTS ""FirstName"" VARCHAR(100),
            ADD COLUMN IF NOT EXISTS ""LastName"" VARCHAR(100);
        ",
        DataMigrationSql = @"
            UPDATE ""Customers""
            SET
                ""FirstName"" = SPLIT_PART(""FullName"", ' ', 1),
                ""LastName"" = COALESCE(
                    NULLIF(TRIM(SUBSTRING(""FullName"" FROM
                        POSITION(' ' IN ""FullName"") + 1)), ''),
                    'Unknown'
                )
            WHERE ""FirstName"" IS NULL
        ",
        CountSql = @"SELECT COUNT(*) FROM ""Customers"" WHERE ""FirstName"" IS NULL",
        BatchSize = 500,
        DelayBetweenBatchesMs = 100
    };
}
```

### Blue-Green Deployment Pattern กับ EF Core

```csharp
// MigrationHealthCheck.cs
public class MigrationHealthCheck : IHealthCheck
{
    private readonly AppDbContext _context;

    public MigrationHealthCheck(AppDbContext context)
    {
        _context = context;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var pendingMigrations = await _context.Database
                .GetPendingMigrationsAsync(cancellationToken);

            if (pendingMigrations.Any())
            {
                return HealthCheckResult.Degraded(
                    $"Pending migrations: {string.Join(", ", pendingMigrations)}");
            }

            return HealthCheckResult.Healthy("Database is up to date");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Database health check failed", ex);
        }
    }
}

// Startup migration helper
public static class DatabaseMigrationExtensions
{
    public static async Task EnsureMigratedAsync(this WebApplication app)
    {
        using var scope = app.Services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var logger = scope.ServiceProvider
            .GetRequiredService<ILogger<AppDbContext>>();

        try
        {
            var pendingMigrations = await context.Database
                .GetPendingMigrationsAsync();

            var migrations = pendingMigrations.ToList();
            if (migrations.Any())
            {
                logger.LogInformation(
                    "Applying {Count} pending migrations: {Migrations}",
                    migrations.Count,
                    string.Join(", ", migrations));

                await context.Database.MigrateAsync();

                logger.LogInformation("Migrations applied successfully");
            }
            else
            {
                logger.LogInformation("Database is already up to date");
            }
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Failed to apply database migrations");
            throw;
        }
    }
}
```

---

## สรุป Part 97: Database Advanced Patterns

ในส่วนนี้เราได้เรียนรู้ Pattern ขั้นสูงสำหรับการจัดการฐานข้อมูล ดังนี้:

| ขั้นตอน | หัวข้อ | สิ่งที่ได้เรียน |
|--------|--------|----------------|
| 961 | Multi-tenancy | Row-level, Schema-level, Database-level tenancy |
| 962 | Database Sharding | Hash sharding, Range sharding, Cross-shard queries |
| 963 | Read Replicas | Connection routing, Round-robin load balancing |
| 964 | Temporal Tables | SQL Server IsTemporal(), Time travel queries |
| 965 | JSON Columns | JSONB, OwnsOne/OwnsMany, JSON operators |
| 966 | Full-text Search | tsvector, GIN index, Ranking, Fuzzy search |
| 967 | CDC / Debezium | Kafka consumer, Event handlers, Search indexing |
| 968 | Database Testing | TestContainers, In-memory, Respawner |
| 969 | EF Core Interceptors | Audit log, Soft delete, Query hints |
| 970 | Data Migration | Expand-Contract, Batch migration, Zero-downtime |

### Key Takeaways

- **Multi-tenancy**: เลือก pattern ตาม isolation requirement และ cost
- **Sharding**: ระวัง cross-shard query ที่มี performance ต่ำ
- **Read Replicas**: ใช้ `AsNoTracking()` เสมอสำหรับ read-only queries
- **Temporal Tables**: ช่วยด้าน audit trail โดยไม่ต้องเพิ่ม code
- **JSONB**: ยืดหยุ่นสำหรับ schema ที่เปลี่ยนแปลงบ่อย
- **Full-text Search**: GIN index จำเป็นสำหรับ performance ที่ดี
- **CDC**: เหมาะสำหรับ event-driven architecture
- **TestContainers**: ให้ความมั่นใจสูงสุดในการทดสอบ
- **Interceptors**: ลด boilerplate code สำหรับ cross-cutting concerns
- **Expand-Contract**: วิธีที่ปลอดภัยที่สุดสำหรับ production migration

---

## แหล่งข้อมูลเพิ่มเติม

- [EF Core Documentation - Multi-tenancy](https://learn.microsoft.com/en-us/ef/core/miscellaneous/multitenancy)
- [EF Core Temporal Tables](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-6.0/whatsnew#sql-server-temporal-tables)
- [Debezium Documentation](https://debezium.io/documentation/reference/)
- [TestContainers for .NET](https://dotnet.testcontainers.org/)
- [PostgreSQL JSONB](https://www.postgresql.org/docs/current/datatype-json.html)
- [PostgreSQL Full Text Search](https://www.postgresql.org/docs/current/textsearch.html)

---

## การนำทาง

[← Part 96: Performance Optimization Patterns](part96-performance-optimization.md) | [Part 98: Microservices Advanced Patterns →](part98-microservices-advanced.md)
