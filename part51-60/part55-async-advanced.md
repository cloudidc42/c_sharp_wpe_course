# Part 55: Advanced Async & Concurrency
## ขั้นตอนที่ 541-550: Async Patterns ขั้นสูง

---

## 🎯 เป้าหมายของ Part นี้
- async/await deep dive
- SemaphoreSlim for rate limiting
- Channels for producer-consumer
- Parallel.ForEachAsync
- TaskCompletionSource
- IAsyncEnumerable
- CancellationToken best practices
- Actor model ด้วย Channels

---

## ขั้นตอนที่ 541: SemaphoreSlim - Control Concurrency

```csharp
// SemaphoreSlim: จำกัดจำนวน concurrent operations
// ใช้แทน lock() สำหรับ async code

// Limit concurrent HTTP requests to 5
var semaphore = new SemaphoreSlim(5, 5);

async Task<string> FetchWithRateLimit(string url)
{
    await semaphore.WaitAsync();
    try
    {
        return await _httpClient.GetStringAsync(url);
    }
    finally
    {
        semaphore.Release();
    }
}

// Process 100 URLs with max 5 concurrent
var urls = Enumerable.Range(1, 100).Select(i => $"https://api.example.com/data/{i}");
var tasks = urls.Select(url => FetchWithRateLimit(url));
var results = await Task.WhenAll(tasks);

// Throttled parallel processing
public static async Task<List<TResult>> ParallelWithLimit<T, TResult>(
    IEnumerable<T> items,
    Func<T, Task<TResult>> processor,
    int maxConcurrency = 10)
{
    var sem = new SemaphoreSlim(maxConcurrency, maxConcurrency);
    var tasks = items.Select(async item =>
    {
        await sem.WaitAsync();
        try { return await processor(item); }
        finally { sem.Release(); }
    });
    return (await Task.WhenAll(tasks)).ToList();
}

// Usage:
var results = await ParallelWithLimit(productIds, 
    async id => await FetchProductAsync(id), 
    maxConcurrency: 5);
```

---

## ขั้นตอนที่ 542: Channels - High-Performance Queues

```csharp
// System.Threading.Channels: lock-free producer-consumer
// Better than BlockingCollection for async code

// Bounded channel (backpressure when full)
var options = new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleWriter = false,
    SingleReader = false
};
var channel = Channel.CreateBounded<OrderEvent>(options);

// Producer: write events
async Task ProduceOrderEvents(CancellationToken ct)
{
    await foreach (var order in GetOrdersAsync(ct))
    {
        var evt = new OrderEvent(order.Id, EventType.Created);
        await channel.Writer.WriteAsync(evt, ct);
        Console.WriteLine($"Produced: {evt}");
    }
    channel.Writer.Complete();
}

// Consumer: process events
async Task ConsumeOrderEvents(int consumerId, CancellationToken ct)
{
    await foreach (var evt in channel.Reader.ReadAllAsync(ct))
    {
        await ProcessEventAsync(evt);
        Console.WriteLine($"Consumer {consumerId} processed: {evt}");
    }
}

// Multiple consumers (fan-out)
var cts = new CancellationTokenSource();
var producer = ProduceOrderEvents(cts.Token);
var consumers = Enumerable.Range(1, 3).Select(i => ConsumeOrderEvents(i, cts.Token));
await Task.WhenAll(new[] { producer }.Concat(consumers));

// Broadcast channel (one producer, many consumers)
public class EventBus<T>
{
    private readonly List<Channel<T>> _channels = new();
    private readonly object _lock = new();
    
    public ChannelReader<T> Subscribe()
    {
        var ch = Channel.CreateUnbounded<T>();
        lock (_lock) _channels.Add(ch);
        return ch.Reader;
    }
    
    public async Task PublishAsync(T message)
    {
        List<Channel<T>> snapshot;
        lock (_lock) snapshot = new List<Channel<T>>(_channels);
        
        foreach (var ch in snapshot)
            await ch.Writer.WriteAsync(message);
    }
}
```

---

## ขั้นตอนที่ 543: IAsyncEnumerable - Streaming Data

```csharp
// IAsyncEnumerable: stream data lazily from async source
// Perfect for: database streaming, file reading, API pagination

// Database streaming with EF Core
public async IAsyncEnumerable<ProductDto> StreamProductsAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    // AsAsyncEnumerable() streams rows instead of loading all
    await foreach (var product in _ctx.Products
        .AsNoTracking()
        .AsAsyncEnumerable()
        .WithCancellation(ct))
    {
        yield return new ProductDto(product.Id, product.Name, product.Price);
    }
}

// Consumer: process streamed results
await foreach (var dto in StreamProductsAsync())
{
    await ProcessProductAsync(dto);
    // Memory stays constant regardless of dataset size!
}

// Pagination streaming
public async IAsyncEnumerable<T> StreamPagedAsync<T>(
    Func<int, int, Task<IReadOnlyList<T>>> fetcher,
    int pageSize = 100,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    int page = 1;
    IReadOnlyList<T> current;
    do
    {
        current = await fetcher(page++, pageSize);
        foreach (var item in current)
            yield return item;
    } while (current.Count == pageSize && !ct.IsCancellationRequested);
}

// Usage:
await foreach (var product in StreamPagedAsync(
    (page, size) => _repo.GetProductsAsync(page, size)))
{
    Console.WriteLine(product.Name);
}

// CSV streaming writer
public async Task ExportToCsvAsync(IAsyncEnumerable<ProductDto> products, string path)
{
    await using var writer = new StreamWriter(path, false, Encoding.UTF8, bufferSize: 65536);
    await writer.WriteLineAsync("Id,Name,Price");
    
    await foreach (var p in products)
        await writer.WriteLineAsync($"{p.Id},{p.Name},{p.Price}");
}
```

---

## ขั้นตอนที่ 544: TaskCompletionSource - Wrap Non-Async APIs

```csharp
// TaskCompletionSource: bridge between callback and async/await
// Use when wrapping event-driven or callback APIs

// Wrap event-based async (old pattern)
public Task<string> WaitForResponse(string requestId)
{
    var tcs = new TaskCompletionSource<string>();
    
    // Register one-time event handler
    void OnResponseReceived(object? s, ResponseEventArgs e)
    {
        if (e.RequestId != requestId) return;
        _server.ResponseReceived -= OnResponseReceived;
        tcs.TrySetResult(e.Data);
    }
    
    _server.ResponseReceived += OnResponseReceived;
    return tcs.Task;
}

// With timeout and cancellation
public Task<string> WaitForResponseWithTimeout(string requestId, TimeSpan timeout)
{
    var tcs = new TaskCompletionSource<string>();
    var cts = new CancellationTokenSource(timeout);
    
    void OnResponseReceived(object? s, ResponseEventArgs e)
    {
        if (e.RequestId != requestId) return;
        _server.ResponseReceived -= OnResponseReceived;
        cts.Dispose();
        tcs.TrySetResult(e.Data);
    }
    
    cts.Token.Register(() =>
    {
        _server.ResponseReceived -= OnResponseReceived;
        tcs.TrySetCanceled(cts.Token);
    });
    
    _server.ResponseReceived += OnResponseReceived;
    return tcs.Task;
}

// Coordinate multiple async operations
public class AsyncBarrier
{
    private readonly TaskCompletionSource _tcs = new();
    private int _count;
    
    public AsyncBarrier(int count) => _count = count;
    
    public Task WaitAsync() => _tcs.Task;
    
    public void Signal()
    {
        if (Interlocked.Decrement(ref _count) == 0)
            _tcs.TrySetResult();
    }
}

var barrier = new AsyncBarrier(3);
// 3 workers signal when ready
var worker1 = StartWorkerAsync("w1", barrier);
var worker2 = StartWorkerAsync("w2", barrier);
var worker3 = StartWorkerAsync("w3", barrier);

await barrier.WaitAsync(); // wait for all 3 to be ready
Console.WriteLine("All workers ready, starting...");
```

---

## ขั้นตอนที่ 545: Parallel.ForEachAsync (.NET 6+)

```csharp
// Parallel.ForEachAsync: built-in parallel async processing
// Better than Task.WhenAll for large collections

var products = await _repo.GetAllAsync();

// Process with max 5 concurrent async operations
await Parallel.ForEachAsync(products, 
    new ParallelOptions { MaxDegreeOfParallelism = 5, CancellationToken = ct },
    async (product, token) =>
    {
        var enriched = await _enrichmentService.EnrichAsync(product, token);
        await _cache.SetAsync(product.Id.ToString(), enriched, token);
    });

// Batch processing with Parallel.ForEachAsync
var batches = products.Chunk(100); // .NET 6+ built-in
await Parallel.ForEachAsync(batches,
    new ParallelOptions { MaxDegreeOfParallelism = 3 },
    async (batch, token) =>
    {
        await _bulkImporter.ImportAsync(batch, token);
        Console.WriteLine($"Imported batch of {batch.Length}");
    });

// Progress reporting
var processed = 0;
var total = products.Count;

await Parallel.ForEachAsync(products,
    new ParallelOptions { MaxDegreeOfParallelism = 10 },
    async (product, token) =>
    {
        await ProcessAsync(product, token);
        var count = Interlocked.Increment(ref processed);
        Console.Write($"\r{count}/{total} ({count * 100 / total}%)");
    });
```

---

## ขั้นตอนที่ 546: Actor Model with Channels

```csharp
// Actor pattern: each actor has own message queue
// No shared state between actors = no race conditions!

public abstract class Actor<TMessage> : IAsyncDisposable
{
    private readonly Channel<TMessage> _mailbox = Channel.CreateUnbounded<TMessage>();
    private readonly CancellationTokenSource _cts = new();
    private readonly Task _processingTask;
    
    protected Actor()
    {
        _processingTask = Task.Run(ProcessMailboxAsync);
    }
    
    public ValueTask SendAsync(TMessage message)
        => _mailbox.Writer.WriteAsync(message, _cts.Token);
    
    protected abstract Task HandleAsync(TMessage message);
    
    private async Task ProcessMailboxAsync()
    {
        await foreach (var msg in _mailbox.Reader.ReadAllAsync(_cts.Token))
        {
            try { await HandleAsync(msg); }
            catch (Exception ex) { OnError(ex); }
        }
    }
    
    protected virtual void OnError(Exception ex) 
        => Console.WriteLine($"Actor error: {ex.Message}");
    
    public async ValueTask DisposeAsync()
    {
        _mailbox.Writer.Complete();
        _cts.Cancel();
        await _processingTask;
        _cts.Dispose();
    }
}

// Concrete actor: Order Processor
public record ProcessOrderMessage(int OrderId, decimal Amount);
public record CancelOrderMessage(int OrderId, string Reason);

public class OrderActor : Actor<object>
{
    private readonly IOrderService _service;
    
    public OrderActor(IOrderService service) => _service = service;
    
    protected override async Task HandleAsync(object message)
    {
        switch (message)
        {
            case ProcessOrderMessage m:
                await _service.ProcessAsync(m.OrderId, m.Amount);
                break;
            case CancelOrderMessage m:
                await _service.CancelAsync(m.OrderId, m.Reason);
                break;
        }
    }
}

// Usage: no race conditions!
await using var actor = new OrderActor(orderService);
await actor.SendAsync(new ProcessOrderMessage(1, 1500m));
await actor.SendAsync(new ProcessOrderMessage(2, 2000m));
await actor.SendAsync(new CancelOrderMessage(3, "Out of stock"));
```

---

## ขั้นตอนที่ 547-550: CancellationToken Deep Dive

```csharp
// CancellationToken: cooperative cancellation
// Always propagate tokens through the call chain!

// Link multiple tokens (cancel when any cancels)
var userCts = new CancellationTokenSource();
var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
using var linked = CancellationTokenSource.CreateLinkedTokenSource(
    userCts.Token, timeoutCts.Token);

try
{
    await LongRunningOperationAsync(linked.Token);
}
catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested)
{
    Console.WriteLine("Timed out after 30 seconds");
}
catch (OperationCanceledException) when (userCts.IsCancellationRequested)
{
    Console.WriteLine("Cancelled by user");
}

// Custom cancellation with callback
async Task LongRunningOperationAsync(CancellationToken ct)
{
    using var registration = ct.Register(() => Console.WriteLine("Cancellation requested"));
    
    for (int i = 0; i < 1000; i++)
    {
        ct.ThrowIfCancellationRequested(); // check at checkpoints
        await Task.Delay(10, ct); // pass token to awaitable ops
        Console.Write($"\r{i}/1000");
    }
}

// Graceful shutdown in hosted services
public class BackgroundWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await DoWorkAsync(stoppingToken);
                await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
            }
            catch (OperationCanceledException) { break; }
        }
    }
}
```

---

## 📝 สรุป Part 55

| Concept | ใช้เมื่อ |
|---------|---------|
| SemaphoreSlim | จำกัด concurrent operations |
| Channels | Producer-consumer queue |
| IAsyncEnumerable | Streaming large datasets |
| TaskCompletionSource | Wrap event-based APIs |
| Parallel.ForEachAsync | Parallel async processing |
| Actor model | Isolated stateful processing |
| CancellationToken | Cooperative cancellation |

---

**ก่อนหน้า → [Part 54: Security](part54-security.md)**  
**ต่อไป → [Part 56: Advanced C# Features](part56-advanced-csharp.md)**
