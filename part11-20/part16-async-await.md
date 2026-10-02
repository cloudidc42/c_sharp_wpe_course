# Part 16: Async/Await
## ขั้นตอนที่ 151-160: Asynchronous Programming

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ async/await และ Task
- Task Parallelism (Task.WhenAll, Task.WhenAny)
- CancellationToken
- Progress Reporting
- Async Streams (IAsyncEnumerable)
- สร้าง Async API Client จริง

---

## ขั้นตอนที่ 151: async/await พื้นฐาน

### Task และ async/await
```csharp
using System.Threading.Tasks;

// Synchronous (blocking)
void FetchDataSync()
{
    Console.WriteLine("Fetching...");
    Thread.Sleep(2000);  // Blocks thread!
    Console.WriteLine("Done");
}

// Asynchronous (non-blocking)
async Task FetchDataAsync()
{
    Console.WriteLine("Fetching...");
    await Task.Delay(2000);  // Releases thread during wait
    Console.WriteLine("Done");
}

// async method ที่ return ค่า
async Task<string> GetUserNameAsync(int userId)
{
    await Task.Delay(100);  // จำลอง DB call
    return $"User {userId}";
}

// การเรียกใช้
async Task RunAsync()
{
    // await - รอ async operation เสร็จ
    string name = await GetUserNameAsync(1);
    Console.WriteLine($"Name: {name}");
    
    // ไม่ควร .Result หรือ .Wait() (deadlock risk!)
    // string bad = GetUserNameAsync(1).Result;  // ❌
}

await RunAsync();

// async ValueTask - ประหยัด allocation สำหรับ hot paths
async ValueTask<int> GetCountAsync()
{
    await Task.Yield();  // Force async context switch
    return 42;
}
```

### Task Status
```csharp
// Task states
var task = Task.Run(async () =>
{
    await Task.Delay(100);
    return 42;
});

Console.WriteLine($"Status: {task.Status}");  // Running or WaitingToRun

await task;
Console.WriteLine($"Status: {task.Status}");  // RanToCompletion
Console.WriteLine($"Result: {task.Result}");   // 42

// Exception handling
async Task MayFailAsync(bool shouldFail)
{
    await Task.Delay(50);
    if (shouldFail) throw new InvalidOperationException("Something went wrong");
}

try
{
    await MayFailAsync(true);
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"Caught: {ex.Message}");
}

// Task with timeout
async Task<T> WithTimeout<T>(Task<T> task, TimeSpan timeout)
{
    var timeoutTask = Task.Delay(timeout);
    var completed = await Task.WhenAny(task, timeoutTask);
    
    if (completed == timeoutTask)
        throw new TimeoutException("Operation timed out");
    
    return await task;
}
```

---

## ขั้นตอนที่ 152: Task Parallelism

### Task.WhenAll และ Task.WhenAny
```csharp
// WhenAll - รอทุก tasks เสร็จ
async Task<(string user, string[] orders, decimal balance)> LoadDashboardAsync(int userId)
{
    // รันพร้อมกัน
    var userTask = GetUserAsync(userId);
    var ordersTask = GetOrdersAsync(userId);
    var balanceTask = GetBalanceAsync(userId);
    
    // รอทั้งหมด
    await Task.WhenAll(userTask, ordersTask, balanceTask);
    
    return (await userTask, await ordersTask, await balanceTask);
}

async Task<string> GetUserAsync(int id)
{
    await Task.Delay(100);
    return $"User {id}";
}

async Task<string[]> GetOrdersAsync(int id)
{
    await Task.Delay(200);
    return new[] { "Order 1", "Order 2" };
}

async Task<decimal> GetBalanceAsync(int id)
{
    await Task.Delay(150);
    return 50000m;
}

// WhenAny - รอ task แรกที่เสร็จ (Race)
async Task<T> FirstSuccessAsync<T>(IEnumerable<Task<T>> tasks)
{
    var taskList = tasks.ToList();
    
    while (taskList.Any())
    {
        var completed = await Task.WhenAny(taskList);
        
        if (completed.IsCompletedSuccessfully)
            return await completed;
        
        taskList.Remove(completed);
    }
    
    throw new Exception("All tasks failed");
}

// Parallel processing with degree of parallelism
async Task ProcessInParallelAsync<T>(
    IEnumerable<T> items, 
    Func<T, Task> process, 
    int maxParallelism = 5)
{
    using var semaphore = new SemaphoreSlim(maxParallelism);
    
    var tasks = items.Select(async item =>
    {
        await semaphore.WaitAsync();
        try { await process(item); }
        finally { semaphore.Release(); }
    });
    
    await Task.WhenAll(tasks);
}

// การใช้งาน
var ids = Enumerable.Range(1, 20).ToList();
await ProcessInParallelAsync(
    ids,
    async id =>
    {
        await Task.Delay(100);
        Console.Write($"{id} ");
    },
    maxParallelism: 5
);
```

---

## ขั้นตอนที่ 153: CancellationToken

### ยกเลิก async operations
```csharp
// CancellationToken
async Task LongRunningOperationAsync(CancellationToken cancellationToken = default)
{
    Console.WriteLine("Starting long operation...");
    
    for (int i = 0; i < 10; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();
        
        await Task.Delay(500, cancellationToken);
        Console.WriteLine($"Step {i + 1}/10 completed");
    }
    
    Console.WriteLine("Operation completed!");
}

// การใช้งาน
var cts = new CancellationTokenSource();

// Cancel after 2 seconds
cts.CancelAfter(TimeSpan.FromSeconds(2));

try
{
    await LongRunningOperationAsync(cts.Token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Operation was cancelled!");
}
finally
{
    cts.Dispose();
}

// Linked tokens
async Task ProcessWithTimeoutAsync(int id, TimeSpan timeout, CancellationToken externalToken)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(externalToken);
    cts.CancelAfter(timeout);
    
    try
    {
        await Task.Delay(1000, cts.Token);
        Console.WriteLine($"Item {id} processed");
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine($"Item {id} timed out or cancelled");
    }
}
```

---

## ขั้นตอนที่ 154: IProgress<T>

### Progress Reporting
```csharp
record ProgressReport(int Current, int Total, string Message)
{
    public double Percent => Total > 0 ? (double)Current / Total * 100 : 0;
}

async Task ProcessFilesAsync(
    string[] files, 
    IProgress<ProgressReport>? progress = null,
    CancellationToken cancellationToken = default)
{
    for (int i = 0; i < files.Length; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();
        
        // Process file
        await Task.Delay(200, cancellationToken);
        
        // Report progress
        progress?.Report(new ProgressReport(i + 1, files.Length, $"Processed: {files[i]}"));
    }
}

// Console progress bar
var progress = new Progress<ProgressReport>(report =>
{
    int barWidth = 40;
    int filled = (int)(report.Percent / 100 * barWidth);
    string bar = new string('█', filled) + new string('░', barWidth - filled);
    
    Console.Write($"\r[{bar}] {report.Percent:F1}% - {report.Message,-30}");
    
    if (report.Current == report.Total)
        Console.WriteLine();
});

var files = Enumerable.Range(1, 20).Select(i => $"file_{i:000}.txt").ToArray();
await ProcessFilesAsync(files, progress);
```

---

## ขั้นตอนที่ 155: Async Streams

### IAsyncEnumerable<T>
```csharp
// IAsyncEnumerable - stream ข้อมูลแบบ async
async IAsyncEnumerable<int> GenerateNumbersAsync(
    int count, 
    [System.Runtime.CompilerServices.EnumeratorCancellation] 
    CancellationToken cancellationToken = default)
{
    for (int i = 0; i < count; i++)
    {
        await Task.Delay(100, cancellationToken);
        yield return i;
    }
}

// อ่าน async stream
await foreach (int n in GenerateNumbersAsync(10))
{
    Console.Write($"{n} ");
}

// Async stream จาก API
async IAsyncEnumerable<string> ReadLinesAsync(string path)
{
    using var reader = new System.IO.StreamReader(path);
    string? line;
    while ((line = await reader.ReadLineAsync()) != null)
        yield return line;
}

// Async LINQ (ใช้ System.Linq.Async NuGet package)
// var largeNumbers = await GenerateNumbersAsync(100)
//     .Where(n => n % 2 == 0)
//     .Select(n => n * n)
//     .ToListAsync();

// Manual async stream processing
async Task<List<string>> ProcessStreamAsync(IAsyncEnumerable<string> stream)
{
    var results = new List<string>();
    await foreach (var item in stream)
    {
        results.Add(item.ToUpper());
    }
    return results;
}
```

---

## ขั้นตอนที่ 156-160: โปรแกรมตัวอย่าง - Async HTTP Client

```csharp
using System;
using System.Collections.Generic;
using System.Net.Http;
using System.Text.Json;
using System.Threading;
using System.Threading.Tasks;

namespace AsyncHttpClient
{
    // Response models (จำลอง JSONPlaceholder API structure)
    record Post(int UserId, int Id, string Title, string Body);
    record User(int Id, string Name, string Email, string Phone);
    record Comment(int PostId, int Id, string Name, string Email, string Body);
    
    // Retry policy
    class RetryPolicy
    {
        private readonly int _maxRetries;
        private readonly TimeSpan _initialDelay;
        
        public RetryPolicy(int maxRetries = 3, TimeSpan? initialDelay = null)
        {
            _maxRetries = maxRetries;
            _initialDelay = initialDelay ?? TimeSpan.FromSeconds(1);
        }
        
        public async Task<T> ExecuteAsync<T>(Func<Task<T>> operation, 
            CancellationToken cancellationToken = default)
        {
            Exception? lastException = null;
            
            for (int attempt = 1; attempt <= _maxRetries; attempt++)
            {
                try
                {
                    return await operation();
                }
                catch (HttpRequestException ex) when (attempt < _maxRetries)
                {
                    lastException = ex;
                    TimeSpan delay = TimeSpan.FromMilliseconds(
                        _initialDelay.TotalMilliseconds * Math.Pow(2, attempt - 1));
                    
                    Console.WriteLine($"Attempt {attempt} failed. Retrying in {delay.TotalSeconds:F1}s...");
                    await Task.Delay(delay, cancellationToken);
                }
                catch (OperationCanceledException)
                {
                    throw;
                }
            }
            
            throw lastException ?? new Exception("All retries failed");
        }
    }
    
    // API Client
    class ApiClient : IDisposable
    {
        private readonly HttpClient _httpClient;
        private readonly RetryPolicy _retryPolicy;
        private readonly string _baseUrl;
        private static readonly JsonSerializerOptions _jsonOptions = new()
        {
            PropertyNameCaseInsensitive = true
        };
        
        public ApiClient(string baseUrl, RetryPolicy? retryPolicy = null)
        {
            _baseUrl = baseUrl.TrimEnd('/');
            _httpClient = new HttpClient
            {
                Timeout = TimeSpan.FromSeconds(30)
            };
            _retryPolicy = retryPolicy ?? new RetryPolicy();
        }
        
        private async Task<T> GetAsync<T>(string path, CancellationToken cancellationToken = default)
        {
            return await _retryPolicy.ExecuteAsync(async () =>
            {
                string url = $"{_baseUrl}/{path.TrimStart('/')}";
                Console.WriteLine($"  GET {url}");
                
                var response = await _httpClient.GetAsync(url, cancellationToken);
                response.EnsureSuccessStatusCode();
                
                string json = await response.Content.ReadAsStringAsync(cancellationToken);
                return JsonSerializer.Deserialize<T>(json, _jsonOptions)
                    ?? throw new JsonException("Null response");
            }, cancellationToken);
        }
        
        // Posts
        public Task<Post[]> GetPostsAsync(CancellationToken ct = default)
            => GetAsync<Post[]>("posts", ct);
        
        public Task<Post> GetPostAsync(int id, CancellationToken ct = default)
            => GetAsync<Post>($"posts/{id}", ct);
        
        public Task<Comment[]> GetPostCommentsAsync(int postId, CancellationToken ct = default)
            => GetAsync<Comment[]>($"posts/{postId}/comments", ct);
        
        // Users
        public Task<User[]> GetUsersAsync(CancellationToken ct = default)
            => GetAsync<User[]>("users", ct);
        
        public Task<User> GetUserAsync(int id, CancellationToken ct = default)
            => GetAsync<User>($"users/{id}", ct);
        
        // Load user with all their posts (parallel)
        public async Task<(User user, Post[] posts)> GetUserWithPostsAsync(
            int userId, CancellationToken ct = default)
        {
            var userTask = GetUserAsync(userId, ct);
            var postsTask = GetAsync<Post[]>($"users/{userId}/posts", ct);
            
            await Task.WhenAll(userTask, postsTask);
            
            return (await userTask, await postsTask);
        }
        
        // Batch load with progress
        public async Task<List<(User user, Post[] posts)>> LoadAllUsersWithPostsAsync(
            IProgress<ProgressInfo>? progress = null,
            CancellationToken ct = default)
        {
            var users = await GetUsersAsync(ct);
            var results = new List<(User, Post[])>();
            
            for (int i = 0; i < users.Length; i++)
            {
                ct.ThrowIfCancellationRequested();
                
                var user = users[i];
                var posts = await GetAsync<Post[]>($"users/{user.Id}/posts", ct);
                results.Add((user, posts));
                
                progress?.Report(new ProgressInfo(i + 1, users.Length, user.Name));
                await Task.Delay(50, ct);  // Rate limiting
            }
            
            return results;
        }
        
        // Async stream - lazy loading posts page by page
        public async IAsyncEnumerable<Post> StreamPostsAsync(
            int pageSize = 10,
            [System.Runtime.CompilerServices.EnumeratorCancellation] 
            CancellationToken ct = default)
        {
            var allPosts = await GetPostsAsync(ct);
            
            for (int i = 0; i < allPosts.Length; i += pageSize)
            {
                ct.ThrowIfCancellationRequested();
                
                var page = allPosts.Skip(i).Take(pageSize).ToArray();
                await Task.Delay(100, ct);  // จำลอง pagination delay
                
                foreach (var post in page)
                    yield return post;
                
                Console.WriteLine($"  Loaded page {i / pageSize + 1}");
            }
        }
        
        public void Dispose() => _httpClient.Dispose();
    }
    
    record ProgressInfo(int Current, int Total, string CurrentItem)
    {
        public double Percent => Total > 0 ? (double)Current / Total * 100 : 0;
    }
    
    class Program
    {
        static async Task Main()
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            Console.Title = "Async HTTP Client Demo";
            
            // ใช้ mock client เพื่อ demo (ไม่ต้องเชื่อมต่อ internet จริง)
            await RunMockDemo();
        }
        
        static async Task RunMockDemo()
        {
            Console.WriteLine("=== Async Programming Demo ===\n");
            
            // 1. Basic async/await
            Console.WriteLine("1. Basic async simulation:");
            await SimulateApiCallAsync("Fetching users", 200);
            await SimulateApiCallAsync("Fetching posts", 150);
            
            // 2. Parallel execution
            Console.WriteLine("\n2. Parallel execution:");
            var sw = System.Diagnostics.Stopwatch.StartNew();
            
            await Task.WhenAll(
                SimulateApiCallAsync("Users", 300),
                SimulateApiCallAsync("Posts", 200),
                SimulateApiCallAsync("Comments", 250)
            );
            
            Console.WriteLine($"All parallel tasks done in {sw.ElapsedMilliseconds}ms");
            
            // 3. Cancellation
            Console.WriteLine("\n3. Cancellation:");
            var cts = new CancellationTokenSource();
            cts.CancelAfter(TimeSpan.FromMilliseconds(300));
            
            try
            {
                for (int i = 0; i < 10; i++)
                {
                    cts.Token.ThrowIfCancellationRequested();
                    await Task.Delay(100, cts.Token);
                    Console.WriteLine($"  Step {i + 1} completed");
                }
            }
            catch (OperationCanceledException)
            {
                Console.WriteLine("  ⚠️ Operation cancelled after timeout!");
            }
            
            // 4. Progress reporting
            Console.WriteLine("\n4. Progress reporting:");
            var progress = new Progress<ProgressInfo>(info =>
            {
                int width = 30;
                int filled = (int)(info.Percent / 100 * width);
                string bar = new string('█', filled) + new string('░', width - filled);
                Console.Write($"\r  [{bar}] {info.Percent:F0}% - {info.CurrentItem,-20}");
                if (info.Current == info.Total) Console.WriteLine();
            });
            
            await ProcessItemsAsync(20, progress);
            
            // 5. Async streams
            Console.WriteLine("\n5. Async streams:");
            int count = 0;
            await foreach (var item in GenerateItemsAsync(15))
            {
                Console.Write($"{item} ");
                count++;
                if (count % 5 == 0) Console.WriteLine();
            }
            
            Console.WriteLine("\n\nกด Enter เพื่อออก...");
            Console.ReadLine();
        }
        
        static async Task SimulateApiCallAsync(string name, int delay)
        {
            await Task.Delay(delay);
            Console.WriteLine($"  ✅ {name} loaded ({delay}ms)");
        }
        
        static async Task ProcessItemsAsync(int count, IProgress<ProgressInfo> progress)
        {
            for (int i = 0; i < count; i++)
            {
                await Task.Delay(100);
                progress.Report(new ProgressInfo(i + 1, count, $"Item {i + 1}"));
            }
        }
        
        static async IAsyncEnumerable<string> GenerateItemsAsync(int count)
        {
            for (int i = 1; i <= count; i++)
            {
                await Task.Delay(80);
                yield return $"[{i}]";
            }
        }
    }
}
```

---

## 📝 สรุป Part 16

| หัวข้อ | Key Points |
|--------|-----------|
| async/await | Non-blocking asynchronous code |
| Task<T> | Represents async operation |
| Task.WhenAll | Wait for all tasks |
| Task.WhenAny | Wait for first task |
| CancellationToken | Cancel long operations |
| IProgress<T> | Report async progress |
| IAsyncEnumerable | Async streams |
| SemaphoreSlim | Limit parallelism |
| Retry Policy | Handle transient failures |

---

**ก่อนหน้า → [Part 15: Delegates & Events](part15-delegates-events.md)**  
**ต่อไป → [Part 17: Pattern Matching](part17-pattern-matching.md)**
