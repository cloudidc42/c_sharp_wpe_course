# Part 53: Performance Optimization
## ขั้นตอนที่ 521-530: เขียน C# ให้เร็วและใช้ Memory น้อย

---

## 🎯 เป้าหมายของ Part นี้
- Span<T> และ Memory<T>
- ArrayPool<T>
- ValueTask vs Task
- StringBuilder vs string concatenation
- Object pooling
- Profiling with BenchmarkDotNet
- Async/Await best practices
- GC pressure reduction

---

## ขั้นตอนที่ 521: Span<T> - Zero-allocation Slicing

```csharp
// Span<T>: ทำงานกับ memory โดยตรง ไม่ allocate ใหม่
// เหมาะกับ: parsing, substring operations, array slicing

// BAD: สร้าง string ใหม่ทุกครั้ง
string ParseBad(string csv)
{
    var parts = csv.Split(',');      // allocates array + strings
    return parts[0].Trim().ToUpper(); // more allocations
}

// GOOD: ใช้ Span<T>
ReadOnlySpan<char> ParseGood(ReadOnlySpan<char> csv)
{
    var comma = csv.IndexOf(',');
    var first = comma >= 0 ? csv[..comma] : csv;
    // No allocation! Span is a view into existing memory
    return first.Trim();
}

// Span with stackalloc (stack allocation instead of heap)
void ProcessNumbers(ReadOnlySpan<int> input)
{
    // Allocate on stack for small buffers (< ~1KB)
    Span<int> doubled = stackalloc int[input.Length];
    for (int i = 0; i < input.Length; i++)
        doubled[i] = input[i] * 2;
    
    // Process doubled without any heap allocation
    int sum = 0;
    foreach (var n in doubled) sum += n;
    Console.WriteLine($"Sum of doubled: {sum}");
}

// Parse CSV line without allocations
public static void ParseCsvLine(ReadOnlySpan<char> line, List<string> result)
{
    int start = 0;
    for (int i = 0; i <= line.Length; i++)
    {
        if (i == line.Length || line[i] == ',')
        {
            var field = line[start..i].Trim();
            result.Add(field.ToString()); // only allocate when needed
            start = i + 1;
        }
    }
}

// Memory<T>: async-compatible Span
async Task ProcessFileAsync(string path)
{
    await using var fs = File.OpenRead(path);
    using var owner = MemoryPool<byte>.Shared.Rent(4096);
    Memory<byte> buffer = owner.Memory[..4096];
    
    int read;
    while ((read = await fs.ReadAsync(buffer)) > 0)
    {
        ProcessBuffer(buffer.Span[..read]); // pass Span to sync method
    }
}
```

---

## ขั้นตอนที่ 522: ArrayPool<T> - Reuse Arrays

```csharp
// ArrayPool<T>: reuse arrays แทนการ allocate ใหม่
// ลด GC pressure สำหรับ short-lived large arrays

// BAD: allocate ใหม่ทุกครั้ง
byte[] ProcessRequestBad(Stream input)
{
    var buffer = new byte[65536]; // new allocation every call
    var count = input.Read(buffer, 0, buffer.Length);
    return ProcessData(buffer, count);
}

// GOOD: rent from pool
byte[] ProcessRequestGood(Stream input)
{
    var pool = ArrayPool<byte>.Shared;
    var buffer = pool.Rent(65536); // get from pool (may be larger than requested)
    try
    {
        var count = input.Read(buffer, 0, 65536);
        return ProcessData(buffer, count); // if you need to return a copy
    }
    finally
    {
        pool.Return(buffer, clearArray: false); // return to pool
    }
}

// Custom pool for specific types
var pool = ArrayPool<int>.Create(maxArrayLength: 1024 * 1024, maxArraysPerBucket: 50);

// RecyclableMemoryStream - for MemoryStream pooling
// Install: Microsoft.IO.RecyclableMemoryStream
var msManager = new RecyclableMemoryStreamManager();

using var ms = msManager.GetStream();
// Use ms like a regular MemoryStream
// Automatically returns to pool on Dispose
```

---

## ขั้นตอนที่ 523: ValueTask vs Task

```csharp
// ValueTask: ใช้เมื่อ operation มักจะ sync (no heap allocation)
// Task: ใช้เมื่อ operation มักจะ async (always allocates)

// ตัวอย่าง: Cache lookup - ส่วนใหญ่จะ hit cache (sync path)
public class CacheService
{
    private readonly Dictionary<string, object> _cache = new();
    private readonly IDatabase _db;
    
    // BAD: Task allocates even when value is in cache
    public async Task<T?> GetTaskAsync<T>(string key)
    {
        if (_cache.TryGetValue(key, out var cached))
            return (T?)cached; // still allocates Task<T>
        
        var value = await _db.GetAsync<T>(key);
        if (value != null) _cache[key] = value;
        return value;
    }
    
    // GOOD: ValueTask - no allocation on cache hit (hot path)
    public ValueTask<T?> GetValueTaskAsync<T>(string key)
    {
        if (_cache.TryGetValue(key, out var cached))
            return new ValueTask<T?>((T?)cached); // NO allocation!
        
        return new ValueTask<T?>(GetFromDbAsync<T>(key)); // wraps Task
    }
    
    private async Task<T?> GetFromDbAsync<T>(string key)
    {
        var value = await _db.GetAsync<T>(key);
        if (value != null) _cache[key] = value;
        return value;
    }
}

// Rules for ValueTask:
// ✓ Use when result often available synchronously
// ✓ Use for high-frequency, low-latency operations
// ✗ Don't await ValueTask multiple times
// ✗ Don't use in fire-and-forget scenarios
```

---

## ขั้นตอนที่ 524: String Performance

```csharp
// String is immutable: concatenation creates new strings
// Bad for loops!

// BAD: O(n²) memory, n allocations
string BuildBad(int count)
{
    string result = "";
    for (int i = 0; i < count; i++)
        result += $"item{i}, "; // new string each iteration!
    return result;
}

// GOOD: StringBuilder
string BuildGood(int count)
{
    var sb = new StringBuilder(count * 10); // pre-size estimate
    for (int i = 0; i < count; i++)
        sb.Append("item").Append(i).Append(", ");
    
    if (sb.Length > 2) sb.Length -= 2; // remove trailing ", "
    return sb.ToString(); // ONE allocation at end
}

// String.Create - zero-copy string creation (advanced)
string CreateFormatted(int value, string name)
{
    return string.Create(20, (value, name), static (span, state) =>
    {
        var (v, n) = state;
        n.CopyTo(span);
        span[n.Length] = ':';
        v.TryFormat(span[(n.Length + 1)..], out _);
    });
}

// Interpolated string handlers (.NET 6+) - avoid allocation in common cases
// CompositeFormat for reusable format strings
var format = CompositeFormat.Parse("{0}: {1:N0} บาท");
string msg = string.Format(CultureInfo.CurrentCulture, format, "ยอดขาย", 150000);

// StringPool for interning
var pool = new StringPool();
string a = pool.GetOrAdd("hello");
string b = pool.GetOrAdd("hello"); // same reference as a
Console.WriteLine(ReferenceEquals(a, b)); // True
```

---

## ขั้นตอนที่ 525: BenchmarkDotNet - Measure Performance

```csharp
// Install: dotnet add package BenchmarkDotNet

[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
public class StringBenchmarks
{
    private const int Count = 1000;
    
    [Benchmark(Baseline = true)]
    public string Concatenation()
    {
        string result = "";
        for (int i = 0; i < Count; i++)
            result += $"item{i}";
        return result;
    }
    
    [Benchmark]
    public string StringBuilderPreSized()
    {
        var sb = new StringBuilder(Count * 7);
        for (int i = 0; i < Count; i++)
            sb.Append("item").Append(i);
        return sb.ToString();
    }
    
    [Benchmark]
    public string StringCreate()
    {
        return string.Create(Count * 7, Count, static (span, c) =>
        {
            int pos = 0;
            for (int i = 0; i < c; i++)
            {
                "item".CopyTo(span[pos..]);
                pos += 4;
                i.TryFormat(span[pos..], out int written);
                pos += written;
            }
        });
    }
}

// Run benchmarks
// dotnet run -c Release -- --filter *StringBenchmarks*
// Output shows:
// Method                  | Mean       | Gen0   | Allocated
// ----------------------- | ---------- | ------ | ---------
// Concatenation           | 2,345.6 μs | 4600   | 4.5 MB
// StringBuilderPreSized   |    45.2 μs |   32   | 32 KB
// StringCreate            |    12.1 μs |    8   | 8 KB
```

---

## ขั้นตอนที่ 526: Async/Await Best Practices

```csharp
// ✓ Always use ConfigureAwait(false) in libraries
public async Task<string> LibraryMethodAsync()
{
    var data = await GetDataAsync().ConfigureAwait(false);
    return data.ToString();
}

// ✗ Avoid async void (can't catch exceptions)
// void ButtonClick(object s, EventArgs e) => await LoadAsync(); // ERROR

// ✓ Use IAsyncEnumerable for streaming
public async IAsyncEnumerable<int> StreamNumbers(int count, [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < count; i++)
    {
        await Task.Delay(1, ct); // simulate work
        yield return i;
    }
}

await foreach (var n in StreamNumbers(100))
    Console.WriteLine(n);

// ✓ Parallel async operations
var task1 = GetDataAsync();
var task2 = GetMoreDataAsync();
var (data1, data2) = (await task1, await task2); // both run concurrently

// ✓ Better: WhenAll
var (r1, r2, r3) = await (
    await Task.WhenAll(task1, task2, task3) // parallel
).Deconstruct();

// CancellationToken everywhere
public async Task<List<Product>> GetProductsAsync(CancellationToken ct = default)
{
    return await _ctx.Products
        .AsNoTracking()
        .ToListAsync(ct); // pass token to EF
}

// Timeout
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
try
{
    var result = await GetProductsAsync(cts.Token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Request timed out");
}
```

---

## ขั้นตอนที่ 527-530: Real Performance Optimization

```csharp
// Record struct for small value types (stack-allocated)
public readonly record struct Point2D(double X, double Y)
{
    public double Distance(Point2D other)
        => Math.Sqrt(Math.Pow(X - other.X, 2) + Math.Pow(Y - other.Y, 2));
}

// Frozen collections (.NET 8) for read-only lookup
var lookup = FrozenDictionary.ToFrozenDictionary(
    items.Select(i => KeyValuePair.Create(i.Id, i))
);
// 2-5x faster than Dictionary for read-heavy workloads

// CollectionsMarshal for unsafe but fast Dictionary access
var dict = new Dictionary<string, int>();
ref int val = ref CollectionsMarshal.GetValueRefOrAddDefault(dict, "key", out bool exists);
val = exists ? val + 1 : 1; // update in-place, no double lookup

// Object Reuse: IDisposable pooling
public class ExpensiveObject : IDisposable
{
    private static readonly ObjectPool<ExpensiveObject> _pool = 
        new DefaultObjectPool<ExpensiveObject>(new DefaultPooledObjectPolicy<ExpensiveObject>());
    
    public static ExpensiveObject Rent() => _pool.Get();
    
    public void Dispose() => _pool.Return(this);
    
    public void Reset() { /* Clear state for reuse */ }
}

// Channels for producer-consumer (no lock, high throughput)
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
});

// Producer
var producer = Task.Run(async () =>
{
    await foreach (var item in GetItemsAsync())
        await channel.Writer.WriteAsync(item);
    channel.Writer.Complete();
});

// Consumer
var consumer = Task.Run(async () =>
{
    await foreach (var item in channel.Reader.ReadAllAsync())
        await ProcessAsync(item);
});

await Task.WhenAll(producer, consumer);

// Performance tips summary
Console.WriteLine("""
Performance Checklist:
✓ Use Span<T>/Memory<T> for buffer operations
✓ Use ArrayPool<T> for temporary large arrays
✓ Use ValueTask for frequently synchronous ops
✓ Use StringBuilder for string building in loops
✓ Avoid async void
✓ Always pass CancellationToken
✓ Use AsNoTracking() for read-only queries
✓ Use compiled queries for hot paths
✓ Profile with BenchmarkDotNet before optimizing
✓ Measure, don't guess!
""");
```

---

## 📝 สรุป Part 53

| Technique | ประโยชน์ |
|-----------|---------|
| Span<T> | Zero-allocation buffer ops |
| ArrayPool<T> | Reuse large temp arrays |
| ValueTask | No alloc on sync path |
| StringBuilder | Fast string building |
| Frozen collections | Faster read-only lookups |
| Channels | Lock-free producer-consumer |
| BenchmarkDotNet | Accurate measurement |

---

**ก่อนหน้า → [Part 52: Unit Testing](part52-unit-testing.md)**  
**ต่อไป → [Part 54: Security](part54-security.md)**
