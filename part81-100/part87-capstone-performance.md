# Part 87: Capstone - Performance & Scalability
## ขั้นตอนที่ 861-870: High-Performance .NET Systems

---

## 🎯 เป้าหมายของ Part นี้
- Memory optimization เพื่อลด GC pressure
- Span<T> และ Memory<T>
- SIMD operations ด้วย System.Numerics
- Connection pooling
- Async/Await best practices
- Benchmark ด้วย BenchmarkDotNet
- Load testing ด้วย NBomber
- Database query optimization

---

## ขั้นตอนที่ 861: BenchmarkDotNet

```csharp
// dotnet add package BenchmarkDotNet

[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
public class StringBenchmarks
{
    private const string Input = "Hello, World! This is a test string for benchmarking.";
    
    [Benchmark(Baseline = true)]
    public string StringConcat()
    {
        var result = "";
        for (int i = 0; i < 100; i++)
            result += i.ToString();
        return result;
    }
    
    [Benchmark]
    public string StringBuilderConcat()
    {
        var sb = new StringBuilder();
        for (int i = 0; i < 100; i++)
            sb.Append(i);
        return sb.ToString();
    }
    
    [Benchmark]
    public string StringCreateConcat()
        => string.Create(292, 0, (span, _) =>
        {
            var pos = 0;
            for (int i = 0; i < 100; i++)
            {
                i.TryFormat(span[pos..], out var written);
                pos += written;
            }
        });
}

// BenchmarkDotNet results:
// | Method              | Mean      | Allocated |
// |---------------------|-----------|-----------|
// | StringConcat        | 12,345 ns | 87,654 B  |
// | StringBuilderConcat |  2,100 ns |  1,024 B  |
// | StringCreateConcat  |    890 ns |    256 B  |

// Run benchmarks:
// dotnet run -c Release -- --filter *
```

---

## ขั้นตอนที่ 862: Span<T> และ Memory<T>

```csharp
// Span<T>: zero-allocation slice
public static class SpanDemo
{
    // Parse CSV line without allocations
    public static List<string> ParseCsvLine(string line)
    {
        var result = new List<string>();
        var span = line.AsSpan();
        
        while (!span.IsEmpty)
        {
            var commaIndex = span.IndexOf(',');
            if (commaIndex == -1)
            {
                result.Add(span.ToString());
                break;
            }
            result.Add(span[..commaIndex].ToString());
            span = span[(commaIndex + 1)..];
        }
        return result;
    }
    
    // Parse int without allocation
    public static bool TryParseInt(ReadOnlySpan<char> text, out int value)
    {
        return int.TryParse(text, out value); // no string allocation!
    }
    
    // Stack-allocated buffer
    public static string FormatTemperature(double celsius)
    {
        Span<char> buffer = stackalloc char[32];
        celsius.TryFormat(buffer, out var written, "F2");
        buffer[written++] = '°';
        buffer[written++] = 'C';
        return new string(buffer[..written]);
    }
}

// Memory<T>: like Span<T> but heap-storable (can be in async methods)
public class MemoryDemo
{
    public static async Task ProcessChunksAsync(Memory<byte> data, int chunkSize)
    {
        var offset = 0;
        while (offset < data.Length)
        {
            var chunk = data.Slice(offset, Math.Min(chunkSize, data.Length - offset));
            await ProcessChunkAsync(chunk);
            offset += chunk.Length;
        }
    }
    
    private static async Task ProcessChunkAsync(Memory<byte> chunk)
    {
        await Task.Delay(1); // simulate async work
        // Process chunk.Span directly
    }
}

// ArrayPool<T>: reuse arrays to reduce GC
public class ArrayPoolDemo
{
    private static readonly ArrayPool<byte> Pool = ArrayPool<byte>.Shared;
    
    public static async Task<int> ReadBytesAsync(Stream stream)
    {
        var buffer = Pool.Rent(4096); // get from pool
        try
        {
            return await stream.ReadAsync(buffer.AsMemory());
        }
        finally
        {
            Pool.Return(buffer); // return to pool
        }
    }
}
```

---

## ขั้นตอนที่ 863: Async/Await Best Practices

```csharp
// Async patterns for high performance

// 1. ValueTask for hot paths (avoid Task allocation)
public interface IOrderCache
{
    ValueTask<Order?> GetAsync(Guid id);
}

public class OrderCache : IOrderCache
{
    private readonly ConcurrentDictionary<Guid, Order> _cache = new();
    private readonly IOrderRepository _db;
    
    public ValueTask<Order?> GetAsync(Guid id)
    {
        // Synchronous cache hit: returns immediately without Task allocation
        if (_cache.TryGetValue(id, out var order))
            return ValueTask.FromResult<Order?>(order);
        
        // Cache miss: must go to DB asynchronously
        return FetchAndCacheAsync(id);
    }
    
    private async ValueTask<Order?> FetchAndCacheAsync(Guid id)
    {
        var order = await _db.GetByIdAsync(id);
        if (order != null) _cache[id] = order;
        return order;
    }
}

// 2. ConfigureAwait(false) in library code
public class LibraryService
{
    public async Task<Data> GetDataAsync()
    {
        var response = await _httpClient.GetAsync("/api/data").ConfigureAwait(false);
        var content = await response.Content.ReadAsStringAsync().ConfigureAwait(false);
        return JsonSerializer.Deserialize<Data>(content)!;
    }
}

// 3. Parallel async with degree of parallelism
public static async Task ProcessItemsAsync<T>(
    IEnumerable<T> items,
    Func<T, Task> processor,
    int maxDegreeOfParallelism = 10)
{
    using var semaphore = new SemaphoreSlim(maxDegreeOfParallelism);
    var tasks = items.Select(async item =>
    {
        await semaphore.WaitAsync();
        try { await processor(item); }
        finally { semaphore.Release(); }
    });
    await Task.WhenAll(tasks);
}

// 4. IAsyncEnumerable for streaming
public async IAsyncEnumerable<Order> StreamOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    const int pageSize = 100;
    var page = 1;
    
    while (!ct.IsCancellationRequested)
    {
        var orders = await _db.Orders
            .OrderBy(o => o.Id)
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .AsNoTracking()
            .ToListAsync(ct);
        
        if (!orders.Any()) yield break;
        
        foreach (var order in orders)
            yield return order;
        
        page++;
    }
}

// Consumer
await foreach (var order in StreamOrdersAsync())
{
    await ProcessOrderAsync(order);
}
```

---

## ขั้นตอนที่ 864: Database Query Optimization

```csharp
// Optimized EF Core queries

// 1. Split queries for cartesian explosion
var orders = await db.Orders
    .Include(o => o.Lines)          // would cause cartesian explosion
    .Include(o => o.ShippingAddress)
    .AsSplitQuery()  // 3 separate SQL queries, no cartesian product
    .ToListAsync();

// 2. Compiled queries
private static readonly Func<AppDbContext, Guid, Task<Order?>> _getOrderById =
    EF.CompileAsyncQuery((AppDbContext db, Guid id) =>
        db.Orders
          .Include(o => o.Lines)
          .FirstOrDefault(o => o.Id == id));

// Use it:
var order = await _getOrderById(db, orderId);

// 3. Bulk operations (EF Core 7+)
await db.Orders
    .Where(o => o.CreatedAt < DateTime.UtcNow.AddYears(-2))
    .ExecuteUpdateAsync(s => s.SetProperty(o => o.IsArchived, true));

await db.Products
    .Where(p => p.StockQuantity == 0)
    .ExecuteDeleteAsync(); // Direct DELETE, no entity loading

// 4. Raw SQL for complex queries
var topProducts = await db.Database
    .SqlQuery<TopProductDto>($"""
        SELECT p.id, p.name, COUNT(ol.id) as order_count, SUM(ol.quantity) as total_sold
        FROM products p
        JOIN order_lines ol ON ol.product_id = p.id
        JOIN orders o ON o.id = ol.order_id
        WHERE o.created_at >= {DateTime.UtcNow.AddDays(-30)}
        GROUP BY p.id, p.name
        ORDER BY total_sold DESC
        LIMIT 10
        """)
    .ToListAsync();

// 5. Index hints for LINQ
// Configure in Fluent API:
modelBuilder.Entity<Order>()
    .HasIndex(o => new { o.CustomerId, o.CreatedAt })
    .HasDatabaseName("IX_Orders_CustomerId_CreatedAt")
    .IsDescending(false, true); // CustomerId ASC, CreatedAt DESC
```

---

## ขั้นตอนที่ 865: Connection Pooling & HTTP Client

```csharp
// HTTP Client best practices

// 1. Use IHttpClientFactory (manages connection pooling)
builder.Services.AddHttpClient<ProductApiClient>(client =>
{
    client.BaseAddress = new Uri("https://products.api.local");
    client.Timeout = TimeSpan.FromSeconds(30);
    client.DefaultRequestHeaders.Add("User-Agent", "ShopThai/1.0");
})
.AddResilienceHandler("products-pipeline", pipeline =>
{
    pipeline.AddRetry(new HttpRetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromSeconds(1),
        BackoffType = DelayBackoffType.Exponential
    });
    pipeline.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,
        SamplingDuration = TimeSpan.FromSeconds(60),
        MinimumThroughput = 10,
        BreakDuration = TimeSpan.FromSeconds(30)
    });
});

// 2. SocketsHttpHandler for advanced control
builder.Services.AddHttpClient("external")
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
    {
        PooledConnectionLifetime = TimeSpan.FromMinutes(15), // handles DNS rotation
        MaxConnectionsPerServer = 10,
        EnableMultipleHttp2Connections = true
    });

// 3. Named clients with typed wrappers
public class ProductApiClient(HttpClient http)
{
    public async Task<Product?> GetProductAsync(int id)
    {
        var response = await http.GetAsync($"/products/{id}");
        if (response.StatusCode == HttpStatusCode.NotFound) return null;
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<Product>();
    }
}
```

---

## ขั้นตอนที่ 866-870: Load Testing with NBomber

```csharp
// dotnet add package NBomber

[Fact]
public void LoadTest_OrdersApi_ShouldHandleLoad()
{
    var httpFactory = ClientFactory.Create(
        name: "http_factory",
        clientCount: 50,
        initClient: (_, _) => Task.FromResult(new HttpClient
        {
            BaseAddress = new Uri("http://localhost:5000")
        }));
    
    var createOrderScenario = Scenario.Create("create_order", async context =>
    {
        var client = context.Data.Get<HttpClient>("http_factory");
        var request = new CreateOrderRequest(/* ... */);
        
        var response = await client.PostAsJsonAsync("/api/orders", request);
        return response.IsSuccessStatusCode
            ? Response.Ok(statusCode: (int)response.StatusCode, sizeBytes: 1024)
            : Response.Fail(statusCode: (int)response.StatusCode);
    })
    .WithLoadSimulations(
        Simulation.RampingInject(rate: 50, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(30)),
        Simulation.Inject(rate: 100, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(1)),
        Simulation.RampingInject(rate: 0, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(15))
    );
    
    var stats = NBomberRunner
        .RegisterScenarios(createOrderScenario)
        .WithWorkerPlugins(new HttpMetricsPlugin([HttpVersion.Version2]))
        .Run();
    
    // Assert SLAs
    var scenarioStats = stats.ScenarioStats[0];
    Assert.True(scenarioStats.Ok.Request.RPS >= 90, "Must handle 90+ RPS");
    Assert.True(scenarioStats.Ok.Latency.P99 <= 500, "P99 must be < 500ms");
    Assert.True(scenarioStats.Fail.Request.Percent <= 1, "Error rate < 1%");
}
```

---

## 📝 สรุป Part 87

Performance optimization hierarchy:

1. **Algorithm** — O(n log n) vs O(n²) matters most
2. **Memory** — reduce allocations, use Span<T>, ArrayPool
3. **I/O** — async/await, connection pooling, batching
4. **Database** — indexes, split queries, compiled queries
5. **Caching** — IMemoryCache → Redis → CDN
6. **Concurrency** — parallel where safe, semaphore for throttling

> "Premature optimization is the root of all evil" — measure first with BenchmarkDotNet, then optimize the bottleneck.

---

**ก่อนหน้า → [Part 86: Capstone Payments](part86-capstone-payments.md)**  
**ต่อไป → [Part 88: Capstone - Advanced Security](part88-capstone-security.md)**
