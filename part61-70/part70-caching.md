# Part 70: Caching Strategies
## ขั้นตอนที่ 691-700: Caching ใน .NET

---

## 🎯 เป้าหมายของ Part นี้
- IMemoryCache - in-process caching
- IDistributedCache - distributed caching
- Redis caching patterns
- Cache-aside, Write-through, Write-behind
- Cache invalidation strategies
- Response caching (HTTP)
- Output caching (.NET 7+)
- Hybrid cache (.NET 9)

---

## ขั้นตอนที่ 691: IMemoryCache

```csharp
// In-memory cache: เร็วที่สุด แต่ใช้ได้เฉพาะ single-instance
// Install: built-in ใน ASP.NET Core

// Program.cs
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024; // limit total cache size
    options.CompactionPercentage = 0.25; // remove 25% when full
});

// Service using IMemoryCache
public class ProductService
{
    private readonly IMemoryCache _cache;
    private readonly IProductRepository _repo;
    
    public ProductService(IMemoryCache cache, IProductRepository repo)
    {
        _cache = cache;
        _repo = repo;
    }
    
    // Cache-aside pattern
    public async Task<Product?> GetByIdAsync(int id)
    {
        var key = $"product:{id}";
        
        if (_cache.TryGetValue(key, out Product? cached))
            return cached;
        
        var product = await _repo.GetByIdAsync(id);
        if (product == null) return null;
        
        var options = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30),
            SlidingExpiration = TimeSpan.FromMinutes(5),
            Size = 1,
            Priority = CacheItemPriority.Normal
        };
        
        // Register callback on eviction
        options.RegisterPostEvictionCallback((k, v, reason, state) =>
        {
            Console.WriteLine($"Cache evicted: {k}, reason: {reason}");
        });
        
        _cache.Set(key, product, options);
        return product;
    }
    
    // GetOrCreate helper
    public async Task<IEnumerable<Product>> GetByCategoryAsync(string category)
    {
        return await _cache.GetOrCreateAsync($"products:category:{category}", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15);
            entry.Size = 10;
            return await _repo.GetByCategoryAsync(category);
        }) ?? [];
    }
    
    public void Invalidate(int productId)
    {
        _cache.Remove($"product:{productId}");
    }
}
```

---

## ขั้นตอนที่ 692: IDistributedCache + Redis

```csharp
// Distributed cache: ใช้ได้กับ multi-instance deployment
// Install:
// dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis

// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "myapp:"; // key prefix
});

// Generic distributed cache service
public class DistributedCacheService
{
    private readonly IDistributedCache _cache;
    private static readonly JsonSerializerOptions JsonOpts = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase
    };
    
    public DistributedCacheService(IDistributedCache cache) => _cache = cache;
    
    public async Task<T?> GetAsync<T>(string key, CancellationToken ct = default) where T : class
    {
        var bytes = await _cache.GetAsync(key, ct);
        if (bytes == null) return null;
        return JsonSerializer.Deserialize<T>(bytes, JsonOpts);
    }
    
    public async Task SetAsync<T>(string key, T value, TimeSpan? absoluteExpiry = null, 
        TimeSpan? slidingExpiry = null, CancellationToken ct = default) where T : class
    {
        var bytes = JsonSerializer.SerializeToUtf8Bytes(value, JsonOpts);
        var options = new DistributedCacheEntryOptions();
        
        if (absoluteExpiry.HasValue)
            options.AbsoluteExpirationRelativeToNow = absoluteExpiry;
        if (slidingExpiry.HasValue)
            options.SlidingExpiration = slidingExpiry;
        
        await _cache.SetAsync(key, bytes, options, ct);
    }
    
    public async Task<T> GetOrSetAsync<T>(string key, Func<Task<T>> factory,
        TimeSpan? expiry = null, CancellationToken ct = default) where T : class
    {
        var cached = await GetAsync<T>(key, ct);
        if (cached != null) return cached;
        
        var value = await factory();
        await SetAsync(key, value, expiry ?? TimeSpan.FromMinutes(30), ct: ct);
        return value;
    }
    
    public async Task RemoveAsync(string key, CancellationToken ct = default)
        => await _cache.RemoveAsync(key, ct);
    
    // Remove by pattern (Redis only)
    public async Task RemoveByPrefixAsync(string prefix)
    {
        if (_cache is not Microsoft.Extensions.Caching.StackExchangeRedis.RedisCache)
            return;
        // Use StackExchange.Redis directly for pattern delete
    }
}
```

---

## ขั้นตอนที่ 693: Redis Advanced Patterns

```csharp
// Direct Redis access with StackExchange.Redis
// dotnet add package StackExchange.Redis

// Redis connection setup
builder.Services.AddSingleton<IConnectionMultiplexer>(_ =>
    ConnectionMultiplexer.Connect(builder.Configuration.GetConnectionString("Redis")!));

builder.Services.AddSingleton<IDatabase>(sp =>
    sp.GetRequiredService<IConnectionMultiplexer>().GetDatabase());

// Redis cache with advanced features
public class RedisAdvancedCache
{
    private readonly IDatabase _db;
    
    public RedisAdvancedCache(IDatabase db) => _db = db;
    
    // Atomic increment (counter, rate limiting)
    public async Task<long> IncrementAsync(string key, TimeSpan? expiry = null)
    {
        var result = await _db.StringIncrementAsync(key);
        if (expiry.HasValue && result == 1) // first increment - set expiry
            await _db.KeyExpireAsync(key, expiry);
        return result;
    }
    
    // Distributed lock
    public async Task<bool> AcquireLockAsync(string lockKey, string lockValue, TimeSpan timeout)
        => await _db.StringSetAsync(lockKey, lockValue, timeout, When.NotExists);
    
    public async Task ReleaseLockAsync(string lockKey, string lockValue)
    {
        // Lua script for atomic check-and-delete
        var script = @"
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end";
        await _db.ScriptEvaluateAsync(script, new RedisKey[] { lockKey }, new RedisValue[] { lockValue });
    }
    
    // Pub/Sub for cache invalidation
    public async Task PublishInvalidationAsync(string pattern)
    {
        var pub = _db.Multiplexer.GetSubscriber();
        await pub.PublishAsync(RedisChannel.Literal("cache:invalidate"), pattern);
    }
    
    public void SubscribeToInvalidation(Action<string> onInvalidate)
    {
        var sub = _db.Multiplexer.GetSubscriber();
        sub.Subscribe(RedisChannel.Pattern("cache:invalidate"), (_, message) =>
        {
            onInvalidate(message!);
        });
    }
    
    // Sorted set for leaderboard
    public async Task UpdateLeaderboardAsync(string leaderboardKey, string userId, double score)
        => await _db.SortedSetAddAsync(leaderboardKey, userId, score);
    
    public async Task<IEnumerable<(string UserId, double Score)>> GetTopAsync(string key, int count)
    {
        var entries = await _db.SortedSetRangeByRankWithScoresAsync(key, 0, count - 1, Order.Descending);
        return entries.Select(e => ((string)e.Element!, e.Score));
    }
    
    // Hash for user sessions
    public async Task SetSessionAsync(string sessionId, Dictionary<string, string> data, TimeSpan expiry)
    {
        var hashFields = data.Select(kv => new HashEntry(kv.Key, kv.Value)).ToArray();
        var key = $"session:{sessionId}";
        await _db.HashSetAsync(key, hashFields);
        await _db.KeyExpireAsync(key, expiry);
    }
}
```

---

## ขั้นตอนที่ 694: Cache Invalidation Patterns

```csharp
// Tag-based invalidation using Redis Sets
public class TaggedCacheService
{
    private readonly IDatabase _db;
    private readonly DistributedCacheService _cache;
    
    public TaggedCacheService(IDatabase db, DistributedCacheService cache)
    { _db = db; _cache = cache; }
    
    // Set cache with tags
    public async Task SetWithTagsAsync<T>(string key, T value, string[] tags, 
        TimeSpan? expiry = null) where T : class
    {
        await _cache.SetAsync(key, value, expiry);
        
        // Store key -> tags relationship
        foreach (var tag in tags)
        {
            await _db.SetAddAsync($"tag:{tag}", key);
            if (expiry.HasValue)
                await _db.KeyExpireAsync($"tag:{tag}", expiry.Value.Add(TimeSpan.FromMinutes(5)));
        }
    }
    
    // Invalidate all keys with a tag
    public async Task InvalidateTagAsync(string tag)
    {
        var tagKey = $"tag:{tag}";
        var keys = await _db.SetMembersAsync(tagKey);
        
        var tasks = keys.Select(k => _cache.RemoveAsync(k!));
        await Task.WhenAll(tasks);
        
        await _db.KeyDeleteAsync(tagKey);
    }
    
    // Example: product cache with category tags
    public async Task<Product?> GetProductAsync(int id)
    {
        return await _cache.GetOrSetAsync(
            $"product:{id}",
            async () =>
            {
                // fetch from DB...
                var product = new Product { Id = id, Name = "Test", Category = "Electronics" };
                
                // Tag with category for group invalidation
                await SetWithTagsAsync(
                    $"product:{id}",
                    product,
                    new[] { $"category:{product.Category}", "products:all" },
                    TimeSpan.FromMinutes(60)
                );
                return product;
            }
        );
    }
    
    // When category changes, invalidate all products in it
    public async Task OnCategoryUpdatedAsync(string category)
        => await InvalidateTagAsync($"category:{category}");
}

// Cache version key pattern
public class VersionedCache
{
    private readonly IDatabase _db;
    
    private async Task<long> GetVersionAsync(string entityType)
    {
        var version = await _db.StringGetAsync($"version:{entityType}");
        return version.IsNull ? 1 : (long)version;
    }
    
    private async Task IncrementVersionAsync(string entityType)
        => await _db.StringIncrementAsync($"version:{entityType}");
    
    public async Task<string> BuildKeyAsync(string entityType, string id)
    {
        var version = await GetVersionAsync(entityType);
        return $"{entityType}:v{version}:{id}";
    }
    
    // Invalidate all by bumping version - O(1) operation!
    public async Task InvalidateAllAsync(string entityType)
        => await IncrementVersionAsync(entityType);
}
```

---

## ขั้นตอนที่ 695: Response Caching & Output Caching

```csharp
// HTTP Response Caching (cache at HTTP layer)
builder.Services.AddResponseCaching();

// Apply in pipeline
app.UseResponseCaching();

// Controller action
[HttpGet("{id}")]
[ResponseCache(Duration = 60, VaryByQueryKeys = new[] { "id" })]
public async Task<IActionResult> GetProduct(int id)
{
    var product = await _productService.GetByIdAsync(id);
    return product == null ? NotFound() : Ok(product);
}

// Cache-Control headers
[ResponseCache(Duration = 300, Location = ResponseCacheLocation.Any, VaryByHeader = "Accept-Language")]
[HttpGet("list")]
public async Task<IActionResult> GetList()
{
    return Ok(await _productService.GetAllAsync());
}

// Output Caching (.NET 7+) - more flexible
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(b => b.Expire(TimeSpan.FromSeconds(10)));
    
    options.AddPolicy("ByProductId", b => b
        .Expire(TimeSpan.FromMinutes(5))
        .SetVaryByRouteValue("id")
        .Tag("products"));
    
    options.AddPolicy("PublicLong", b => b
        .Expire(TimeSpan.FromHours(1))
        .SetVaryByHeader("Accept-Language"));
});

app.UseOutputCache();

// Endpoint with output cache policy
app.MapGet("/api/products/{id}", async (int id, IProductService svc) =>
{
    var product = await svc.GetByIdAsync(id);
    return product == null ? Results.NotFound() : Results.Ok(product);
})
.CacheOutput("ByProductId");

// Programmatic invalidation
app.MapPost("/api/products/{id}", async (int id, Product updated, 
    IOutputCacheStore store, IProductService svc, CancellationToken ct) =>
{
    await svc.UpdateAsync(updated);
    await store.EvictByTagAsync("products", ct); // invalidate all product cache
    return Results.Ok();
});
```

---

## ขั้นตอนที่ 696: Cache Stampede Prevention

```csharp
// Cache stampede (thundering herd): many requests hit DB simultaneously when cache expires
// Solution: probabilistic early expiration + locking

public class StampedeProtectedCache
{
    private readonly IDistributedCache _cache;
    private readonly SemaphoreSlim _lock = new(1, 1);
    private readonly Random _rng = new();
    
    // Probabilistic Early Recomputation (PER)
    public async Task<T?> GetWithPERAsync<T>(
        string key, 
        Func<Task<T>> factory, 
        TimeSpan ttl,
        double beta = 1.0) where T : class
    {
        var cached = await GetInternalAsync<CacheEntry<T>>(key);
        
        if (cached != null)
        {
            // PER: recompute early with probability based on remaining TTL
            var remaining = cached.ExpiresAt - DateTime.UtcNow;
            var shouldRecompute = -beta * Math.Log(_rng.NextDouble()) > remaining.TotalSeconds;
            
            if (!shouldRecompute)
                return cached.Value;
        }
        
        // Double-checked locking
        await _lock.WaitAsync();
        try
        {
            // Check again after acquiring lock
            var recheck = await GetInternalAsync<CacheEntry<T>>(key);
            if (recheck != null)
            {
                var remaining = recheck.ExpiresAt - DateTime.UtcNow;
                if (remaining.TotalSeconds > 0)
                    return recheck.Value;
            }
            
            var value = await factory();
            var entry = new CacheEntry<T>(value, DateTime.UtcNow.Add(ttl));
            await SetInternalAsync(key, entry, ttl);
            return value;
        }
        finally
        {
            _lock.Release();
        }
    }
    
    private async Task<T?> GetInternalAsync<T>(string key) where T : class
    {
        var bytes = await _cache.GetAsync(key);
        return bytes == null ? null : JsonSerializer.Deserialize<T>(bytes);
    }
    
    private async Task SetInternalAsync<T>(string key, T value, TimeSpan ttl)
    {
        var bytes = JsonSerializer.SerializeToUtf8Bytes(value);
        await _cache.SetAsync(key, bytes, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = ttl
        });
    }
}

private record CacheEntry<T>(T Value, DateTime ExpiresAt);
```

---

## ขั้นตอนที่ 697: Hybrid Cache (.NET 9)

```csharp
// HybridCache: combines IMemoryCache + IDistributedCache in one API
// dotnet add package Microsoft.Extensions.Caching.Hybrid (preview in .NET 9)

builder.Services.AddHybridCache(options =>
{
    options.MaximumPayloadBytes = 1024 * 1024; // 1MB max
    options.MaximumKeyLength = 1024;
    options.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(5),
        LocalCacheExpiration = TimeSpan.FromMinutes(1)
    };
});

public class HybridProductService
{
    private readonly HybridCache _cache;
    private readonly IProductRepository _repo;
    
    public HybridProductService(HybridCache cache, IProductRepository repo)
    { _cache = cache; _repo = repo; }
    
    public async Task<Product?> GetAsync(int id, CancellationToken ct = default)
    {
        return await _cache.GetOrCreateAsync(
            $"product:{id}",
            async (cancel) => await _repo.GetByIdAsync(id, cancel),
            cancellationToken: ct
        );
    }
    
    // Invalidate
    public async Task InvalidateAsync(int id)
        => await _cache.RemoveAsync($"product:{id}");
    
    // Invalidate by tag
    public async Task InvalidateCategoryAsync(string category)
        => await _cache.RemoveByTagAsync($"category:{category}");
    
    // With custom options per call
    public async Task<IEnumerable<Product>> GetFeaturedAsync(CancellationToken ct = default)
    {
        return await _cache.GetOrCreateAsync(
            "products:featured",
            async cancel => await _repo.GetFeaturedAsync(cancel),
            new HybridCacheEntryOptions
            {
                Expiration = TimeSpan.FromHours(1),
                LocalCacheExpiration = TimeSpan.FromMinutes(10)
            },
            tags: new[] { "products:all", "products:featured" },
            cancellationToken: ct
        ) ?? [];
    }
}
```

---

## ขั้นตอนที่ 698: Cache Monitoring & Metrics

```csharp
// Monitor cache health and hit ratio
public class CacheMetrics : IDisposable
{
    private readonly IMemoryCache _cache;
    private long _hits;
    private long _misses;
    
    // .NET 8: MemoryCache exposes stats
    public MemoryCacheStatistics? GetStats() => _cache.GetCurrentStatistics();
    
    public double HitRatio => _hits + _misses == 0 ? 0 : (double)_hits / (_hits + _misses);
    
    public void RecordHit() => Interlocked.Increment(ref _hits);
    public void RecordMiss() => Interlocked.Increment(ref _misses);
    
    public void Dispose() { }
}

// Custom cache decorator with metrics
public class MetricsMemoryCache : IMemoryCache
{
    private readonly IMemoryCache _inner;
    private readonly IMeterFactory _meterFactory;
    private readonly Counter<long> _hitCounter;
    private readonly Counter<long> _missCounter;
    
    public MetricsMemoryCache(IMemoryCache inner, IMeterFactory meterFactory)
    {
        _inner = inner;
        _meterFactory = meterFactory;
        var meter = meterFactory.Create("App.Cache");
        _hitCounter = meter.CreateCounter<long>("cache.hits");
        _missCounter = meter.CreateCounter<long>("cache.misses");
    }
    
    public bool TryGetValue(object key, out object? value)
    {
        if (_inner.TryGetValue(key, out value))
        {
            _hitCounter.Add(1, new TagList { { "key_prefix", GetPrefix(key) } });
            return true;
        }
        _missCounter.Add(1, new TagList { { "key_prefix", GetPrefix(key) } });
        return false;
    }
    
    private static string GetPrefix(object key)
    {
        var keyStr = key.ToString() ?? "";
        var colon = keyStr.IndexOf(':');
        return colon > 0 ? keyStr[..colon] : keyStr;
    }
    
    public ICacheEntry CreateEntry(object key) => _inner.CreateEntry(key);
    public void Remove(object key) => _inner.Remove(key);
    public void Dispose() => _inner.Dispose();
}
```

---

## ขั้นตอนที่ 699: Multi-Level Caching

```csharp
// L1: In-memory (nanoseconds)
// L2: Redis distributed (milliseconds)
// L3: Database (milliseconds-seconds)
public class MultiLevelCache
{
    private readonly IMemoryCache _l1;
    private readonly DistributedCacheService _l2;
    private readonly IProductRepository _l3;
    
    private static readonly TimeSpan L1Ttl = TimeSpan.FromSeconds(30);
    private static readonly TimeSpan L2Ttl = TimeSpan.FromMinutes(10);
    
    public MultiLevelCache(IMemoryCache l1, DistributedCacheService l2, IProductRepository l3)
    { _l1 = l1; _l2 = l2; _l3 = l3; }
    
    public async Task<Product?> GetProductAsync(int id, CancellationToken ct = default)
    {
        var key = $"product:{id}";
        
        // L1: memory cache
        if (_l1.TryGetValue(key, out Product? product))
            return product;
        
        // L2: Redis
        product = await _l2.GetAsync<Product>(key, ct);
        if (product != null)
        {
            _l1.Set(key, product, L1Ttl);
            return product;
        }
        
        // L3: Database
        product = await _l3.GetByIdAsync(id, ct);
        if (product == null) return null;
        
        // Populate both caches
        await _l2.SetAsync(key, product, L2Ttl, ct: ct);
        _l1.Set(key, product, L1Ttl);
        
        return product;
    }
    
    public async Task InvalidateAsync(int id)
    {
        var key = $"product:{id}";
        _l1.Remove(key);
        await _l2.RemoveAsync(key);
    }
}
```

---

## ขั้นตอนที่ 700: Caching Best Practices สรุป

```csharp
// Anti-patterns to avoid

// ❌ Cache entire DbContext result (mutable)
// _cache.Set("all", dbContext.Products.ToList());

// ✅ Cache DTO/read-only projections only
var products = await db.Products
    .Where(p => p.IsActive)
    .Select(p => new ProductDto(p.Id, p.Name, p.Price))
    .ToListAsync();
_cache.Set("products:active", products, TimeSpan.FromMinutes(5));

// ❌ Cache without expiry (memory leak)
// _cache.Set("key", value);

// ✅ Always set expiry
_cache.Set("key", value, TimeSpan.FromMinutes(30));

// ❌ Not handling stampede on cold start
// Many requests all call DB at once

// ✅ Use GetOrCreateAsync or SemaphoreSlim lock
await _cache.GetOrCreateAsync("key", async entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
    return await _repo.GetExpensiveDataAsync();
});

// Cache key conventions
// Good: "entity:type:id:version"
// - product:1 → single product
// - products:category:electronics → category list
// - user:1:cart → user-specific data
// - dashboard:2026-01 → time-scoped data

// Decision guide:
// Single instance, low latency → IMemoryCache
// Multi-instance, shared state → IDistributedCache (Redis)
// Expensive queries, shared → Redis + sliding expiry
// HTTP responses → OutputCache
// Mixed workload → HybridCache (.NET 9)
```

---

## 📝 สรุป Part 70

| Strategy | Use Case | Library |
|----------|----------|---------|
| In-memory | Single-instance, low latency | IMemoryCache |
| Distributed | Multi-instance, session state | IDistributedCache |
| Redis advanced | Pub/Sub, sorted sets, locks | StackExchange.Redis |
| Response cache | HTTP GET responses | AddResponseCaching |
| Output cache | Endpoint-level (.NET 7+) | AddOutputCache |
| Hybrid cache | Both L1+L2 in one API (.NET 9) | HybridCache |

Cache invalidation strategies:
- **TTL**: ง่ายที่สุด, eventual consistency
- **Event-based**: precise, ต้องจัดการ event
- **Tag-based**: flexible, invalidate กลุ่ม
- **Version key**: simple global invalidation

---

**ก่อนหน้า → [Part 69: GraphQL](part69-graphql.md)**  
**ต่อไป → [Part 71: Observability & Monitoring](../part71-80/part71-observability.md)**
