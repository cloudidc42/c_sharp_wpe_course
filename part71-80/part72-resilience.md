# Part 72: Resilience Patterns ใน C# ด้วย Polly v8

## ภาพรวม

ในระบบซอฟต์แวร์สมัยใหม่ โดยเฉพาะระบบที่ต้องสื่อสารผ่านเครือข่าย (HTTP, gRPC, Message Queue) ความล้มเหลวชั่วคราว (Transient Failures) เป็นเรื่องปกติที่หลีกเลี่ยงไม่ได้ Part นี้จะสอนวิธีสร้างระบบที่มี **Resilience** (ความยืดหยุ่น/ทนทาน) โดยใช้ **Polly v8** ซึ่งเป็น library มาตรฐานสำหรับ .NET

---

## Step 711: ทำไม Resilience ถึงสำคัญ และภาพรวม Polly v8

### ทำไมระบบถึงล้มเหลว?

ในสถาปัตยกรรม Microservices หรือระบบที่ต้องพึ่งพา External Services ปัญหาที่พบบ่อยได้แก่:

- **Network Timeout**: การเชื่อมต่อหมดเวลา
- **Transient Errors**: ข้อผิดพลาดชั่วคราวที่หายไปเองหลังลองใหม่
- **Service Overload**: Service ที่ถูกเรียกรับโหลดมากเกินไป
- **Cascading Failures**: ความล้มเหลวที่กระจายไปยัง Service อื่น ๆ

### Polly v8 คืออะไร?

**Polly** เป็น .NET library สำหรับ Resilience และ Transient Fault Handling โดย Polly v8 มีการออกแบบใหม่ทั้งหมดเพื่อรองรับ .NET 8+ พร้อม API ที่ชัดเจนและ Fluent Builder Pattern

**ความแตกต่างจาก Polly v7:**

| Feature | Polly v7 | Polly v8 |
|---------|---------|---------|
| API Style | `Policy.Handle<>().Retry()` | `ResiliencePipelineBuilder` |
| Async/Sync | แยก class | รองรับทั้งคู่ |
| Telemetry | ต้องทำเอง | Built-in via `DiagnosticSource` |
| DI Integration | ผ่าน Extension | `AddResiliencePipeline()` |
| Rate Limiting | ไม่มี | Built-in |

### การติดตั้ง Polly v8

```xml
<!-- ใน .csproj -->
<PackageReference Include="Polly" Version="8.*" />
<PackageReference Include="Polly.Extensions" Version="8.*" />
<PackageReference Include="Microsoft.Extensions.Http.Resilience" Version="8.*" />
```

หรือผ่าน CLI:

```bash
dotnet add package Polly
dotnet add package Polly.Extensions
dotnet add package Microsoft.Extensions.Http.Resilience
```

### โครงสร้าง Polly v8

```csharp
using Polly;
using Polly.Retry;
using Polly.CircuitBreaker;
using Polly.Timeout;

// โครงสร้างพื้นฐาน: ResiliencePipeline
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions())         // Step 712
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions()) // Step 713
    .AddTimeout(TimeSpan.FromSeconds(10))          // Step 714
    .Build();

// การใช้งาน
await pipeline.ExecuteAsync(async cancellationToken =>
{
    // โค้ดที่ต้องการความ Resilience
    var result = await httpClient.GetStringAsync("https://api.example.com", cancellationToken);
    return result;
});
```

### Core Concepts ใน Polly v8

```csharp
// 1. ResiliencePipeline - สำหรับ void หรือ non-generic
ResiliencePipeline pipeline = new ResiliencePipelineBuilder().Build();

// 2. ResiliencePipeline<T> - สำหรับ return type ที่ระบุ
ResiliencePipeline<HttpResponseMessage> typedPipeline = 
    new ResiliencePipelineBuilder<HttpResponseMessage>().Build();

// 3. ResilienceContext - ข้อมูล context ที่ส่งระหว่าง execution
ResilienceContext context = ResilienceContextPool.Shared.Get();
context.Properties.Set(new ResiliencePropertyKey<string>("requestId"), "REQ-001");

try
{
    await pipeline.ExecuteAsync(async ctx =>
    {
        var requestId = ctx.Properties.GetValue(
            new ResiliencePropertyKey<string>("requestId"), "unknown");
        Console.WriteLine($"Executing request: {requestId}");
        // ... logic
    }, context);
}
finally
{
    ResilienceContextPool.Shared.Return(context);
}
```

---

## Step 712: Retry Policy (Exponential Backoff + Jitter)

### พื้นฐาน Retry

Retry Policy คือการลองทำซ้ำเมื่อเกิดข้อผิดพลาด ซึ่งเหมาะสำหรับ **Transient Failures** ที่หายไปเองหลังจากเวลาผ่านไป

```csharp
using Polly;
using Polly.Retry;

// Retry แบบพื้นฐาน - ลอง 3 ครั้ง
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromSeconds(1),
        BackoffType = DelayBackoffType.Constant, // รอเท่ากันทุกครั้ง
        UseJitter = false
    })
    .Build();
```

### Exponential Backoff

Exponential Backoff คือการเพิ่มเวลารอแบบทวีคูณ เพื่อลดแรงกดดันต่อ Service ที่กำลังมีปัญหา

```csharp
// Exponential Backoff: 1s -> 2s -> 4s -> 8s
var retryOptions = new RetryStrategyOptions
{
    MaxRetryAttempts = 4,
    Delay = TimeSpan.FromSeconds(1),
    MaxDelay = TimeSpan.FromSeconds(30), // จำกัดเวลารอสูงสุด
    BackoffType = DelayBackoffType.Exponential,
    UseJitter = false // เปิด jitter ในขั้นถัดไป
};

var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(retryOptions)
    .Build();
```

### Jitter คืออะไร และทำไมต้องใช้?

**Jitter** คือการเพิ่ม "ความสุ่ม" ให้กับเวลารอ เพื่อป้องกัน **Thundering Herd Problem** ซึ่งเกิดเมื่อ Client หลายตัวลองใหม่พร้อมกันและทำให้ Server โหลดพุ่งสูง

```csharp
// Exponential Backoff + Jitter (แนะนำสำหรับ Production)
var retryOptions = new RetryStrategyOptions
{
    MaxRetryAttempts = 5,
    Delay = TimeSpan.FromMilliseconds(500),
    MaxDelay = TimeSpan.FromSeconds(60),
    BackoffType = DelayBackoffType.Exponential,
    UseJitter = true, // เพิ่ม ±25% random jitter
    
    // กำหนด Exception ที่จะ Retry
    ShouldHandle = new PredicateBuilder()
        .Handle<HttpRequestException>()
        .Handle<TimeoutRejectedException>()
        .HandleResult<HttpResponseMessage>(r => 
            r.StatusCode == System.Net.HttpStatusCode.ServiceUnavailable ||
            r.StatusCode == System.Net.HttpStatusCode.TooManyRequests),
    
    // Callback เมื่อเกิด Retry
    OnRetry = args =>
    {
        Console.WriteLine($"Retry #{args.AttemptNumber} after {args.RetryDelay.TotalSeconds:F1}s. " +
                          $"Exception: {args.Outcome.Exception?.Message}");
        return ValueTask.CompletedTask;
    }
};

var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddRetry(retryOptions)
    .Build();
```

### ตัวอย่างเต็ม: HTTP Client with Retry

```csharp
using System.Net.Http;
using Polly;
using Polly.Retry;

public class WeatherService
{
    private readonly HttpClient _httpClient;
    private readonly ResiliencePipeline<string> _pipeline;

    public WeatherService(HttpClient httpClient)
    {
        _httpClient = httpClient;
        _pipeline = BuildPipeline();
    }

    private ResiliencePipeline<string> BuildPipeline()
    {
        return new ResiliencePipelineBuilder<string>()
            .AddRetry(new RetryStrategyOptions<string>
            {
                MaxRetryAttempts = 3,
                Delay = TimeSpan.FromSeconds(1),
                BackoffType = DelayBackoffType.Exponential,
                UseJitter = true,
                
                ShouldHandle = new PredicateBuilder<string>()
                    .Handle<HttpRequestException>()
                    .Handle<TaskCanceledException>(),
                
                OnRetry = args =>
                {
                    var attempt = args.AttemptNumber + 1;
                    Console.WriteLine($"[Weather] Retry attempt {attempt}/3, " +
                                      $"waiting {args.RetryDelay.TotalMilliseconds}ms");
                    return ValueTask.CompletedTask;
                }
            })
            .Build();
    }

    public async Task<string> GetWeatherAsync(string city, CancellationToken ct = default)
    {
        return await _pipeline.ExecuteAsync(async token =>
        {
            var response = await _httpClient.GetAsync(
                $"/api/weather?city={city}", token);
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadAsStringAsync(token);
        }, ct);
    }
}
```

### Retry Delay แบบ Custom

```csharp
// Custom delay ที่คำนวณเองได้
var retryOptions = new RetryStrategyOptions
{
    MaxRetryAttempts = 5,
    
    // คำนวณ delay แบบ custom
    DelayGenerator = args =>
    {
        // ดึงค่าจาก Retry-After header ถ้ามี
        if (args.Outcome.Result is HttpResponseMessage response &&
            response.Headers.RetryAfter?.Delta is TimeSpan retryAfter)
        {
            return new ValueTask<TimeSpan?>(retryAfter);
        }
        
        // ไม่งั้นใช้ exponential backoff
        var delay = TimeSpan.FromSeconds(Math.Pow(2, args.AttemptNumber));
        return new ValueTask<TimeSpan?>(delay);
    }
};
```

---

## Step 713: Circuit Breaker Pattern (Closed/Open/Half-Open)

### Circuit Breaker คืออะไร?

**Circuit Breaker** ได้รับแรงบันดาลใจจาก "สวิตช์ตัดไฟ" ในระบบไฟฟ้า โดยมีเป้าหมายเพื่อ:

1. **ป้องกัน Cascading Failures**: ถ้า Service ปลายทางล้มเหลว อย่าพยายามเรียกซ้ำไปเรื่อย ๆ
2. **Fast Fail**: ตอบกลับทันทีโดยไม่รอ Timeout
3. **Allow Recovery**: ให้ Service ได้หายใจหายคอ ก่อนลองใหม่

### สามสถานะของ Circuit Breaker

```
   [Closed] ──── ความล้มเหลวถึง threshold ───→ [Open]
      ↑                                              │
      │                                              │
   ทดสอบสำเร็จ                               รอ Duration
      │                                              │
      └──── [Half-Open] ←─── หมดเวลา break ─────────┘
                │
                └── ทดสอบล้มเหลว → กลับไป [Open]
```

- **Closed**: ทำงานปกติ นับความล้มเหลว
- **Open**: หยุดรับ request ทั้งหมด ตอบ BrokenCircuitException ทันที
- **Half-Open**: ทดลองส่ง request บางส่วน เพื่อดูว่า Service ฟื้นตัวแล้วหรือยัง

### Circuit Breaker พื้นฐาน

```csharp
using Polly.CircuitBreaker;

var circuitBreakerOptions = new CircuitBreakerStrategyOptions
{
    // เปิด circuit เมื่อ 50% ของ request ล้มเหลว ใน 10 request ล่าสุด
    FailureRatio = 0.5,
    MinimumThroughput = 10,       // ต้องมีอย่างน้อย 10 request ก่อนวิเคราะห์
    SamplingDuration = TimeSpan.FromSeconds(30), // ช่วงเวลาที่วิเคราะห์
    
    // รอ 1 นาที ก่อนเปลี่ยนเป็น Half-Open
    BreakDuration = TimeSpan.FromMinutes(1),
    
    // Callback เมื่อ Circuit เปิด
    OnOpened = args =>
    {
        Console.WriteLine($"[Circuit] OPENED! Duration: {args.BreakDuration.TotalSeconds}s. " +
                          $"Reason: {args.Outcome.Exception?.Message}");
        return ValueTask.CompletedTask;
    },
    
    // Callback เมื่อ Circuit ปิด (ฟื้นตัวแล้ว)
    OnClosed = args =>
    {
        Console.WriteLine("[Circuit] CLOSED - Service recovered!");
        return ValueTask.CompletedTask;
    },
    
    // Callback เมื่อเข้า Half-Open
    OnHalfOpened = args =>
    {
        Console.WriteLine("[Circuit] HALF-OPEN - Testing service...");
        return ValueTask.CompletedTask;
    }
};

var pipeline = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(circuitBreakerOptions)
    .Build();
```

### จัดการ BrokenCircuitException

```csharp
public class PaymentService
{
    private readonly ResiliencePipeline _pipeline;
    private readonly ILogger<PaymentService> _logger;

    public PaymentService(ILogger<PaymentService> logger)
    {
        _logger = logger;
        _pipeline = BuildPipeline();
    }

    private ResiliencePipeline BuildPipeline()
    {
        return new ResiliencePipelineBuilder()
            .AddCircuitBreaker(new CircuitBreakerStrategyOptions
            {
                FailureRatio = 0.6,
                MinimumThroughput = 5,
                SamplingDuration = TimeSpan.FromSeconds(10),
                BreakDuration = TimeSpan.FromSeconds(30),
                
                ShouldHandle = new PredicateBuilder()
                    .Handle<HttpRequestException>()
                    .Handle<TimeoutRejectedException>()
            })
            .Build();
    }

    public async Task<PaymentResult> ProcessPaymentAsync(Payment payment, CancellationToken ct = default)
    {
        try
        {
            return await _pipeline.ExecuteAsync(async token =>
            {
                // เรียก Payment Gateway
                var response = await CallPaymentGatewayAsync(payment, token);
                return response;
            }, ct);
        }
        catch (BrokenCircuitException ex)
        {
            // Circuit เปิดอยู่ - ตอบกลับทันที ไม่รอ
            _logger.LogWarning("Payment circuit is OPEN. Returning cached/fallback response.");
            return new PaymentResult 
            { 
                Success = false, 
                ErrorCode = "CIRCUIT_OPEN",
                Message = "Payment service temporarily unavailable. Please try again later.",
                RetryAfter = TimeSpan.FromSeconds(30)
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Payment processing failed");
            throw;
        }
    }

    private async Task<PaymentResult> CallPaymentGatewayAsync(Payment payment, CancellationToken ct)
    {
        // Simulate payment gateway call
        await Task.Delay(100, ct);
        return new PaymentResult { Success = true };
    }
}

public record Payment(string OrderId, decimal Amount, string Currency);
public record PaymentResult 
{ 
    public bool Success { get; init; }
    public string? ErrorCode { get; init; }
    public string? Message { get; init; }
    public TimeSpan RetryAfter { get; init; }
}
```

### การตรวจสอบสถานะ Circuit Breaker

```csharp
// ใช้ CircuitBreakerStateProvider เพื่อตรวจสอบสถานะ
var stateProvider = new CircuitBreakerStateProvider();

var pipeline = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,
        MinimumThroughput = 5,
        SamplingDuration = TimeSpan.FromSeconds(10),
        BreakDuration = TimeSpan.FromSeconds(30),
        StateProvider = stateProvider // ลงทะเบียน provider
    })
    .Build();

// ตรวจสอบสถานะ
CircuitBreakerState state = stateProvider.CircuitState;
Console.WriteLine($"Current state: {state}"); // Closed, Open, HalfOpen, Isolated
```

---

## Step 714: Timeout และ Bulkhead Isolation

### Timeout Strategy

Timeout ป้องกันการรอนานเกินไปเมื่อ Service ช้าหรือไม่ตอบสนอง

```csharp
using Polly.Timeout;

// Timeout แบบพื้นฐาน
var timeoutPipeline = new ResiliencePipelineBuilder()
    .AddTimeout(new TimeoutStrategyOptions
    {
        Timeout = TimeSpan.FromSeconds(5),
        
        OnTimeout = args =>
        {
            Console.WriteLine($"[Timeout] Operation exceeded {args.Timeout.TotalSeconds}s");
            return ValueTask.CompletedTask;
        }
    })
    .Build();

// หรือแบบสั้น
var shortPipeline = new ResiliencePipelineBuilder()
    .AddTimeout(TimeSpan.FromSeconds(5))
    .Build();
```

### TimeProvider สำหรับ Testability

Polly v8 รองรับ `TimeProvider` เพื่อให้ทดสอบ Timeout ได้โดยไม่ต้องรอเวลาจริง

```csharp
// Production: ใช้ TimeProvider.System
var productionPipeline = new ResiliencePipelineBuilder()
    .AddTimeout(new TimeoutStrategyOptions
    {
        Timeout = TimeSpan.FromSeconds(30),
        TimeProvider = TimeProvider.System // default
    })
    .Build();

// Testing: ใช้ FakeTimeProvider จาก Microsoft.Extensions.TimeProvider.Testing
// dotnet add package Microsoft.Extensions.TimeProvider.Testing
using Microsoft.Extensions.Time.Testing;

var fakeTime = new FakeTimeProvider();
var testPipeline = new ResiliencePipelineBuilder()
    .AddTimeout(new TimeoutStrategyOptions
    {
        Timeout = TimeSpan.FromSeconds(30),
        TimeProvider = fakeTime
    })
    .Build();

// ใน test: เดิน time ไปข้างหน้าเพื่อ trigger timeout
// fakeTime.Advance(TimeSpan.FromSeconds(31));
```

### Bulkhead Isolation

**Bulkhead** (กันน้ำเรือ) คือการแบ่ง Resource Pool เพื่อป้องกัน Service หนึ่งกิน Resource ของ Service อื่นทั้งหมด

ใน Polly v8, Bulkhead ถูกแทนที่ด้วย **Rate Limiter** (Step 716) แต่แนวคิดคือการจำกัด Concurrency

```csharp
using System.Threading.RateLimiting;
using Polly.RateLimiter;

// จำกัด Concurrency สูงสุด 10 requests พร้อมกัน
// รอ queue ได้อีก 5 requests
var bulkheadPipeline = new ResiliencePipelineBuilder()
    .AddConcurrencyLimiter(
        permitLimit: 10,    // สูงสุด 10 concurrent requests
        queueLimit: 5       // รอ queue ได้ 5 requests
    )
    .Build();

try
{
    await bulkheadPipeline.ExecuteAsync(async ct =>
    {
        await DoWorkAsync(ct);
    }, cancellationToken);
}
catch (RateLimiterRejectedException ex)
{
    // Queue เต็ม - request ถูกปฏิเสธ
    Console.WriteLine($"Too many concurrent requests! RetryAfter: {ex.RetryAfter}");
}
```

### ตัวอย่าง: Timeout + Circuit Breaker ร่วมกัน

```csharp
public class DatabaseService
{
    private readonly ResiliencePipeline _pipeline;

    public DatabaseService()
    {
        _pipeline = new ResiliencePipelineBuilder()
            // Timeout ต้องมาก่อน (จะ wrap รอบ retry ด้วย)
            .AddTimeout(TimeSpan.FromSeconds(3))
            
            // Retry เมื่อ timeout
            .AddRetry(new RetryStrategyOptions
            {
                MaxRetryAttempts = 2,
                Delay = TimeSpan.FromMilliseconds(500),
                ShouldHandle = new PredicateBuilder()
                    .Handle<TimeoutRejectedException>()
            })
            
            // Circuit Breaker ป้องกัน Database overload
            .AddCircuitBreaker(new CircuitBreakerStrategyOptions
            {
                FailureRatio = 0.8,
                MinimumThroughput = 10,
                SamplingDuration = TimeSpan.FromSeconds(30),
                BreakDuration = TimeSpan.FromSeconds(60)
            })
            .Build();
    }

    public async Task<T?> QueryAsync<T>(string sql, CancellationToken ct = default)
    {
        return await _pipeline.ExecuteAsync(async token =>
        {
            // Execute database query
            await Task.Delay(50, token); // Simulate query
            return default(T);
        }, ct);
    }
}
```

---

## Step 715: Fallback และ Hedging Strategies

### Fallback Strategy

**Fallback** คือการตอบกลับด้วยค่า default หรือค่าจาก Cache เมื่อ primary operation ล้มเหลว

```csharp
using Polly.Fallback;

// Fallback พื้นฐาน
var fallbackPipeline = new ResiliencePipelineBuilder<UserProfile>()
    .AddFallback(new FallbackStrategyOptions<UserProfile>
    {
        // กำหนดว่าจะ fallback เมื่อไหร่
        ShouldHandle = new PredicateBuilder<UserProfile>()
            .Handle<HttpRequestException>()
            .Handle<TimeoutRejectedException>()
            .HandleResult(profile => profile == null),
        
        // ค่าที่จะ return เมื่อ fallback
        FallbackAction = args =>
        {
            var defaultProfile = new UserProfile 
            { 
                UserId = "unknown",
                DisplayName = "Guest User",
                IsFromCache = true
            };
            return new ValueTask<Outcome<UserProfile>>(
                Outcome.FromResult(defaultProfile));
        },
        
        OnFallback = args =>
        {
            Console.WriteLine($"[Fallback] Using default profile. " +
                              $"Reason: {args.Outcome.Exception?.Message ?? "null result"}");
            return ValueTask.CompletedTask;
        }
    })
    .Build();
```

### Fallback with Cache

```csharp
public class ProductService
{
    private readonly ResiliencePipeline<Product?> _pipeline;
    private readonly IMemoryCache _cache;
    private readonly HttpClient _httpClient;

    public ProductService(IMemoryCache cache, HttpClient httpClient)
    {
        _cache = cache;
        _httpClient = httpClient;
        _pipeline = BuildPipeline();
    }

    private ResiliencePipeline<Product?> BuildPipeline()
    {
        return new ResiliencePipelineBuilder<Product?>()
            // Fallback ต้องมาก่อนเสมอ (outermost)
            .AddFallback(new FallbackStrategyOptions<Product?>
            {
                ShouldHandle = new PredicateBuilder<Product?>()
                    .Handle<HttpRequestException>()
                    .Handle<BrokenCircuitException>(),
                
                FallbackAction = args =>
                {
                    // ลองดึงจาก Cache
                    if (args.Context.Properties.TryGetValue(
                        new ResiliencePropertyKey<string>("productId"), 
                        out var productId) &&
                        _cache.TryGetValue($"product:{productId}", out Product? cached))
                    {
                        Console.WriteLine($"[Fallback] Returning cached product: {productId}");
                        return new ValueTask<Outcome<Product?>>(Outcome.FromResult(cached));
                    }
                    
                    // ไม่มี cache - return null
                    Console.WriteLine("[Fallback] No cache available, returning null");
                    return new ValueTask<Outcome<Product?>>(Outcome.FromResult<Product?>(null));
                }
            })
            
            // Retry
            .AddRetry(new RetryStrategyOptions<Product?>
            {
                MaxRetryAttempts = 2,
                Delay = TimeSpan.FromSeconds(1),
                BackoffType = DelayBackoffType.Exponential
            })
            
            // Circuit Breaker
            .AddCircuitBreaker(new CircuitBreakerStrategyOptions<Product?>
            {
                FailureRatio = 0.5,
                MinimumThroughput = 5,
                SamplingDuration = TimeSpan.FromSeconds(20),
                BreakDuration = TimeSpan.FromSeconds(30)
            })
            .Build();
    }

    public async Task<Product?> GetProductAsync(string productId, CancellationToken ct = default)
    {
        var context = ResilienceContextPool.Shared.Get(ct);
        context.Properties.Set(new ResiliencePropertyKey<string>("productId"), productId);

        try
        {
            return await _pipeline.ExecuteAsync(async ctx =>
            {
                var response = await _httpClient.GetAsync($"/products/{productId}", ctx.CancellationToken);
                
                if (response.IsSuccessStatusCode)
                {
                    var product = await response.Content.ReadFromJsonAsync<Product>(ctx.CancellationToken);
                    // บันทึก cache ถ้าสำเร็จ
                    if (product != null)
                        _cache.Set($"product:{productId}", product, TimeSpan.FromMinutes(5));
                    return product;
                }
                
                throw new HttpRequestException($"HTTP {response.StatusCode}");
            }, context);
        }
        finally
        {
            ResilienceContextPool.Shared.Return(context);
        }
    }
}

public record Product(string Id, string Name, decimal Price, bool IsFromCache = false);
public record UserProfile(string UserId, string DisplayName, bool IsFromCache = false);
```

### Hedging Strategy

**Hedging** (การป้องกันความเสี่ยง) คือการส่ง request หลายอัน พร้อมกัน หรือ ทยอยส่ง เพื่อรับ response แรกที่สำเร็จ

```csharp
using Polly.Hedging;

// Hedging: ถ้า request แรกช้า ให้ส่ง request ที่ 2 โดยไม่รอ
var hedgingPipeline = new ResiliencePipelineBuilder<string>()
    .AddHedging(new HedgingStrategyOptions<string>
    {
        // จำนวน hedged attempts สูงสุด
        MaxHedgedAttempts = 2,
        
        // รอ 500ms ก่อนส่ง hedged request
        Delay = TimeSpan.FromMilliseconds(500),
        
        // Callback สำหรับ hedged request
        ActionGenerator = args =>
        {
            // ส่ง request ไปยัง backup endpoint
            return () => new ValueTask<Outcome<string>>(
                Outcome.FromResult("hedged result"));
        }
    })
    .Build();
```

### Parallel Hedging (ส่งพร้อมกัน)

```csharp
// ส่งไปยัง 3 replica พร้อมกัน รับ response ที่เร็วที่สุด
var parallelHedgingOptions = new HedgingStrategyOptions<string>
{
    MaxHedgedAttempts = 3,
    Delay = TimeSpan.Zero, // ส่งพร้อมกันทันที
    
    ShouldHandle = new PredicateBuilder<string>()
        .Handle<Exception>(),
    
    ActionGenerator = args =>
    {
        // ใช้ replica URL ที่ต่างกัน
        var endpoints = new[] 
        { 
            "https://replica1.api.com", 
            "https://replica2.api.com",
            "https://replica3.api.com"
        };
        var endpoint = endpoints[args.AttemptNumber % endpoints.Length];
        
        return async () =>
        {
            using var client = new HttpClient();
            var result = await client.GetStringAsync(endpoint);
            return Outcome.FromResult(result);
        };
    }
};
```

---

## Step 716: Rate Limiting และ Throttling

### ทำไมต้องมี Rate Limiting?

Rate Limiting ป้องกัน:
- **Overloading**: กัน Service จาก request มากเกินไป
- **Cost Control**: ควบคุมค่าใช้จ่าย API ภายนอก
- **Fair Usage**: แบ่ง Resource อย่างยุติธรรม

### Rate Limiter ประเภทต่าง ๆ

```csharp
using System.Threading.RateLimiting;
using Polly.RateLimiter;

// 1. Fixed Window: 100 requests ต่อ 1 นาที
var fixedWindowPipeline = new ResiliencePipelineBuilder()
    .AddRateLimiter(new RateLimiterStrategyOptions
    {
        RateLimiter = args => new FixedWindowRateLimiter(new FixedWindowRateLimiterOptions
        {
            PermitLimit = 100,
            Window = TimeSpan.FromMinutes(1),
            QueueLimit = 0 // ไม่มี queue, ปฏิเสธทันที
        }).AttemptAcquire()
    })
    .Build();

// 2. Sliding Window: 100 requests ต่อ 1 นาทีแบบ sliding
var slidingWindowPipeline = new ResiliencePipelineBuilder()
    .AddRateLimiter(new RateLimiterStrategyOptions
    {
        RateLimiter = args => new SlidingWindowRateLimiter(new SlidingWindowRateLimiterOptions
        {
            PermitLimit = 100,
            Window = TimeSpan.FromMinutes(1),
            SegmentsPerWindow = 6 // แบ่งเป็น 6 segment (10s ต่อ segment)
        }).AttemptAcquire()
    })
    .Build();

// 3. Token Bucket: รองรับ burst
var tokenBucketPipeline = new ResiliencePipelineBuilder()
    .AddRateLimiter(new RateLimiterStrategyOptions
    {
        RateLimiter = args => new TokenBucketRateLimiter(new TokenBucketRateLimiterOptions
        {
            TokenLimit = 100,           // จำนวน token สูงสุด
            ReplenishmentPeriod = TimeSpan.FromSeconds(1),
            TokensPerPeriod = 10        // เพิ่ม 10 token ต่อวินาที
        }).AttemptAcquire()
    })
    .Build();
```

### Concurrency Limiter (Bulkhead Pattern)

```csharp
// จำกัด concurrent requests
var concurrencyPipeline = new ResiliencePipelineBuilder()
    .AddConcurrencyLimiter(new ConcurrencyLimiterOptions
    {
        PermitLimit = 20, // สูงสุด 20 concurrent requests
        QueueLimit = 10   // รอ queue ได้ 10
    })
    .Build();
```

### API Client with Rate Limiting

```csharp
public class ExternalApiClient
{
    private readonly HttpClient _httpClient;
    private readonly ResiliencePipeline<HttpResponseMessage> _pipeline;
    private readonly ILogger<ExternalApiClient> _logger;

    public ExternalApiClient(HttpClient httpClient, ILogger<ExternalApiClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
        _pipeline = BuildPipeline();
    }

    private ResiliencePipeline<HttpResponseMessage> BuildPipeline()
    {
        // Rate Limiter: 1000 requests ต่อชั่วโมง
        var rateLimiter = new SlidingWindowRateLimiter(new SlidingWindowRateLimiterOptions
        {
            PermitLimit = 1000,
            Window = TimeSpan.FromHours(1),
            SegmentsPerWindow = 60
        });

        return new ResiliencePipelineBuilder<HttpResponseMessage>()
            // Rate Limiting มาก่อน
            .AddRateLimiter(new RateLimiterStrategyOptions<HttpResponseMessage>
            {
                DefaultRateLimiterOptions = null,
                RateLimiter = args =>
                {
                    return rateLimiter.AttemptAcquire();
                },
                OnRejected = args =>
                {
                    _logger.LogWarning("Rate limit exceeded. RetryAfter: {RetryAfter}", 
                        args.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter) 
                            ? retryAfter 
                            : "unknown");
                    return ValueTask.CompletedTask;
                }
            })
            
            // Retry สำหรับ 429 Too Many Requests
            .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
            {
                MaxRetryAttempts = 3,
                ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                    .HandleResult(r => r.StatusCode == System.Net.HttpStatusCode.TooManyRequests),
                
                DelayGenerator = args =>
                {
                    if (args.Outcome.Result?.Headers.RetryAfter?.Delta is TimeSpan delay)
                        return new ValueTask<TimeSpan?>(delay);
                    return new ValueTask<TimeSpan?>(TimeSpan.FromSeconds(60));
                }
            })
            .Build();
    }

    public async Task<T?> GetAsync<T>(string path, CancellationToken ct = default)
    {
        var response = await _pipeline.ExecuteAsync(async token =>
            await _httpClient.GetAsync(path, token), ct);
        
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>(ct);
    }
}
```

---

## Step 717: Polly v8 ResiliencePipelineBuilder

### Pipeline Execution Order

ลำดับการทำงานใน pipeline มีความสำคัญมาก: strategy ที่เพิ่มก่อนจะทำงาน "รอบนอก" (wrap)

```
Request → [Timeout] → [Retry] → [CircuitBreaker] → [Actual Work]
Response ← [Timeout] ← [Retry] ← [CircuitBreaker] ← [Actual Work]
```

**แนวทางปฏิบัติ (Best Practices):**
1. **Timeout** - รอบนอกสุด (จำกัดเวลาทั้ง pipeline)
2. **Retry** - รอบต่อมา (ลองซ้ำเมื่อ inner ล้มเหลว)
3. **Circuit Breaker** - รอบถัดมา (ป้องกัน Service)
4. **Rate Limiter** - อยู่ด้านใน
5. **Actual Operation** - core

```csharp
// Pipeline ที่สมบูรณ์และ order ถูกต้อง
var completePipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    // 1. Overall timeout (รอบนอกสุด)
    .AddTimeout(TimeSpan.FromSeconds(30))
    
    // 2. Retry with jitter
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        MaxRetryAttempts = 3,
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
        Delay = TimeSpan.FromMilliseconds(200),
        MaxDelay = TimeSpan.FromSeconds(10),
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .Handle<TimeoutRejectedException>()
            .HandleResult(r => (int)r.StatusCode >= 500)
    })
    
    // 3. Circuit Breaker
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
    {
        FailureRatio = 0.5,
        MinimumThroughput = 8,
        SamplingDuration = TimeSpan.FromSeconds(30),
        BreakDuration = TimeSpan.FromSeconds(60),
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .HandleResult(r => (int)r.StatusCode >= 500)
    })
    
    // 4. Rate Limiter
    .AddConcurrencyLimiter(permitLimit: 50, queueLimit: 10)
    
    // 5. Per-attempt timeout
    .AddTimeout(TimeSpan.FromSeconds(5))
    
    .Build();
```

### ResiliencePropertyKey สำหรับ Context Passing

```csharp
// กำหนด typed keys
public static class ResilienceKeys
{
    public static readonly ResiliencePropertyKey<string> RequestId = 
        new("requestId");
    
    public static readonly ResiliencePropertyKey<string> UserId = 
        new("userId");
    
    public static readonly ResiliencePropertyKey<string> OperationName = 
        new("operationName");
    
    public static readonly ResiliencePropertyKey<DateTime> RequestStartTime = 
        new("requestStartTime");
}

// การใช้งาน
var context = ResilienceContextPool.Shared.Get(cancellationToken);

try
{
    context.Properties.Set(ResilienceKeys.RequestId, Guid.NewGuid().ToString());
    context.Properties.Set(ResilienceKeys.UserId, "user-123");
    context.Properties.Set(ResilienceKeys.OperationName, "GetUserProfile");
    context.Properties.Set(ResilienceKeys.RequestStartTime, DateTime.UtcNow);

    await pipeline.ExecuteAsync(async ctx =>
    {
        var requestId = ctx.Properties.GetValue(ResilienceKeys.RequestId, "unknown");
        var userId = ctx.Properties.GetValue(ResilienceKeys.UserId, "anonymous");
        
        Console.WriteLine($"[{requestId}] Processing request for user: {userId}");
        
        // ... logic
    }, context);
}
finally
{
    ResilienceContextPool.Shared.Return(context);
}
```

### Telemetry และ Observability

```csharp
// เปิดใช้ telemetry
var pipeline = new ResiliencePipelineBuilder()
    .ConfigureTelemetry(new TelemetryOptions
    {
        // Custom event listener
        TelemetryListeners = 
        {
            new CustomTelemetryListener()
        }
    })
    .AddRetry(new RetryStrategyOptions { MaxRetryAttempts = 3 })
    .Build();

public class CustomTelemetryListener : TelemetryListener
{
    public override void Write<TResult, TArgs>(in TelemetryEventArguments<TResult, TArgs> args)
    {
        if (args.Event.EventName == "ExecutionAttempt")
        {
            Console.WriteLine($"[Telemetry] Attempt: {args.Context.OperationKey}, " +
                              $"Result: {(args.Outcome.Exception != null ? "failed" : "success")}");
        }
    }
}
```

---

## Step 718: HttpClientFactory กับ Polly Integration

### AddHttpClient + AddResiliencePipeline

```csharp
// Program.cs หรือ Startup.cs
using Microsoft.Extensions.Http.Resilience;

var builder = WebApplication.CreateBuilder(args);

// วิธีที่ 1: AddStandardResilienceHandler (มาพร้อม defaults ที่ดี)
builder.Services.AddHttpClient<UserApiClient>(client =>
{
    client.BaseAddress = new Uri("https://users.api.internal/");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
})
.AddStandardResilienceHandler(); // Retry + CircuitBreaker + Timeout อัตโนมัติ

// วิธีที่ 2: กำหนด pipeline เอง
builder.Services.AddHttpClient<ProductApiClient>(client =>
{
    client.BaseAddress = new Uri("https://products.api.internal/");
})
.AddResilienceHandler("product-pipeline", pipelineBuilder =>
{
    pipelineBuilder
        .AddTimeout(TimeSpan.FromSeconds(15))
        .AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true
        })
        .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            SamplingDuration = TimeSpan.FromSeconds(30),
            BreakDuration = TimeSpan.FromSeconds(60),
            FailureRatio = 0.5,
            MinimumThroughput = 5
        });
});

// วิธีที่ 3: Named Pipeline ที่ใช้ร่วมกัน
builder.Services.AddResiliencePipeline("shared-http-pipeline", pipelineBuilder =>
{
    pipelineBuilder
        .AddRetry(new RetryStrategyOptions { MaxRetryAttempts = 3 })
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            MinimumThroughput = 10,
            SamplingDuration = TimeSpan.FromSeconds(30),
            BreakDuration = TimeSpan.FromSeconds(30)
        })
        .AddTimeout(TimeSpan.FromSeconds(10));
});

var app = builder.Build();
```

### AddStandardResilienceHandler ทำอะไร?

```csharp
// AddStandardResilienceHandler เทียบเท่ากับ:
.AddResilienceHandler("standard", builder =>
{
    builder
        // 1. Rate limiting: 1000 req/min
        .AddRateLimiter(new HttpRateLimiterStrategyOptions())
        
        // 2. Total timeout: 30s
        .AddTotalRequestTimeout(new HttpTimeoutStrategyOptions
        {
            Timeout = TimeSpan.FromSeconds(30)
        })
        
        // 3. Retry: 3 attempts, exponential backoff
        .AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            Delay = TimeSpan.FromMilliseconds(200)
        })
        
        // 4. Circuit Breaker
        .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            SamplingDuration = TimeSpan.FromSeconds(30),
            BreakDuration = TimeSpan.FromSeconds(5),
            FailureRatio = 0.1,
            MinimumThroughput = 100
        })
        
        // 5. Per-attempt timeout: 10s
        .AddAttemptTimeout(new HttpTimeoutStrategyOptions
        {
            Timeout = TimeSpan.FromSeconds(10)
        });
})
```

### การใช้งาน HttpClient ใน Service

```csharp
public class UserApiClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<UserApiClient> _logger;

    // Typed HttpClient - inject ผ่าน DI
    public UserApiClient(HttpClient httpClient, ILogger<UserApiClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<UserDto?> GetUserAsync(int userId, CancellationToken ct = default)
    {
        try
        {
            // Polly pipeline จะทำงานอัตโนมัติ (ผ่าน DelegatingHandler)
            var response = await _httpClient.GetAsync($"/users/{userId}", ct);
            
            if (response.StatusCode == System.Net.HttpStatusCode.NotFound)
                return null;
            
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<UserDto>(ct);
        }
        catch (BrokenCircuitException)
        {
            _logger.LogWarning("User API circuit is open for userId: {UserId}", userId);
            throw;
        }
    }

    public async Task<IReadOnlyList<UserDto>> SearchUsersAsync(
        string query, CancellationToken ct = default)
    {
        var response = await _httpClient.GetAsync(
            $"/users?search={Uri.EscapeDataString(query)}", ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<List<UserDto>>(ct) 
               ?? new List<UserDto>();
    }
}

public record UserDto(int Id, string Name, string Email);
```

### Configure StandardResilienceHandler Options

```csharp
// ปรับแต่ง StandardResilienceHandler ผ่าน appsettings.json
builder.Services.AddHttpClient<OrderApiClient>(client =>
{
    client.BaseAddress = new Uri("https://orders.api.internal/");
})
.AddStandardResilienceHandler()
.Configure(options =>
{
    // Override retry options
    options.Retry.MaxRetryAttempts = 5;
    options.Retry.Delay = TimeSpan.FromSeconds(2);
    
    // Override timeout
    options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(60);
    options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(20);
    
    // Override circuit breaker
    options.CircuitBreaker.FailureRatio = 0.3;
    options.CircuitBreaker.MinimumThroughput = 20;
});
```

---

## Step 719: Custom Resilience Strategies

### สร้าง Custom Strategy

บางครั้ง built-in strategies ไม่เพียงพอ เราสามารถสร้าง custom strategy ได้

```csharp
using Polly.Utils;

// 1. กำหนด Options สำหรับ strategy
public class RetryOnSpecificErrorOptions
{
    public int MaxRetryAttempts { get; set; } = 3;
    public TimeSpan Delay { get; set; } = TimeSpan.FromSeconds(1);
    public required Func<Exception, bool> ShouldRetry { get; set; }
}

// 2. สร้าง Strategy class
public class RetryOnSpecificErrorStrategy<T> : ResilienceStrategy<T>
{
    private readonly RetryOnSpecificErrorOptions _options;
    private readonly ILogger _logger;

    public RetryOnSpecificErrorStrategy(
        RetryOnSpecificErrorOptions options,
        ILogger logger)
    {
        _options = options;
        _logger = logger;
    }

    protected override async ValueTask<Outcome<T>> ExecuteCore<TState>(
        Func<ResilienceContext, TState, ValueTask<Outcome<T>>> callback,
        ResilienceContext context,
        TState state)
    {
        int attempt = 0;
        
        while (true)
        {
            var outcome = await callback(context, state);
            
            // ถ้าสำเร็จหรือ exception ไม่ตรง condition
            if (outcome.Exception == null || 
                attempt >= _options.MaxRetryAttempts ||
                !_options.ShouldRetry(outcome.Exception))
            {
                return outcome;
            }
            
            attempt++;
            _logger.LogWarning(
                "Custom retry #{Attempt}/{Max}: {Error}", 
                attempt, _options.MaxRetryAttempts, outcome.Exception.Message);
            
            await Task.Delay(_options.Delay, context.CancellationToken);
        }
    }
}
```

### Extension Method สำหรับ Custom Strategy

```csharp
// Extension method เพื่อใช้งานได้สะดวก
public static class ResiliencePipelineBuilderExtensions
{
    public static ResiliencePipelineBuilder<T> AddRetryOnSpecificError<T>(
        this ResiliencePipelineBuilder<T> builder,
        RetryOnSpecificErrorOptions options,
        ILogger logger)
    {
        return builder.AddStrategy(
            context => new RetryOnSpecificErrorStrategy<T>(options, logger),
            options);
    }
}

// การใช้งาน
var pipeline = new ResiliencePipelineBuilder<string>()
    .AddRetryOnSpecificError(
        new RetryOnSpecificErrorOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromSeconds(2),
            ShouldRetry = ex => ex is HttpRequestException || ex is IOException
        },
        logger)
    .Build();
```

### Decorator Pattern สำหรับ Resilience

```csharp
// แทนที่จะใช้ Polly โดยตรง ใช้ Decorator pattern
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<IEnumerable<Product>> GetAllAsync(CancellationToken ct = default);
}

public class ProductRepository : IProductRepository
{
    private readonly DbContext _context;
    
    public ProductRepository(DbContext context)
    {
        _context = context;
    }
    
    public async Task<Product?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        return await _context.Set<Product>().FindAsync(new object[] { id }, ct);
    }
    
    public async Task<IEnumerable<Product>> GetAllAsync(CancellationToken ct = default)
    {
        return await _context.Set<Product>().ToListAsync(ct);
    }
}

// Resilient Decorator
public class ResilientProductRepository : IProductRepository
{
    private readonly IProductRepository _inner;
    private readonly ResiliencePipeline _pipeline;
    private readonly ILogger<ResilientProductRepository> _logger;

    public ResilientProductRepository(
        IProductRepository inner, 
        ILogger<ResilientProductRepository> logger)
    {
        _inner = inner;
        _logger = logger;
        _pipeline = BuildPipeline();
    }

    private ResiliencePipeline BuildPipeline()
    {
        return new ResiliencePipelineBuilder()
            .AddRetry(new RetryStrategyOptions
            {
                MaxRetryAttempts = 3,
                Delay = TimeSpan.FromMilliseconds(200),
                BackoffType = DelayBackoffType.Exponential,
                UseJitter = true,
                ShouldHandle = new PredicateBuilder()
                    .Handle<DbException>()
                    .Handle<TimeoutException>(),
                OnRetry = args =>
                {
                    _logger.LogWarning("DB retry #{Attempt}", args.AttemptNumber + 1);
                    return ValueTask.CompletedTask;
                }
            })
            .AddTimeout(TimeSpan.FromSeconds(10))
            .Build();
    }

    public Task<Product?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        return _pipeline.ExecuteAsync(
            token => _inner.GetByIdAsync(id, token).AsValueTask(),
            ct).AsTask();
    }

    public Task<IEnumerable<Product>> GetAllAsync(CancellationToken ct = default)
    {
        return _pipeline.ExecuteAsync(
            token => _inner.GetAllAsync(token).AsValueTask(),
            ct).AsTask();
    }
}

// ลงทะเบียนใน DI
services.AddScoped<ProductRepository>();
services.AddScoped<IProductRepository>(sp =>
    new ResilientProductRepository(
        sp.GetRequiredService<ProductRepository>(),
        sp.GetRequiredService<ILogger<ResilientProductRepository>>()));
```

---

## Step 720: Resilience ใน Microservices (Saga Compensation)

### Saga Pattern คืออะไร?

**Saga Pattern** ใช้สำหรับจัดการ **Distributed Transactions** ใน Microservices โดยแบ่งเป็น:

1. **Choreography Saga**: แต่ละ Service publish events และ react ต่อ events จาก Service อื่น
2. **Orchestration Saga**: มี Saga Orchestrator ควบคุมลำดับขั้นตอน

### Compensation Transaction

เมื่อ step หนึ่งล้มเหลว ต้อง **Compensate** (ยกเลิก) step ที่ทำไปแล้ว

```
Step 1: Reserve Inventory  →  Compensate 1: Release Inventory
Step 2: Process Payment    →  Compensate 2: Refund Payment  
Step 3: Create Order       →  Compensate 3: Cancel Order
Step 4: Send Notification  →  (idempotent, ไม่ต้อง compensate)
```

### ตัวอย่าง Orchestration Saga

```csharp
// Saga state
public class OrderSagaState
{
    public string OrderId { get; set; } = Guid.NewGuid().ToString();
    public string? PaymentId { get; set; }
    public bool InventoryReserved { get; set; }
    public bool PaymentProcessed { get; set; }
    public bool OrderCreated { get; set; }
    public SagaStatus Status { get; set; } = SagaStatus.Started;
}

public enum SagaStatus { Started, Completed, Compensating, Compensated, Failed }

// Saga interfaces
public interface IInventoryService
{
    Task<string> ReserveAsync(string orderId, IEnumerable<OrderItem> items, CancellationToken ct);
    Task ReleaseAsync(string reservationId, CancellationToken ct);
}

public interface IPaymentService
{
    Task<string> ChargeAsync(string orderId, decimal amount, string paymentMethod, CancellationToken ct);
    Task RefundAsync(string paymentId, CancellationToken ct);
}

public interface IOrderService
{
    Task<Order> CreateAsync(OrderSagaState state, CancellationToken ct);
    Task CancelAsync(string orderId, CancellationToken ct);
}
```

### Saga Orchestrator with Resilience

```csharp
public class CreateOrderSaga
{
    private readonly IInventoryService _inventory;
    private readonly IPaymentService _payment;
    private readonly IOrderService _orders;
    private readonly ILogger<CreateOrderSaga> _logger;
    private readonly ResiliencePipeline _pipeline;

    public CreateOrderSaga(
        IInventoryService inventory,
        IPaymentService payment,
        IOrderService orders,
        ILogger<CreateOrderSaga> logger)
    {
        _inventory = inventory;
        _payment = payment;
        _orders = orders;
        _logger = logger;
        _pipeline = BuildSagaPipeline();
    }

    private ResiliencePipeline BuildSagaPipeline()
    {
        // Saga steps ต้องมี Retry แต่ไม่ควรมี Circuit Breaker แบบเดียวกัน
        return new ResiliencePipelineBuilder()
            .AddRetry(new RetryStrategyOptions
            {
                MaxRetryAttempts = 3,
                Delay = TimeSpan.FromSeconds(1),
                BackoffType = DelayBackoffType.Exponential,
                UseJitter = true,
                ShouldHandle = new PredicateBuilder()
                    .Handle<HttpRequestException>()
                    .Handle<TimeoutRejectedException>()
                    // ไม่ retry เมื่อ business exception
                    // Handle<BusinessException> ไม่ควรมี
            })
            .AddTimeout(TimeSpan.FromSeconds(15))
            .Build();
    }

    public async Task<OrderResult> ExecuteAsync(CreateOrderCommand command, CancellationToken ct = default)
    {
        var state = new OrderSagaState();
        
        try
        {
            _logger.LogInformation("Starting CreateOrder saga: {OrderId}", state.OrderId);

            // Step 1: Reserve Inventory
            var reservationId = await _pipeline.ExecuteAsync(async token =>
                await _inventory.ReserveAsync(state.OrderId, command.Items, token), ct);
            
            state.InventoryReserved = true;
            _logger.LogInformation("Step 1 complete: Inventory reserved {ReservationId}", reservationId);

            // Step 2: Process Payment
            var paymentId = await _pipeline.ExecuteAsync(async token =>
                await _payment.ChargeAsync(
                    state.OrderId, command.TotalAmount, command.PaymentMethod, token), ct);
            
            state.PaymentId = paymentId;
            state.PaymentProcessed = true;
            _logger.LogInformation("Step 2 complete: Payment processed {PaymentId}", paymentId);

            // Step 3: Create Order Record
            var order = await _pipeline.ExecuteAsync(async token =>
                await _orders.CreateAsync(state, token), ct);
            
            state.OrderCreated = true;
            state.Status = SagaStatus.Completed;
            _logger.LogInformation("Saga completed: Order {OrderId} created", state.OrderId);

            return new OrderResult(Success: true, OrderId: state.OrderId, Order: order);
        }
        catch (Exception ex) when (ex is not OperationCanceledException)
        {
            _logger.LogError(ex, "Saga failed at state: {@State}", state);
            
            // Compensate ใน reverse order
            await CompensateAsync(state, ct);
            
            return new OrderResult(Success: false, OrderId: state.OrderId, Error: ex.Message);
        }
    }

    private async Task CompensateAsync(OrderSagaState state, CancellationToken ct)
    {
        state.Status = SagaStatus.Compensating;
        _logger.LogWarning("Starting compensation for saga: {OrderId}", state.OrderId);

        // Compensation pipeline - ต้องพยายามให้ได้
        var compensationPipeline = new ResiliencePipelineBuilder()
            .AddRetry(new RetryStrategyOptions
            {
                MaxRetryAttempts = 5, // retry มากขึ้นสำหรับ compensation
                Delay = TimeSpan.FromSeconds(2),
                BackoffType = DelayBackoffType.Exponential
            })
            .Build();

        // Compensate Step 2: Refund Payment
        if (state.PaymentProcessed && state.PaymentId != null)
        {
            try
            {
                await compensationPipeline.ExecuteAsync(async token =>
                    await _payment.RefundAsync(state.PaymentId, token), ct);
                _logger.LogInformation("Compensation 2: Payment refunded");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "CRITICAL: Failed to refund payment {PaymentId}! Manual intervention required!", 
                    state.PaymentId);
                // บันทึกลง Dead Letter Queue สำหรับ manual processing
                await PublishToDeadLetterQueueAsync(state, "RefundFailed", ex);
            }
        }

        // Compensate Step 1: Release Inventory
        if (state.InventoryReserved)
        {
            try
            {
                await compensationPipeline.ExecuteAsync(async token =>
                    await _inventory.ReleaseAsync(state.OrderId, token), ct);
                _logger.LogInformation("Compensation 1: Inventory released");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to release inventory for {OrderId}", state.OrderId);
            }
        }

        state.Status = SagaStatus.Compensated;
        _logger.LogWarning("Saga compensation complete: {OrderId}", state.OrderId);
    }

    private Task PublishToDeadLetterQueueAsync(OrderSagaState state, string reason, Exception ex)
    {
        // Publish to message broker for manual processing
        _logger.LogCritical("DLQ: {Reason} for OrderId: {OrderId}", reason, state.OrderId);
        return Task.CompletedTask;
    }
}

public record CreateOrderCommand(
    string UserId,
    IEnumerable<OrderItem> Items,
    decimal TotalAmount,
    string PaymentMethod);

public record OrderItem(string ProductId, int Quantity, decimal Price);
public record Order(string Id, string Status);
public record OrderResult(bool Success, string OrderId, Order? Order = null, string? Error = null);
```

### Idempotency ใน Saga

```csharp
// ทุก step ต้องเป็น Idempotent เพราะ Retry อาจเรียกซ้ำ
public class IdempotentPaymentService : IPaymentService
{
    private readonly IDistributedCache _cache;
    private readonly IPaymentGateway _gateway;

    public async Task<string> ChargeAsync(
        string orderId, decimal amount, string paymentMethod, CancellationToken ct)
    {
        // ตรวจสอบว่าเคยชำระแล้วหรือยัง (Idempotency key = orderId)
        var cacheKey = $"payment:charge:{orderId}";
        var cachedPaymentId = await _cache.GetStringAsync(cacheKey, ct);
        
        if (cachedPaymentId != null)
        {
            // ชำระแล้ว - return existing paymentId
            return cachedPaymentId;
        }

        // ชำระจริง
        var paymentId = await _gateway.ChargeAsync(amount, paymentMethod, orderId, ct);
        
        // บันทึก cache
        await _cache.SetStringAsync(cacheKey, paymentId, 
            new DistributedCacheEntryOptions 
            { 
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24)
            }, ct);
        
        return paymentId;
    }

    public async Task RefundAsync(string paymentId, CancellationToken ct)
    {
        var cacheKey = $"payment:refund:{paymentId}";
        var alreadyRefunded = await _cache.GetStringAsync(cacheKey, ct);
        
        if (alreadyRefunded != null) return; // Idempotent: ไม่ refund ซ้ำ
        
        await _gateway.RefundAsync(paymentId, ct);
        await _cache.SetStringAsync(cacheKey, "done",
            new DistributedCacheEntryOptions 
            { 
                AbsoluteExpirationRelativeToNow = TimeSpan.FromDays(30)
            }, ct);
    }
}
```

---

## Testing Resilience Policies

### Unit Testing ด้วย Polly

```csharp
using Xunit;
using Polly;
using Polly.Testing;

public class ResiliencePipelineTests
{
    [Fact]
    public async Task Retry_ShouldRetryOnTransientFailure()
    {
        // Arrange
        int callCount = 0;
        
        var pipeline = new ResiliencePipelineBuilder()
            .AddRetry(new RetryStrategyOptions
            {
                MaxRetryAttempts = 3,
                Delay = TimeSpan.Zero // ไม่รอเพื่อให้ test เร็ว
            })
            .Build();

        // Act
        await pipeline.ExecuteAsync(async _ =>
        {
            callCount++;
            if (callCount < 3) 
                throw new HttpRequestException("Transient error");
        });

        // Assert
        Assert.Equal(3, callCount); // เรียก 3 ครั้ง (1 + 2 retry)
    }

    [Fact]
    public async Task CircuitBreaker_ShouldOpenAfterThreshold()
    {
        // Arrange
        var pipeline = new ResiliencePipelineBuilder()
            .AddCircuitBreaker(new CircuitBreakerStrategyOptions
            {
                FailureRatio = 1.0,
                MinimumThroughput = 3,
                SamplingDuration = TimeSpan.FromSeconds(60),
                BreakDuration = TimeSpan.FromSeconds(60)
            })
            .Build();

        // ทำให้ circuit เปิด
        for (int i = 0; i < 3; i++)
        {
            try { await pipeline.ExecuteAsync(_ => throw new Exception("failure")); }
            catch { }
        }

        // Act & Assert - circuit ต้องเปิดแล้ว
        await Assert.ThrowsAsync<BrokenCircuitException>(() =>
            pipeline.ExecuteAsync(_ => ValueTask.CompletedTask).AsTask());
    }

    [Fact]
    public async Task Timeout_ShouldThrowWhenExceeded()
    {
        // Arrange - ใช้ FakeTimeProvider
        var fakeTime = new FakeTimeProvider();
        
        var pipeline = new ResiliencePipelineBuilder()
            .AddTimeout(new TimeoutStrategyOptions
            {
                Timeout = TimeSpan.FromSeconds(5),
                TimeProvider = fakeTime
            })
            .Build();

        // Act & Assert
        var task = pipeline.ExecuteAsync(async ct =>
        {
            await Task.Delay(Timeout.Infinite, ct); // รอไม่สิ้นสุด
        }).AsTask();

        // เดิน time ไปข้างหน้า 6 วินาที
        fakeTime.Advance(TimeSpan.FromSeconds(6));

        await Assert.ThrowsAsync<TimeoutRejectedException>(() => task);
    }
}
```

### Integration Testing

```csharp
public class WeatherServiceIntegrationTests
{
    [Fact]
    public async Task GetWeather_ShouldFallbackWhenServiceDown()
    {
        // Arrange
        var mockHandler = new MockHttpMessageHandler();
        mockHandler.When("/api/weather*")
            .Respond(System.Net.HttpStatusCode.ServiceUnavailable);

        var httpClient = new HttpClient(mockHandler);
        var service = new WeatherService(httpClient);

        // Act & Assert - ควรได้ fallback response ไม่ใช่ exception
        var result = await service.GetWeatherAsync("Bangkok");
        Assert.NotNull(result);
    }
}
```

### Verify Pipeline Descriptors

```csharp
[Fact]
public void Pipeline_ShouldHaveCorrectStrategies()
{
    // Arrange
    var pipeline = new ResiliencePipelineBuilder()
        .AddRetry(new RetryStrategyOptions { MaxRetryAttempts = 3 })
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            MinimumThroughput = 5,
            SamplingDuration = TimeSpan.FromSeconds(10),
            BreakDuration = TimeSpan.FromSeconds(30)
        })
        .AddTimeout(TimeSpan.FromSeconds(10))
        .Build();

    // Act - ใช้ GetPipelineDescriptor ใน Polly.Testing
    var descriptor = pipeline.GetPipelineDescriptor();

    // Assert
    Assert.Equal(3, descriptor.Strategies.Count);
    Assert.Contains(descriptor.Strategies, s => s.StrategyType == typeof(RetryResilienceStrategy));
    Assert.Contains(descriptor.Strategies, s => s.StrategyType == typeof(CircuitBreakerResilienceStrategy));
    Assert.Contains(descriptor.Strategies, s => s.StrategyType == typeof(TimeoutResilienceStrategy));
}
```

---

## สรุปภาพรวม Resilience Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│                    Resilience Pipeline                          │
│                                                                 │
│  Request  →  [Fallback] → [Timeout] → [Retry] → [CB] → [Work]  │
│  Response ←  [Fallback] ← [Timeout] ← [Retry] ← [CB] ← [Work]  │
│                                                                 │
│  Fallback: ตอบ default/cache เมื่อทั้ง pipeline ล้มเหลว          │
│  Timeout:  จำกัดเวลารวมของทั้ง pipeline                          │
│  Retry:    ลองใหม่พร้อม exponential backoff + jitter             │
│  CB:       ตัดการเชื่อมต่อเมื่อ failure rate สูง                │
│  Work:     การทำงานจริง                                          │
└─────────────────────────────────────────────────────────────────┘
```

### Quick Reference

| Pattern | ใช้เมื่อ | Polly API |
|---------|---------|----------|
| Retry | Transient failures | `AddRetry()` |
| Circuit Breaker | Service กำลังมีปัญหา | `AddCircuitBreaker()` |
| Timeout | ป้องกันรอนาน | `AddTimeout()` |
| Fallback | ต้องการ graceful degradation | `AddFallback()` |
| Hedging | ต้องการ low latency | `AddHedging()` |
| Rate Limiting | ป้องกัน overload | `AddRateLimiter()` |
| Bulkhead | Isolate resource pools | `AddConcurrencyLimiter()` |

---

**ก่อนหน้า → [Part 71: Observability](part71-observability.md)**
**ต่อไป → [Part 73: Advanced Testing](part73-advanced-testing.md)**
