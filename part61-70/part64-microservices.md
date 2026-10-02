# Part 64: Microservices Architecture (Steps 631-640)

> **หมายเหตุ**: บทนี้ครอบคลุมสถาปัตยกรรม Microservices ตั้งแต่ภาพรวมจนถึงการสร้าง Demo จริง  
> ใช้ .NET 8, YARP, MassTransit, RabbitMQ และ Polly

---

## Step 631: Microservices Overview

### ทำความเข้าใจ Monolith vs Microservices

ก่อนจะเลือกสถาปัตยกรรม ต้องเข้าใจ trade-off ของแต่ละแบบก่อน

#### Monolith (แอปพลิเคชันแบบเดิม)

Monolith คือแอปพลิเคชันที่ทุก component อยู่ใน codebase เดียว, deploy พร้อมกันทั้งหมด

```
┌─────────────────────────────────────┐
│            Monolith App             │
│                                     │
│  ┌──────────┐  ┌──────────────────┐ │
│  │  Orders  │  │    Inventory     │ │
│  │ Module   │  │     Module       │ │
│  └──────────┘  └──────────────────┘ │
│  ┌──────────┐  ┌──────────────────┐ │
│  │ Payment  │  │   Notification   │ │
│  │ Module   │  │     Module       │ │
│  └──────────┘  └──────────────────┘ │
│                                     │
│         ┌───────────┐               │
│         │ Single DB │               │
│         └───────────┘               │
└─────────────────────────────────────┘
```

**ข้อดี Monolith:**
- ง่ายต่อการพัฒนาในช่วงแรก
- Debug และ Test ง่ายกว่า
- ไม่มี network latency ระหว่าง module
- Transaction ง่าย (single database)
- Deploy ง่าย (ไฟล์เดียว)
- ไม่ต้องการ infrastructure ซับซ้อน

**ข้อเสีย Monolith:**
- Scale ทั้งแอปพลิเคชัน ไม่สามารถ scale แค่บางส่วน
- Team หลายทีมทำงานใน codebase เดียว → conflict บ่อย
- Technology stack ต้องเหมือนกันทั้งหมด
- Deploy ทั้งแอปพลิเคชันทุกครั้ง แม้แค่แก้ไขเล็กน้อย
- Fault isolation ต่ำ (bug ส่วนหนึ่งทำให้ทั้งระบบล้มได้)

#### Microservices (บริการขนาดเล็กหลายตัว)

Microservices แบ่งแอปพลิเคชันออกเป็น services เล็กๆ ที่ deploy และ scale แยกกัน

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Order     │    │  Inventory  │    │   Payment   │
│   Service   │    │   Service   │    │   Service   │
│             │    │             │    │             │
│  ┌───────┐  │    │  ┌───────┐  │    │  ┌───────┐  │
│  │  DB   │  │    │  │  DB   │  │    │  │  DB   │  │
│  └───────┘  │    │  └───────┘  │    │  └───────┘  │
└─────────────┘    └─────────────┘    └─────────────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                  ┌───────────────┐
                  │  Message Bus  │
                  │  (RabbitMQ)   │
                  └───────────────┘
```

**ข้อดี Microservices:**
- Scale แต่ละ service แยกกันได้ตามความต้องการ
- Team สามารถทำงาน independent ได้
- Technology heterogeneity (แต่ละ service ใช้ tech ต่างกันได้)
- Fault isolation ดีกว่า
- Deploy แยกกันได้

**ข้อเสีย Microservices:**
- ซับซ้อนกว่ามาก (network, distributed systems)
- Distributed transactions ยาก
- Testing ยากกว่า
- ต้องการ infrastructure มากกว่า (Kubernetes, Service Mesh)
- Debugging ข้าม service ยาก
- Latency เพิ่มขึ้นจาก network calls

### เมื่อไหร่ควรใช้ Microservices (ไม่ใช่เสมอไป!)

> **คำเตือน**: Microservices ไม่ใช่ silver bullet และไม่เหมาะกับทุกสถานการณ์

**ควรใช้ Microservices เมื่อ:**
- มี team หลายทีมทำงานใน domain ต่างกัน (Conway's Law)
- แต่ละส่วนมี scaling requirement ต่างกันมาก
- ต้องการ deploy แต่ละ component แยกกัน
- แอปพลิเคชันมีขนาดใหญ่และ domain ชัดเจน
- มี operational maturity (DevOps, monitoring, tracing)

**ไม่ควรใช้ Microservices เมื่อ:**
- แอปพลิเคชันขนาดเล็กหรือ team เล็ก
- เพิ่งเริ่มต้นโปรเจค (ยังไม่รู้ domain boundary)
- ไม่มี infrastructure team
- Business domain ยังไม่ชัดเจน

**แนวทาง: Start with a Modular Monolith → Extract to Microservices เมื่อจำเป็น**

### Bounded Contexts (การกำหนดขอบเขต Service)

Bounded Context มาจาก Domain-Driven Design (DDD) คือการกำหนดขอบเขตที่ model/term มีความหมายเฉพาะ

```csharp
// ใน Order Context
public class Order
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public List<OrderLine> Lines { get; set; }
    public OrderStatus Status { get; set; }
    public decimal TotalAmount { get; set; }
}

// ใน Inventory Context
// "Product" มีความหมายต่างกันใน context นี้
public class Product
{
    public Guid Id { get; set; }
    public string SKU { get; set; }
    public int QuantityInStock { get; set; }
    public int ReservedQuantity { get; set; }
    public WarehouseLocation Location { get; set; }
}

// ใน Payment Context
// "Order" มีความหมายต่างกัน - แค่ reference สำหรับการชำระเงิน
public class PaymentRequest
{
    public Guid OrderId { get; set; }  // แค่ reference
    public decimal Amount { get; set; }
    public PaymentMethod Method { get; set; }
}
```

### Common Microservices Patterns

#### 1. API Gateway Pattern

```
Client → API Gateway → Service A
                     → Service B
                     → Service C
```

API Gateway ทำหน้าที่เป็น single entry point สำหรับ client:
- Routing requests ไปยัง service ที่เหมาะสม
- Authentication/Authorization
- Rate limiting
- Load balancing
- SSL termination
- Request/Response transformation

#### 2. Service Discovery Pattern

Services ค้นหากันเองแบบ dynamic แทนที่จะ hardcode URL

```
Service A → Service Registry → ได้รับ URL ของ Service B
         ← ─────────────────
Service A → Service B (ใช้ URL ที่ได้มา)
```

#### 3. Circuit Breaker Pattern

ป้องกันการเรียก service ที่ล้มเหลวซ้ำๆ

```
States: Closed → Open → Half-Open → Closed
        (ปกติ)  (ปิด)   (ทดสอบ)    (กลับปกติ)
```

#### 4. Saga Pattern

จัดการ distributed transactions ผ่าน sequence ของ local transactions

```
Order Service         Payment Service      Inventory Service
    │                      │                     │
    │──Create Order─────→  │                     │
    │                      │──Charge Payment───→ │
    │                      │                     │──Reserve Stock→
    │                      │                     │
    │ (ถ้า Reserve ล้มเหลว) │                     │
    │                      │←──Refund Payment──  │
    │←─Cancel Order──       │                     │
```

---

## Step 632: API Gateway with YARP

### YARP คืออะไร?

YARP (Yet Another Reverse Proxy) คือ library สำหรับสร้าง reverse proxy ด้วย .NET  
พัฒนาโดย Microsoft และเป็น open source

**ทำไมต้องใช้ YARP แทน Nginx/Traefik?**
- เขียนด้วย .NET → integrate กับ ecosystem ได้ดี
- Customize ได้ง่ายด้วย middleware
- มี features ครบ: load balancing, health checks, rate limiting
- Configuration แบบ code หรือ JSON

### การติดตั้ง YARP

```bash
dotnet new webapi -n ApiGateway
cd ApiGateway
dotnet add package Yarp.ReverseProxy
```

### โครงสร้างโปรเจค API Gateway

```
ApiGateway/
├── Program.cs
├── appsettings.json
├── Transforms/
│   └── RequestTransforms.cs
└── Middleware/
    └── AuthenticationMiddleware.cs
```

### Configuration แบบ JSON

```json
// appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "order-route": {
        "ClusterId": "order-cluster",
        "Match": {
          "Path": "/api/orders/{**catch-all}"
        },
        "Transforms": [
          {
            "PathPattern": "/api/orders/{**catch-all}"
          }
        ],
        "Metadata": {
          "RequireAuth": "true"
        }
      },
      "inventory-route": {
        "ClusterId": "inventory-cluster",
        "Match": {
          "Path": "/api/inventory/{**catch-all}"
        },
        "Transforms": [
          {
            "PathPattern": "/api/inventory/{**catch-all}"
          }
        ]
      },
      "public-route": {
        "ClusterId": "order-cluster",
        "Match": {
          "Path": "/api/public/{**catch-all}"
        }
      }
    },
    "Clusters": {
      "order-cluster": {
        "LoadBalancingPolicy": "RoundRobin",
        "HealthCheck": {
          "Active": {
            "Enabled": true,
            "Interval": "00:00:10",
            "Timeout": "00:00:05",
            "Policy": "ConsecutiveFailures",
            "Path": "/health"
          }
        },
        "Destinations": {
          "order-service-1": {
            "Address": "http://order-service:5001"
          },
          "order-service-2": {
            "Address": "http://order-service-2:5001"
          }
        }
      },
      "inventory-cluster": {
        "LoadBalancingPolicy": "LeastRequests",
        "Destinations": {
          "inventory-service-1": {
            "Address": "http://inventory-service:5002"
          }
        }
      }
    }
  }
}
```

### Configuration แบบ Code (Dynamic)

```csharp
// Program.cs
using Yarp.ReverseProxy.Configuration;

var builder = WebApplication.CreateBuilder(args);

// วิธีที่ 1: โหลดจาก appsettings.json
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

// วิธีที่ 2: สร้าง config ด้วย code
// builder.Services.AddReverseProxy()
//     .LoadFromMemory(GetRoutes(), GetClusters());

// เพิ่ม Authentication
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = "https://identity-server";
        options.Audience = "api";
    });

builder.Services.AddAuthorization();

// Rate Limiting
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", limiterOptions =>
    {
        limiterOptions.PermitLimit = 100;
        limiterOptions.Window = TimeSpan.FromMinutes(1);
        limiterOptions.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        limiterOptions.QueueLimit = 10;
    });
    
    options.AddSlidingWindowLimiter("sliding", limiterOptions =>
    {
        limiterOptions.PermitLimit = 100;
        limiterOptions.Window = TimeSpan.FromMinutes(1);
        limiterOptions.SegmentsPerWindow = 6;
    });
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();

// Custom middleware สำหรับ gateway
app.Use(async (context, next) =>
{
    // เพิ่ม correlation ID
    if (!context.Request.Headers.ContainsKey("X-Correlation-Id"))
    {
        context.Request.Headers["X-Correlation-Id"] = Guid.NewGuid().ToString();
    }
    
    await next();
});

app.MapReverseProxy(proxyPipeline =>
{
    // Custom pipeline สำหรับ proxy
    proxyPipeline.Use(async (context, next) =>
    {
        // ตรวจสอบ authentication สำหรับ routes ที่ต้องการ
        var endpoint = context.GetEndpoint();
        var routeConfig = context.GetRouteModel();
        
        if (routeConfig?.Config?.Metadata?.TryGetValue("RequireAuth", out var requireAuth) == true
            && requireAuth == "true")
        {
            if (!context.User.Identity?.IsAuthenticated == true)
            {
                context.Response.StatusCode = 401;
                return;
            }
        }
        
        await next();
    });
    
    proxyPipeline.UsePassiveHealthChecks();
});

app.Run();
```

### Load Balancing Policies

```csharp
// สร้าง Custom Load Balancing Policy
using Yarp.ReverseProxy.LoadBalancing;

public class PriorityLoadBalancingPolicy : ILoadBalancingPolicy
{
    public string Name => "Priority";
    
    public DestinationState? PickDestination(
        HttpContext context, 
        ClusterState cluster, 
        IReadOnlyList<DestinationState> availableDestinations)
    {
        if (availableDestinations.Count == 0)
            return null;
            
        // เลือก destination ที่มี priority สูงสุดก่อน
        return availableDestinations
            .OrderByDescending(d => 
                d.Model.Config.Metadata?.TryGetValue("Priority", out var p) == true 
                    ? int.Parse(p) : 0)
            .First();
    }
}

// ลงทะเบียน policy
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddLoadBalancingPolicy<PriorityLoadBalancingPolicy>("Priority");
```

### Request/Response Transforms

```csharp
// Transforms/RequestTransforms.cs
using Yarp.ReverseProxy.Transforms;
using Yarp.ReverseProxy.Transforms.Builder;

public class AddServiceVersionTransform : RequestTransform
{
    private readonly string _serviceVersion;
    
    public AddServiceVersionTransform(string serviceVersion)
    {
        _serviceVersion = serviceVersion;
    }
    
    public override ValueTask ApplyAsync(RequestTransformContext context)
    {
        context.ProxyRequest.Headers.Add("X-Service-Version", _serviceVersion);
        return ValueTask.CompletedTask;
    }
}

public class ServiceVersionTransformProvider : ITransformProvider
{
    public void ValidateRoute(TransformRouteValidationContext context) { }
    
    public void ValidateCluster(TransformClusterValidationContext context) { }
    
    public void Apply(TransformBuilderContext context)
    {
        // เพิ่ม transform สำหรับทุก route
        context.AddRequestTransform(async transformContext =>
        {
            // ลบ sensitive headers
            transformContext.ProxyRequest.Headers.Remove("X-Internal-Secret");
            
            // เพิ่ม headers ที่ต้องการ
            transformContext.ProxyRequest.Headers.TryAddWithoutValidation(
                "X-Gateway-Version", "1.0.0");
                
            await ValueTask.CompletedTask;
        });
    }
}

// ลงทะเบียน
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddTransforms<ServiceVersionTransformProvider>();
```

---

## Step 633: Service-to-Service Communication

### HttpClientFactory และ Typed Clients

HttpClientFactory จัดการ lifecycle ของ HttpClient อย่างถูกต้อง ป้องกัน socket exhaustion

```csharp
// Clients/IInventoryClient.cs
public interface IInventoryClient
{
    Task<InventoryItem?> GetItemAsync(Guid productId, CancellationToken ct = default);
    Task<bool> ReserveStockAsync(Guid productId, int quantity, CancellationToken ct = default);
    Task<bool> ReleaseStockAsync(Guid productId, int quantity, CancellationToken ct = default);
}

// Clients/InventoryClient.cs
public class InventoryClient : IInventoryClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<InventoryClient> _logger;
    
    public InventoryClient(HttpClient httpClient, ILogger<InventoryClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }
    
    public async Task<InventoryItem?> GetItemAsync(Guid productId, CancellationToken ct = default)
    {
        try
        {
            var response = await _httpClient.GetAsync($"/api/inventory/{productId}", ct);
            
            if (response.StatusCode == HttpStatusCode.NotFound)
                return null;
                
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<InventoryItem>(cancellationToken: ct);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Failed to get inventory item {ProductId}", productId);
            throw;
        }
    }
    
    public async Task<bool> ReserveStockAsync(Guid productId, int quantity, CancellationToken ct = default)
    {
        var request = new ReserveStockRequest(productId, quantity);
        var response = await _httpClient.PostAsJsonAsync("/api/inventory/reserve", request, ct);
        return response.IsSuccessStatusCode;
    }
    
    public async Task<bool> ReleaseStockAsync(Guid productId, int quantity, CancellationToken ct = default)
    {
        var request = new ReleaseStockRequest(productId, quantity);
        var response = await _httpClient.PostAsJsonAsync("/api/inventory/release", request, ct);
        return response.IsSuccessStatusCode;
    }
}

// Program.cs - ลงทะเบียน Typed Client
builder.Services.AddHttpClient<IInventoryClient, InventoryClient>(client =>
{
    client.BaseAddress = new Uri("http://inventory-service:5002");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
    client.Timeout = TimeSpan.FromSeconds(30);
});
```

### Polly สำหรับ Resilience

Polly เป็น library สำหรับ resilience patterns: retry, circuit breaker, timeout, fallback

```bash
dotnet add package Microsoft.Extensions.Http.Polly
dotnet add package Polly
dotnet add package Polly.Extensions.Http
```

```csharp
// ResiliencePolicies/HttpPolicies.cs
using Polly;
using Polly.Extensions.Http;

public static class HttpPolicies
{
    // Retry Policy: ลอง 3 ครั้ง, delay แบบ exponential
    public static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
    {
        return HttpPolicyExtensions
            .HandleTransientHttpError()  // 5xx, network errors
            .OrResult(msg => msg.StatusCode == HttpStatusCode.TooManyRequests)
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: retryAttempt => 
                    TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),  // 2, 4, 8 วินาที
                onRetry: (outcome, timespan, retryCount, context) =>
                {
                    Console.WriteLine($"Retry {retryCount} after {timespan}s. " +
                        $"Reason: {outcome.Exception?.Message ?? outcome.Result.StatusCode.ToString()}");
                });
    }
    
    // Circuit Breaker: ปิดวงจรเมื่อล้มเหลว 5 ครั้งใน 30 วินาที
    public static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
    {
        return HttpPolicyExtensions
            .HandleTransientHttpError()
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (exception, duration) =>
                {
                    Console.WriteLine($"Circuit opened for {duration}s. " +
                        $"Reason: {exception.Exception?.Message}");
                },
                onReset: () => Console.WriteLine("Circuit closed - service recovered"),
                onHalfOpen: () => Console.WriteLine("Circuit half-open - testing service")
            );
    }
    
    // Timeout Policy
    public static IAsyncPolicy<HttpResponseMessage> GetTimeoutPolicy()
    {
        return Policy.TimeoutAsync<HttpResponseMessage>(
            seconds: 10,
            timeoutStrategy: TimeoutStrategy.Optimistic);
    }
    
    // รวม Policies (Wrap)
    public static IAsyncPolicy<HttpResponseMessage> GetCombinedPolicy()
    {
        return Policy.WrapAsync(
            GetRetryPolicy(),
            GetCircuitBreakerPolicy(),
            GetTimeoutPolicy()
        );
    }
}

// Program.cs
builder.Services
    .AddHttpClient<IInventoryClient, InventoryClient>(client =>
    {
        client.BaseAddress = new Uri("http://inventory-service:5002");
    })
    .AddPolicyHandler(HttpPolicies.GetRetryPolicy())
    .AddPolicyHandler(HttpPolicies.GetCircuitBreakerPolicy())
    .AddPolicyHandler(HttpPolicies.GetTimeoutPolicy());
```

### Polly v8 (Modern API)

```csharp
// ใช้ Polly v8 และ Microsoft.Extensions.Http.Resilience
dotnet add package Microsoft.Extensions.Http.Resilience

// Program.cs
builder.Services
    .AddHttpClient<IInventoryClient, InventoryClient>(client =>
    {
        client.BaseAddress = new Uri("http://inventory-service:5002");
    })
    .AddStandardResilienceHandler(options =>
    {
        // Retry
        options.Retry = new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            Delay = TimeSpan.FromSeconds(1)
        };
        
        // Circuit Breaker
        options.CircuitBreaker = new HttpCircuitBreakerStrategyOptions
        {
            SamplingDuration = TimeSpan.FromSeconds(30),
            FailureRatio = 0.5,
            MinimumThroughput = 5,
            BreakDuration = TimeSpan.FromSeconds(30)
        };
        
        // Timeout
        options.TotalRequestTimeout = new HttpTimeoutStrategyOptions
        {
            Timeout = TimeSpan.FromSeconds(30)
        };
    });
```

### gRPC สำหรับ Internal Services

gRPC เหมาะสำหรับ high-performance internal communication ระหว่าง services

```bash
dotnet add package Grpc.AspNetCore
dotnet add package Google.Protobuf
dotnet add package Grpc.Tools
```

```protobuf
// Protos/inventory.proto
syntax = "proto3";

option csharp_namespace = "InventoryService.Protos";

package inventory;

service InventoryService {
  rpc GetItem (GetItemRequest) returns (GetItemResponse);
  rpc ReserveStock (ReserveStockRequest) returns (ReserveStockResponse);
  rpc ReleaseStock (ReleaseStockRequest) returns (ReleaseStockResponse);
  rpc StreamInventoryUpdates (StreamRequest) returns (stream InventoryUpdate);
}

message GetItemRequest {
  string product_id = 1;
}

message GetItemResponse {
  string product_id = 1;
  string name = 2;
  int32 quantity_available = 3;
  bool found = 4;
}

message ReserveStockRequest {
  string product_id = 1;
  int32 quantity = 2;
  string order_id = 3;
}

message ReserveStockResponse {
  bool success = 1;
  string message = 2;
}

message ReleaseStockRequest {
  string product_id = 1;
  int32 quantity = 2;
}

message ReleaseStockResponse {
  bool success = 1;
}

message StreamRequest {}

message InventoryUpdate {
  string product_id = 1;
  int32 quantity = 2;
  string event_type = 3;
}
```

```csharp
// Services/InventoryGrpcService.cs (Server Side)
using Grpc.Core;
using InventoryService.Protos;

public class InventoryGrpcService : InventoryService.InventoryServiceBase
{
    private readonly IInventoryRepository _repository;
    private readonly ILogger<InventoryGrpcService> _logger;
    
    public InventoryGrpcService(
        IInventoryRepository repository,
        ILogger<InventoryGrpcService> logger)
    {
        _repository = repository;
        _logger = logger;
    }
    
    public override async Task<GetItemResponse> GetItem(
        GetItemRequest request, 
        ServerCallContext context)
    {
        var item = await _repository.GetByProductIdAsync(Guid.Parse(request.ProductId));
        
        if (item == null)
        {
            return new GetItemResponse { Found = false };
        }
        
        return new GetItemResponse
        {
            ProductId = item.ProductId.ToString(),
            Name = item.Name,
            QuantityAvailable = item.QuantityAvailable,
            Found = true
        };
    }
    
    public override async Task<ReserveStockResponse> ReserveStock(
        ReserveStockRequest request, 
        ServerCallContext context)
    {
        try
        {
            var success = await _repository.ReserveAsync(
                Guid.Parse(request.ProductId), 
                request.Quantity, 
                Guid.Parse(request.OrderId));
                
            return new ReserveStockResponse 
            { 
                Success = success,
                Message = success ? "Reserved successfully" : "Insufficient stock"
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to reserve stock");
            throw new RpcException(new Status(StatusCode.Internal, ex.Message));
        }
    }
    
    // Streaming RPC
    public override async Task StreamInventoryUpdates(
        StreamRequest request,
        IServerStreamWriter<InventoryUpdate> responseStream,
        ServerCallContext context)
    {
        // Stream updates จนกว่า client จะยกเลิก
        while (!context.CancellationToken.IsCancellationRequested)
        {
            var updates = await _repository.GetRecentUpdatesAsync();
            
            foreach (var update in updates)
            {
                await responseStream.WriteAsync(new InventoryUpdate
                {
                    ProductId = update.ProductId.ToString(),
                    Quantity = update.Quantity,
                    EventType = update.EventType
                });
            }
            
            await Task.Delay(5000, context.CancellationToken);
        }
    }
}

// Program.cs (Server)
builder.Services.AddGrpc(options =>
{
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
    options.MaxReceiveMessageSize = 2 * 1024 * 1024; // 2MB
    options.MaxSendMessageSize = 5 * 1024 * 1024;    // 5MB
});

app.MapGrpcService<InventoryGrpcService>();
```

```csharp
// Clients/InventoryGrpcClient.cs (Client Side)
using Grpc.Net.Client;
using InventoryService.Protos;

public class InventoryGrpcClient : IInventoryClient
{
    private readonly InventoryService.InventoryServiceClient _client;
    
    public InventoryGrpcClient(InventoryService.InventoryServiceClient client)
    {
        _client = client;
    }
    
    public async Task<InventoryItem?> GetItemAsync(Guid productId, CancellationToken ct = default)
    {
        var response = await _client.GetItemAsync(
            new GetItemRequest { ProductId = productId.ToString() },
            cancellationToken: ct);
            
        if (!response.Found)
            return null;
            
        return new InventoryItem
        {
            ProductId = Guid.Parse(response.ProductId),
            Name = response.Name,
            QuantityAvailable = response.QuantityAvailable
        };
    }
}

// Program.cs (Client)
builder.Services.AddGrpcClient<InventoryService.InventoryServiceClient>(options =>
{
    options.Address = new Uri("https://inventory-service:5002");
})
.ConfigureChannel(channelOptions =>
{
    channelOptions.HttpHandler = new SocketsHttpHandler
    {
        KeepAlivePingDelay = TimeSpan.FromSeconds(60),
        KeepAlivePingTimeout = TimeSpan.FromSeconds(30),
        EnableMultipleHttp2Connections = true
    };
});
```

### Health Checks

```csharp
// Health Checks สำหรับ Service Discovery
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddRabbitMQ(
        rabbitConnectionString: "amqp://guest:guest@rabbitmq:5672",
        name: "rabbitmq",
        failureStatus: HealthStatus.Degraded)
    .AddNpgSql(
        connectionString: builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "database",
        failureStatus: HealthStatus.Unhealthy)
    .AddCheck<ExternalServiceHealthCheck>("external-service");

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // เช็คแค่ว่า process ยังทำงานอยู่
});

// Custom Health Check
public class ExternalServiceHealthCheck : IHealthCheck
{
    private readonly IHttpClientFactory _clientFactory;
    
    public ExternalServiceHealthCheck(IHttpClientFactory clientFactory)
    {
        _clientFactory = clientFactory;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, 
        CancellationToken cancellationToken = default)
    {
        try
        {
            var client = _clientFactory.CreateClient("payment-service");
            var response = await client.GetAsync("/health", cancellationToken);
            
            return response.IsSuccessStatusCode
                ? HealthCheckResult.Healthy("Payment service is healthy")
                : HealthCheckResult.Unhealthy($"Payment service returned {response.StatusCode}");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(ex.Message);
        }
    }
}
```

---

## Step 634: Message Bus (RabbitMQ / MassTransit)

### ทำความเข้าใจ Message-Driven Architecture

แทนที่จะให้ services เรียกกันตรงๆ (synchronous), ใช้ message bus เป็นตัวกลาง (asynchronous)

```
OrderService → [Message Bus] → InventoryService
                             → NotificationService
                             → AnalyticsService
```

**ข้อดีของ Async Messaging:**
- Decoupling: services ไม่ต้องรู้จักกัน
- Resilience: ถ้า consumer ล้ม messages ยังอยู่ใน queue
- Scalability: เพิ่ม consumer ได้ตามต้องการ
- Temporal decoupling: producer และ consumer ไม่ต้อง online พร้อมกัน

### MassTransit Setup

MassTransit เป็น abstraction layer เหนือ message brokers (RabbitMQ, Azure Service Bus, ฯลฯ)

```bash
dotnet add package MassTransit
dotnet add package MassTransit.RabbitMQ
dotnet add package MassTransit.EntityFrameworkCore
```

### docker-compose.yml สำหรับ RabbitMQ

```yaml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"    # AMQP port
      - "15672:15672"  # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 30s
      timeout: 30s
      retries: 3

volumes:
  rabbitmq_data:
```

### Messages / Events / Commands

ใน MassTransit มีความต่างระหว่าง:
- **Event**: สิ่งที่เกิดขึ้นแล้ว (past tense) - `OrderCreated`, `PaymentProcessed`
- **Command**: สั่งให้ทำ (imperative) - `CreateOrder`, `ProcessPayment`
- **Request/Response**: ถาม-ตอบแบบ synchronous ผ่าน async

```csharp
// Contracts/Events/OrderCreatedEvent.cs
// วางไว้ใน shared contracts project
namespace Contracts.Events;

public record OrderCreatedEvent
{
    public Guid OrderId { get; init; }
    public Guid CustomerId { get; init; }
    public List<OrderLine> Lines { get; init; } = new();
    public decimal TotalAmount { get; init; }
    public DateTime CreatedAt { get; init; }
}

public record OrderLine
{
    public Guid ProductId { get; init; }
    public string ProductName { get; init; } = string.Empty;
    public int Quantity { get; init; }
    public decimal UnitPrice { get; init; }
}

// Contracts/Events/StockReservedEvent.cs
public record StockReservedEvent
{
    public Guid OrderId { get; init; }
    public Guid ProductId { get; init; }
    public int ReservedQuantity { get; init; }
    public DateTime ReservedAt { get; init; }
}

// Contracts/Commands/ReserveStockCommand.cs
public record ReserveStockCommand
{
    public Guid OrderId { get; init; }
    public Guid ProductId { get; init; }
    public int Quantity { get; init; }
}
```

### Publishing Messages: Publish vs Send

```csharp
// Publish: ส่งไปให้ subscriber ทุกคน (fan-out)
// ใช้สำหรับ events ที่หลาย services อาจสนใจ
public class OrderService
{
    private readonly IBus _bus;
    private readonly IOrderRepository _repository;
    
    public OrderService(IBus bus, IOrderRepository repository)
    {
        _bus = bus;
        _repository = repository;
    }
    
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = request.CustomerId,
            Lines = request.Lines.Select(l => new OrderLine
            {
                ProductId = l.ProductId,
                Quantity = l.Quantity
            }).ToList(),
            Status = OrderStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };
        
        await _repository.AddAsync(order);
        
        // Publish event หลังจาก save สำเร็จ
        await _bus.Publish(new OrderCreatedEvent
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            Lines = order.Lines.Select(l => new Contracts.Events.OrderLine
            {
                ProductId = l.ProductId,
                Quantity = l.Quantity
            }).ToList(),
            TotalAmount = order.TotalAmount,
            CreatedAt = order.CreatedAt
        });
        
        return order;
    }
}

// Send: ส่งไปยัง endpoint ที่ระบุ (point-to-point)
// ใช้สำหรับ commands ที่มี handler เดียว
public class OrderController : ControllerBase
{
    private readonly ISendEndpointProvider _sendEndpointProvider;
    
    public OrderController(ISendEndpointProvider sendEndpointProvider)
    {
        _sendEndpointProvider = sendEndpointProvider;
    }
    
    [HttpPost("cancel")]
    public async Task<IActionResult> Cancel(Guid orderId)
    {
        var endpoint = await _sendEndpointProvider.GetSendEndpoint(
            new Uri("queue:cancel-order"));
            
        await endpoint.Send(new CancelOrderCommand { OrderId = orderId });
        
        return Accepted();
    }
}
```

### Consumer: IConsumer<T>

```csharp
// Consumers/OrderCreatedConsumer.cs (ใน InventoryService)
using MassTransit;

public class OrderCreatedConsumer : IConsumer<OrderCreatedEvent>
{
    private readonly IInventoryRepository _repository;
    private readonly ILogger<OrderCreatedConsumer> _logger;
    private readonly IBus _bus;
    
    public OrderCreatedConsumer(
        IInventoryRepository repository,
        ILogger<OrderCreatedConsumer> logger,
        IBus bus)
    {
        _repository = repository;
        _logger = logger;
        _bus = bus;
    }
    
    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        var order = context.Message;
        _logger.LogInformation("Processing OrderCreated event for Order {OrderId}", order.OrderId);
        
        try
        {
            foreach (var line in order.Lines)
            {
                var reserved = await _repository.ReserveStockAsync(
                    line.ProductId, 
                    line.Quantity,
                    order.OrderId);
                    
                if (!reserved)
                {
                    _logger.LogWarning(
                        "Failed to reserve stock for Product {ProductId}, Quantity {Quantity}",
                        line.ProductId, line.Quantity);
                        
                    // Publish failure event
                    await _bus.Publish(new StockReservationFailedEvent
                    {
                        OrderId = order.OrderId,
                        ProductId = line.ProductId,
                        RequestedQuantity = line.Quantity
                    });
                    return;
                }
            }
            
            // Publish success event
            await _bus.Publish(new StockReservedEvent
            {
                OrderId = order.OrderId,
                ReservedAt = DateTime.UtcNow
            });
            
            _logger.LogInformation("Stock reserved successfully for Order {OrderId}", order.OrderId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing OrderCreated for Order {OrderId}", order.OrderId);
            throw; // MassTransit จะ retry ให้อัตโนมัติ
        }
    }
}

// Consumer Definition (ควบคุม behavior)
public class OrderCreatedConsumerDefinition : ConsumerDefinition<OrderCreatedConsumer>
{
    public OrderCreatedConsumerDefinition()
    {
        // กำหนด concurrency
        ConcurrentMessageLimit = 10;
    }
    
    protected override void ConfigureConsumer(
        IReceiveEndpointConfigurator endpointConfigurator,
        IConsumerConfigurator<OrderCreatedConsumer> consumerConfigurator)
    {
        // Retry policy
        endpointConfigurator.UseMessageRetry(retry =>
        {
            retry.Incremental(3, TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(5));
        });
        
        // Circuit breaker
        endpointConfigurator.UseCircuitBreaker(cb =>
        {
            cb.TrackingPeriod = TimeSpan.FromMinutes(1);
            cb.TripThreshold = 15;
            cb.ActiveThreshold = 10;
            cb.ResetInterval = TimeSpan.FromMinutes(5);
        });
    }
}
```

### MassTransit Configuration

```csharp
// Program.cs (InventoryService)
builder.Services.AddMassTransit(x =>
{
    // ลงทะเบียน consumers
    x.AddConsumer<OrderCreatedConsumer, OrderCreatedConsumerDefinition>();
    x.AddConsumer<StockReservationFailedConsumer>();
    
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
        
        // กำหนด exchange/queue
        cfg.ReceiveEndpoint("inventory-order-created", e =>
        {
            e.ConfigureConsumer<OrderCreatedConsumer>(context);
            
            // Dead letter queue
            e.BindDeadLetterQueue("inventory-order-created-dlq");
            
            // Prefetch count
            e.PrefetchCount = 16;
        });
        
        // สร้าง topology อัตโนมัติ
        cfg.ConfigureEndpoints(context);
    });
});
```

### Saga: OrderSaga (State Machine)

Saga จัดการ long-running process ที่ span หลาย services

```csharp
// Sagas/OrderSagaState.cs
public class OrderSagaState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = string.Empty;
    public Guid CustomerId { get; set; }
    public decimal TotalAmount { get; set; }
    public bool StockReserved { get; set; }
    public bool PaymentProcessed { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
}

// Sagas/OrderStateMachine.cs
public class OrderStateMachine : MassTransitStateMachine<OrderSagaState>
{
    // States
    public State Pending { get; private set; } = default!;
    public State ReservingStock { get; private set; } = default!;
    public State ProcessingPayment { get; private set; } = default!;
    public State Completed { get; private set; } = default!;
    public State Cancelled { get; private set; } = default!;
    
    // Events
    public Event<OrderCreatedEvent> OrderCreated { get; private set; } = default!;
    public Event<StockReservedEvent> StockReserved { get; private set; } = default!;
    public Event<StockReservationFailedEvent> StockReservationFailed { get; private set; } = default!;
    public Event<PaymentProcessedEvent> PaymentProcessed { get; private set; } = default!;
    public Event<PaymentFailedEvent> PaymentFailed { get; private set; } = default!;
    
    public OrderStateMachine()
    {
        // กำหนดว่าต้องใช้ property ไหนในการ correlate
        InstanceState(x => x.CurrentState);
        
        Event(() => OrderCreated, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => StockReserved, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => StockReservationFailed, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentProcessed, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentFailed, x => x.CorrelateById(m => m.Message.OrderId));
        
        // Initial state
        Initially(
            When(OrderCreated)
                .Then(context =>
                {
                    context.Saga.CustomerId = context.Message.CustomerId;
                    context.Saga.TotalAmount = context.Message.TotalAmount;
                    context.Saga.CreatedAt = DateTime.UtcNow;
                })
                .TransitionTo(ReservingStock)
        );
        
        // ขณะกำลัง reserve stock
        During(ReservingStock,
            When(StockReserved)
                .Then(context => context.Saga.StockReserved = true)
                .Publish(context => new ProcessPaymentCommand
                {
                    OrderId = context.Saga.CorrelationId,
                    Amount = context.Saga.TotalAmount,
                    CustomerId = context.Saga.CustomerId
                })
                .TransitionTo(ProcessingPayment),
                
            When(StockReservationFailed)
                .Then(context =>
                {
                    context.Saga.UpdatedAt = DateTime.UtcNow;
                })
                .Publish(context => new OrderCancelledEvent
                {
                    OrderId = context.Saga.CorrelationId,
                    Reason = "Stock reservation failed"
                })
                .TransitionTo(Cancelled)
        );
        
        // ขณะกำลัง process payment
        During(ProcessingPayment,
            When(PaymentProcessed)
                .Then(context =>
                {
                    context.Saga.PaymentProcessed = true;
                    context.Saga.UpdatedAt = DateTime.UtcNow;
                })
                .Publish(context => new OrderCompletedEvent
                {
                    OrderId = context.Saga.CorrelationId
                })
                .TransitionTo(Completed),
                
            When(PaymentFailed)
                .Publish(context => new ReleaseStockCommand
                {
                    OrderId = context.Saga.CorrelationId
                })
                .Publish(context => new OrderCancelledEvent
                {
                    OrderId = context.Saga.CorrelationId,
                    Reason = "Payment failed"
                })
                .TransitionTo(Cancelled)
        );
    }
}

// ลงทะเบียน Saga
builder.Services.AddMassTransit(x =>
{
    x.AddSagaStateMachine<OrderStateMachine, OrderSagaState>()
        .EntityFrameworkRepository(r =>
        {
            r.ConcurrencyMode = ConcurrencyMode.Optimistic;
            r.AddDbContext<DbContext, OrderDbContext>(
                (provider, builder) =>
                {
                    builder.UseNpgsql(connectionString);
                });
        });
});
```

### At-Least-Once Delivery และ Idempotency

RabbitMQ รับประกัน "at-least-once" delivery หมายความว่า message อาจถูกส่งมากกว่าหนึ่งครั้ง

```csharp
// Consumers/IdempotentOrderCreatedConsumer.cs
public class IdempotentOrderCreatedConsumer : IConsumer<OrderCreatedEvent>
{
    private readonly IInventoryRepository _repository;
    private readonly IProcessedMessageRepository _processedMessages;
    
    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        var messageId = context.MessageId ?? Guid.NewGuid();
        
        // ตรวจสอบว่าเคย process message นี้แล้วหรือยัง
        if (await _processedMessages.ExistsAsync(messageId))
        {
            // Skip - already processed (idempotent)
            return;
        }
        
        // Process message
        foreach (var line in context.Message.Lines)
        {
            await _repository.ReserveStockAsync(
                line.ProductId, 
                line.Quantity,
                context.Message.OrderId);
        }
        
        // Mark as processed
        await _processedMessages.MarkAsProcessedAsync(messageId, DateTime.UtcNow);
    }
}

// Repository สำหรับ tracking processed messages
public class ProcessedMessageRepository : IProcessedMessageRepository
{
    private readonly DbContext _db;
    
    public async Task<bool> ExistsAsync(Guid messageId)
    {
        return await _db.Set<ProcessedMessage>()
            .AnyAsync(m => m.MessageId == messageId);
    }
    
    public async Task MarkAsProcessedAsync(Guid messageId, DateTime processedAt)
    {
        _db.Set<ProcessedMessage>().Add(new ProcessedMessage
        {
            MessageId = messageId,
            ProcessedAt = processedAt
        });
        await _db.SaveChangesAsync();
    }
}
```

---

## Step 635: Outbox Pattern

### ปัญหา: Save to DB และ Publish Event ไม่ Atomic

```
// ปัญหา: ถ้า Publish ล้มเหลวหลัง Save สำเร็จ → data inconsistency
await _repository.SaveAsync(order);        // ✓ Success
await _bus.Publish(new OrderCreatedEvent); // ✗ FAIL! → event ไม่ถูกส่ง
```

หรือถ้า app crash ระหว่าง Save และ Publish → order ถูก save แต่ inventory ไม่รู้

### Solution: Transactional Outbox Pattern

บันทึก event ลงใน outbox table ในฐานข้อมูลเดียวกัน ในการ transaction เดียวกัน

```
Application → [DB Transaction] → Orders Table  ✓
                               → Outbox Table  ✓ (ทั้งคู่ commit หรือ rollback พร้อมกัน)

Background Worker → Poll Outbox → Publish to Message Bus → Mark as Published
```

### Outbox Table

```csharp
// Models/OutboxMessage.cs
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string Type { get; set; } = string.Empty;      // ชื่อ event type
    public string Content { get; set; } = string.Empty;   // JSON serialized event
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public string? Error { get; set; }
    public int RetryCount { get; set; }
}

// เพิ่ม migration
public class AddOutboxMessagesTable : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "OutboxMessages",
            columns: table => new
            {
                Id = table.Column<Guid>(nullable: false),
                Type = table.Column<string>(maxLength: 500, nullable: false),
                Content = table.Column<string>(nullable: false),
                CreatedAt = table.Column<DateTime>(nullable: false),
                ProcessedAt = table.Column<DateTime>(nullable: true),
                Error = table.Column<string>(nullable: true),
                RetryCount = table.Column<int>(nullable: false, defaultValue: 0)
            },
            constraints: table => table.PrimaryKey("PK_OutboxMessages", x => x.Id));
            
        migrationBuilder.CreateIndex(
            name: "IX_OutboxMessages_ProcessedAt",
            table: "OutboxMessages",
            column: "ProcessedAt");
    }
}
```

### บันทึกลง Outbox ใน Transaction เดียวกัน

```csharp
// Services/OrderService.cs
public class OrderService
{
    private readonly OrderDbContext _dbContext;
    
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        using var transaction = await _dbContext.Database.BeginTransactionAsync();
        
        try
        {
            // 1. สร้าง Order
            var order = new Order
            {
                Id = Guid.NewGuid(),
                CustomerId = request.CustomerId,
                Status = OrderStatus.Pending,
                CreatedAt = DateTime.UtcNow
            };
            
            _dbContext.Orders.Add(order);
            
            // 2. บันทึก Event ลง Outbox (ใน transaction เดียวกัน)
            var orderCreatedEvent = new OrderCreatedEvent
            {
                OrderId = order.Id,
                CustomerId = order.CustomerId,
                TotalAmount = order.TotalAmount,
                CreatedAt = order.CreatedAt
            };
            
            _dbContext.OutboxMessages.Add(new OutboxMessage
            {
                Id = Guid.NewGuid(),
                Type = typeof(OrderCreatedEvent).AssemblyQualifiedName!,
                Content = JsonSerializer.Serialize(orderCreatedEvent),
                CreatedAt = DateTime.UtcNow
            });
            
            // 3. Save ทั้งคู่พร้อมกัน (atomic)
            await _dbContext.SaveChangesAsync();
            await transaction.CommitAsync();
            
            return order;
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

### Outbox Publisher (Background Worker)

```csharp
// Workers/OutboxPublisherWorker.cs
public class OutboxPublisherWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OutboxPublisherWorker> _logger;
    
    public OutboxPublisherWorker(
        IServiceScopeFactory scopeFactory, 
        ILogger<OutboxPublisherWorker> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ProcessOutboxMessages(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing outbox messages");
            }
            
            // Poll ทุก 5 วินาที
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
    
    private async Task ProcessOutboxMessages(CancellationToken ct)
    {
        using var scope = _scopeFactory.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<OrderDbContext>();
        var bus = scope.ServiceProvider.GetRequiredService<IBus>();
        
        // ดึง messages ที่ยังไม่ได้ process (max 20 ต่อรอบ)
        var messages = await dbContext.OutboxMessages
            .Where(m => m.ProcessedAt == null && m.RetryCount < 5)
            .OrderBy(m => m.CreatedAt)
            .Take(20)
            .ToListAsync(ct);
            
        foreach (var message in messages)
        {
            try
            {
                // Deserialize event
                var eventType = Type.GetType(message.Type);
                if (eventType == null)
                {
                    _logger.LogWarning("Unknown event type: {Type}", message.Type);
                    continue;
                }
                
                var @event = JsonSerializer.Deserialize(message.Content, eventType);
                if (@event == null) continue;
                
                // Publish ไปยัง message bus
                await bus.Publish(@event, eventType, ct);
                
                // Mark as processed
                message.ProcessedAt = DateTime.UtcNow;
                
                _logger.LogInformation(
                    "Published outbox message {MessageId} of type {Type}", 
                    message.Id, message.Type);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, 
                    "Failed to publish outbox message {MessageId}", message.Id);
                    
                message.RetryCount++;
                message.Error = ex.Message;
            }
        }
        
        await dbContext.SaveChangesAsync(ct);
    }
}

// Program.cs
builder.Services.AddHostedService<OutboxPublisherWorker>();
```

### MassTransit Outbox (Built-in)

MassTransit มี Outbox pattern built-in ไม่ต้องสร้าง worker เอง

```csharp
// Program.cs - ใช้ MassTransit Outbox
builder.Services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<OrderDbContext>(o =>
    {
        o.UsePostgres();  // หรือ UseSqlServer(), UseMySql()
        o.UseBusOutbox();
        
        // กำหนด cleanup เพื่อลบ processed messages
        o.QueryDelay = TimeSpan.FromSeconds(1);
        o.QueryTimeout = TimeSpan.FromSeconds(30);
    });
    
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq");
        cfg.ConfigureEndpoints(context);
    });
});

// OrderDbContext ต้อง implement IOutboxDbContext
public class OrderDbContext : DbContext, IOutboxDbContext
{
    public DbSet<OutboxMessage> OutboxMessages { get; set; }
    public DbSet<OutboxState> OutboxState { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.AddInboxStateEntity();
        modelBuilder.AddOutboxMessageEntity();
        modelBuilder.AddOutboxStateEntity();
    }
}

// การใช้งาน - ต้อง inject IPublishEndpoint แทน IBus
public class OrderService
{
    private readonly OrderDbContext _dbContext;
    private readonly IPublishEndpoint _publishEndpoint;
    
    public async Task CreateOrderAsync(CreateOrderRequest request)
    {
        using var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);
        
        var order = new Order { Id = Guid.NewGuid(), /* ... */ };
        _dbContext.Orders.Add(order);
        
        // Publish จะถูก save ลง outbox อัตโนมัติ
        await _publishEndpoint.Publish(new OrderCreatedEvent
        {
            OrderId = order.Id,
            // ...
        });
        
        await _dbContext.SaveChangesAsync();
        scope.Complete();
    }
}
```

---

## Step 636-640: Simple Microservices Demo

### Overview ของ Demo

เราจะสร้าง 2 services ที่สื่อสารกัน:

```
Client
  │
  ▼
┌─────────────┐    HTTP     ┌──────────────────┐
│ API Gateway │────────────▶│  Order Service    │
│ (YARP)      │    HTTP     │  Port: 5001       │
│ Port: 5000  │────────────▶│  InventoryService │
└─────────────┘             │  Port: 5002       │
                            └──────────────────┘
                                    │
                             [RabbitMQ]
                                    │
                            ┌──────────────────┐
                            │ Inventory Service │
                            │  Port: 5002       │
                            └──────────────────┘
```

**Flow:**
1. Client สร้าง Order ผ่าน API Gateway
2. Order Service สร้าง order และ publish `OrderCreated` event
3. Inventory Service รับ event และ reserve stock
4. Inventory Service publish `StockReserved` event
5. Order Service อัปเดต status เป็น "Confirmed"

### Step 636: โครงสร้างโปรเจค

```
MicroservicesDemo/
├── docker-compose.yml
├── Contracts/                      # Shared contracts
│   └── Contracts.csproj
│   └── Events/
│       ├── OrderCreatedEvent.cs
│       └── StockReservedEvent.cs
├── ApiGateway/                     # YARP API Gateway
│   ├── ApiGateway.csproj
│   ├── Program.cs
│   └── appsettings.json
├── OrderService/                   # Order microservice
│   ├── OrderService.csproj
│   ├── Program.cs
│   ├── Controllers/
│   │   └── OrdersController.cs
│   ├── Models/
│   │   └── Order.cs
│   ├── Services/
│   │   └── OrderService.cs
│   └── Consumers/
│       └── StockReservedConsumer.cs
└── InventoryService/               # Inventory microservice
    ├── InventoryService.csproj
    ├── Program.cs
    ├── Controllers/
    │   └── InventoryController.cs
    ├── Models/
    │   └── InventoryItem.cs
    └── Consumers/
        └── OrderCreatedConsumer.cs
```

### Step 637: Shared Contracts

```csharp
// Contracts/Events/OrderCreatedEvent.cs
namespace Contracts.Events;

public record OrderCreatedEvent
{
    public Guid OrderId { get; init; }
    public Guid CustomerId { get; init; }
    public List<OrderItemDto> Items { get; init; } = new();
    public decimal TotalAmount { get; init; }
    public DateTime CreatedAt { get; init; }
}

public record OrderItemDto
{
    public Guid ProductId { get; init; }
    public string ProductName { get; init; } = string.Empty;
    public int Quantity { get; init; }
    public decimal UnitPrice { get; init; }
}

// Contracts/Events/StockReservedEvent.cs
namespace Contracts.Events;

public record StockReservedEvent
{
    public Guid OrderId { get; init; }
    public List<ReservationDto> Reservations { get; init; } = new();
    public DateTime ReservedAt { get; init; }
}

public record ReservationDto
{
    public Guid ProductId { get; init; }
    public int ReservedQuantity { get; init; }
}

// Contracts/Events/StockReservationFailedEvent.cs
namespace Contracts.Events;

public record StockReservationFailedEvent
{
    public Guid OrderId { get; init; }
    public Guid ProductId { get; init; }
    public string Reason { get; init; } = string.Empty;
}
```

### Step 638: Order Service

```csharp
// OrderService/Models/Order.cs
public class Order
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public List<OrderItem> Items { get; set; } = new();
    public OrderStatus Status { get; set; }
    public decimal TotalAmount { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
}

public class OrderItem
{
    public Guid ProductId { get; set; }
    public string ProductName { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

public enum OrderStatus
{
    Pending,
    ConfirmedStock,
    PaymentProcessing,
    Completed,
    Cancelled
}

// OrderService/Controllers/OrdersController.cs
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;
    private readonly ILogger<OrdersController> _logger;
    
    public OrdersController(IOrderService orderService, ILogger<OrdersController> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }
    
    [HttpGet]
    public async Task<ActionResult<List<Order>>> GetAll()
    {
        return Ok(await _orderService.GetAllAsync());
    }
    
    [HttpGet("{id:guid}")]
    public async Task<ActionResult<Order>> GetById(Guid id)
    {
        var order = await _orderService.GetByIdAsync(id);
        return order == null ? NotFound() : Ok(order);
    }
    
    [HttpPost]
    public async Task<ActionResult<Order>> Create([FromBody] CreateOrderRequest request)
    {
        var order = await _orderService.CreateOrderAsync(request);
        return CreatedAtAction(nameof(GetById), new { id = order.Id }, order);
    }
    
    [HttpDelete("{id:guid}")]
    public async Task<IActionResult> Cancel(Guid id)
    {
        var result = await _orderService.CancelOrderAsync(id);
        return result ? NoContent() : NotFound();
    }
}

// DTOs
public record CreateOrderRequest(
    Guid CustomerId,
    List<CreateOrderItemRequest> Items
);

public record CreateOrderItemRequest(
    Guid ProductId,
    string ProductName,
    int Quantity,
    decimal UnitPrice
);

// OrderService/Services/OrderService.cs
public class OrderService : IOrderService
{
    private readonly List<Order> _orders = new();  // In-memory for demo
    private readonly IBus _bus;
    private readonly ILogger<OrderService> _logger;
    
    public OrderService(IBus bus, ILogger<OrderService> logger)
    {
        _bus = bus;
        _logger = logger;
    }
    
    public Task<List<Order>> GetAllAsync() => Task.FromResult(_orders.ToList());
    
    public Task<Order?> GetByIdAsync(Guid id) =>
        Task.FromResult(_orders.FirstOrDefault(o => o.Id == id));
    
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = request.CustomerId,
            Items = request.Items.Select(i => new OrderItem
            {
                ProductId = i.ProductId,
                ProductName = i.ProductName,
                Quantity = i.Quantity,
                UnitPrice = i.UnitPrice
            }).ToList(),
            Status = OrderStatus.Pending,
            TotalAmount = request.Items.Sum(i => i.Quantity * i.UnitPrice),
            CreatedAt = DateTime.UtcNow
        };
        
        _orders.Add(order);
        
        // Publish event
        await _bus.Publish(new OrderCreatedEvent
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            Items = order.Items.Select(i => new OrderItemDto
            {
                ProductId = i.ProductId,
                ProductName = i.ProductName,
                Quantity = i.Quantity,
                UnitPrice = i.UnitPrice
            }).ToList(),
            TotalAmount = order.TotalAmount,
            CreatedAt = order.CreatedAt
        });
        
        _logger.LogInformation("Order {OrderId} created and event published", order.Id);
        return order;
    }
    
    public async Task UpdateStatusAsync(Guid orderId, OrderStatus status)
    {
        var order = _orders.FirstOrDefault(o => o.Id == orderId);
        if (order != null)
        {
            order.Status = status;
            order.UpdatedAt = DateTime.UtcNow;
            _logger.LogInformation("Order {OrderId} status updated to {Status}", orderId, status);
        }
        await Task.CompletedTask;
    }
    
    public Task<bool> CancelOrderAsync(Guid id)
    {
        var order = _orders.FirstOrDefault(o => o.Id == id);
        if (order == null) return Task.FromResult(false);
        order.Status = OrderStatus.Cancelled;
        return Task.FromResult(true);
    }
}

// OrderService/Consumers/StockReservedConsumer.cs
public class StockReservedConsumer : IConsumer<StockReservedEvent>
{
    private readonly IOrderService _orderService;
    private readonly ILogger<StockReservedConsumer> _logger;
    
    public StockReservedConsumer(IOrderService orderService, ILogger<StockReservedConsumer> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }
    
    public async Task Consume(ConsumeContext<StockReservedEvent> context)
    {
        _logger.LogInformation(
            "Stock reserved for Order {OrderId}", context.Message.OrderId);
            
        await _orderService.UpdateStatusAsync(
            context.Message.OrderId, 
            OrderStatus.ConfirmedStock);
    }
}

// OrderService/Consumers/StockReservationFailedConsumer.cs
public class StockReservationFailedConsumer : IConsumer<StockReservationFailedEvent>
{
    private readonly IOrderService _orderService;
    private readonly ILogger<StockReservationFailedConsumer> _logger;
    
    public StockReservationFailedConsumer(
        IOrderService orderService, 
        ILogger<StockReservationFailedConsumer> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }
    
    public async Task Consume(ConsumeContext<StockReservationFailedEvent> context)
    {
        _logger.LogWarning(
            "Stock reservation failed for Order {OrderId}: {Reason}",
            context.Message.OrderId, context.Message.Reason);
            
        await _orderService.UpdateStatusAsync(
            context.Message.OrderId, 
            OrderStatus.Cancelled);
    }
}

// OrderService/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

builder.Services.AddSingleton<IOrderService, OrderService>();

builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<StockReservedConsumer>();
    x.AddConsumer<StockReservationFailedConsumer>();
    
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host(builder.Configuration["RabbitMQ:Host"] ?? "localhost", "/", h =>
        {
            h.Username(builder.Configuration["RabbitMQ:Username"] ?? "guest");
            h.Password(builder.Configuration["RabbitMQ:Password"] ?? "guest");
        });
        
        cfg.ReceiveEndpoint("order-stock-events", e =>
        {
            e.ConfigureConsumer<StockReservedConsumer>(context);
            e.ConfigureConsumer<StockReservationFailedConsumer>(context);
        });
    });
});

builder.Services.AddHealthChecks();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();
app.MapControllers();
app.MapHealthChecks("/health");

app.Run();
```

### Step 639: Inventory Service

```csharp
// InventoryService/Models/InventoryItem.cs
public class InventoryItem
{
    public Guid ProductId { get; set; }
    public string Name { get; set; } = string.Empty;
    public int AvailableQuantity { get; set; }
    public int ReservedQuantity { get; set; }
    
    public int TotalQuantity => AvailableQuantity + ReservedQuantity;
}

// InventoryService/Controllers/InventoryController.cs
[ApiController]
[Route("api/inventory")]
public class InventoryController : ControllerBase
{
    private readonly IInventoryService _service;
    
    public InventoryController(IInventoryService service)
    {
        _service = service;
    }
    
    [HttpGet]
    public async Task<ActionResult<List<InventoryItem>>> GetAll()
    {
        return Ok(await _service.GetAllAsync());
    }
    
    [HttpGet("{productId:guid}")]
    public async Task<ActionResult<InventoryItem>> GetByProductId(Guid productId)
    {
        var item = await _service.GetByProductIdAsync(productId);
        return item == null ? NotFound() : Ok(item);
    }
    
    [HttpPost("seed")]
    public async Task<IActionResult> Seed()
    {
        await _service.SeedDataAsync();
        return Ok("Seed data added");
    }
}

// InventoryService/Services/InventoryService.cs
public class InventoryService : IInventoryService
{
    private readonly List<InventoryItem> _items = new();
    private readonly ILogger<InventoryService> _logger;
    
    public InventoryService(ILogger<InventoryService> logger)
    {
        _logger = logger;
        // เพิ่ม seed data
        SeedDataAsync().Wait();
    }
    
    public Task<List<InventoryItem>> GetAllAsync() => Task.FromResult(_items.ToList());
    
    public Task<InventoryItem?> GetByProductIdAsync(Guid productId) =>
        Task.FromResult(_items.FirstOrDefault(i => i.ProductId == productId));
    
    public Task<bool> ReserveStockAsync(Guid productId, int quantity)
    {
        var item = _items.FirstOrDefault(i => i.ProductId == productId);
        
        if (item == null || item.AvailableQuantity < quantity)
        {
            _logger.LogWarning(
                "Cannot reserve {Quantity} of product {ProductId}. Available: {Available}",
                quantity, productId, item?.AvailableQuantity ?? 0);
            return Task.FromResult(false);
        }
        
        item.AvailableQuantity -= quantity;
        item.ReservedQuantity += quantity;
        
        _logger.LogInformation(
            "Reserved {Quantity} of product {ProductId}. Remaining: {Remaining}",
            quantity, productId, item.AvailableQuantity);
            
        return Task.FromResult(true);
    }
    
    public Task SeedDataAsync()
    {
        if (_items.Any()) return Task.CompletedTask;
        
        _items.AddRange(new[]
        {
            new InventoryItem 
            { 
                ProductId = Guid.Parse("00000000-0000-0000-0000-000000000001"),
                Name = "Laptop",
                AvailableQuantity = 50
            },
            new InventoryItem 
            { 
                ProductId = Guid.Parse("00000000-0000-0000-0000-000000000002"),
                Name = "Mouse",
                AvailableQuantity = 200
            },
            new InventoryItem 
            { 
                ProductId = Guid.Parse("00000000-0000-0000-0000-000000000003"),
                Name = "Keyboard",
                AvailableQuantity = 150
            }
        });
        
        return Task.CompletedTask;
    }
}

// InventoryService/Consumers/OrderCreatedConsumer.cs
public class OrderCreatedConsumer : IConsumer<OrderCreatedEvent>
{
    private readonly IInventoryService _service;
    private readonly IBus _bus;
    private readonly ILogger<OrderCreatedConsumer> _logger;
    
    public OrderCreatedConsumer(
        IInventoryService service, 
        IBus bus, 
        ILogger<OrderCreatedConsumer> logger)
    {
        _service = service;
        _bus = bus;
        _logger = logger;
    }
    
    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        var order = context.Message;
        _logger.LogInformation("Processing OrderCreated for Order {OrderId}", order.OrderId);
        
        var reservations = new List<ReservationDto>();
        
        foreach (var item in order.Items)
        {
            var reserved = await _service.ReserveStockAsync(item.ProductId, item.Quantity);
            
            if (!reserved)
            {
                // ส่ง failure event
                await _bus.Publish(new StockReservationFailedEvent
                {
                    OrderId = order.OrderId,
                    ProductId = item.ProductId,
                    Reason = $"Insufficient stock for product {item.ProductName}"
                });
                
                _logger.LogWarning(
                    "Stock reservation failed for Order {OrderId}, Product {ProductId}",
                    order.OrderId, item.ProductId);
                return;
            }
            
            reservations.Add(new ReservationDto
            {
                ProductId = item.ProductId,
                ReservedQuantity = item.Quantity
            });
        }
        
        // ส่ง success event
        await _bus.Publish(new StockReservedEvent
        {
            OrderId = order.OrderId,
            Reservations = reservations,
            ReservedAt = DateTime.UtcNow
        });
        
        _logger.LogInformation(
            "Stock reserved successfully for Order {OrderId}", order.OrderId);
    }
}

// InventoryService/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

builder.Services.AddSingleton<IInventoryService, InventoryService>();

builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderCreatedConsumer>();
    
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host(builder.Configuration["RabbitMQ:Host"] ?? "localhost", "/", h =>
        {
            h.Username(builder.Configuration["RabbitMQ:Username"] ?? "guest");
            h.Password(builder.Configuration["RabbitMQ:Password"] ?? "guest");
        });
        
        cfg.ReceiveEndpoint("inventory-order-created", e =>
        {
            e.ConfigureConsumer<OrderCreatedConsumer>(context);
        });
    });
});

builder.Services.AddHealthChecks();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();
app.MapControllers();
app.MapHealthChecks("/health");

app.Run();
```

### Step 640: docker-compose และการ Run

```yaml
# docker-compose.yml
version: '3.8'

services:
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 10s
      timeout: 5s
      retries: 5

  order-service:
    build:
      context: ./OrderService
      dockerfile: Dockerfile
    ports:
      - "5001:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - RabbitMQ__Host=rabbitmq
      - RabbitMQ__Username=guest
      - RabbitMQ__Password=guest
    depends_on:
      rabbitmq:
        condition: service_healthy

  inventory-service:
    build:
      context: ./InventoryService
      dockerfile: Dockerfile
    ports:
      - "5002:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - RabbitMQ__Host=rabbitmq
      - RabbitMQ__Username=guest
      - RabbitMQ__Password=guest
    depends_on:
      rabbitmq:
        condition: service_healthy

  api-gateway:
    build:
      context: ./ApiGateway
      dockerfile: Dockerfile
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    depends_on:
      - order-service
      - inventory-service
```

```dockerfile
# Dockerfile (ใช้กับทุก service)
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY ["OrderService.csproj", "."]
# หรือ Contracts ถ้าต้องการ
COPY ["../Contracts/Contracts.csproj", "../Contracts/"]
RUN dotnet restore

COPY . .
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "OrderService.dll"]
```

### API Gateway Configuration

```json
// ApiGateway/appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "orders": {
        "ClusterId": "order-cluster",
        "Match": {
          "Path": "/api/orders/{**catch-all}"
        }
      },
      "inventory": {
        "ClusterId": "inventory-cluster",
        "Match": {
          "Path": "/api/inventory/{**catch-all}"
        }
      }
    },
    "Clusters": {
      "order-cluster": {
        "Destinations": {
          "order-service": {
            "Address": "http://order-service:8080"
          }
        }
      },
      "inventory-cluster": {
        "Destinations": {
          "inventory-service": {
            "Address": "http://inventory-service:8080"
          }
        }
      }
    }
  }
}
```

```csharp
// ApiGateway/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

app.MapReverseProxy();
app.Run();
```

### การทดสอบ Demo

```bash
# เริ่ม services ทั้งหมด
docker-compose up -d

# ตรวจสอบ logs
docker-compose logs -f

# Seed inventory data
curl -X POST http://localhost:5000/api/inventory/seed

# ตรวจสอบ inventory
curl http://localhost:5000/api/inventory

# สร้าง Order
curl -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "11111111-1111-1111-1111-111111111111",
    "items": [
      {
        "productId": "00000000-0000-0000-0000-000000000001",
        "productName": "Laptop",
        "quantity": 2,
        "unitPrice": 50000
      },
      {
        "productId": "00000000-0000-0000-0000-000000000002",
        "productName": "Mouse",
        "quantity": 1,
        "unitPrice": 500
      }
    ]
  }'

# ตรวจสอบ Order status (รอสักครู่)
curl http://localhost:5000/api/orders

# ตรวจสอบ Inventory หลังจาก reserve
curl http://localhost:5000/api/inventory

# เปิด RabbitMQ Management UI
# http://localhost:15672 (guest/guest)
```

### Expected Output

```
# Order status progression:
Pending → ConfirmedStock (เมื่อ inventory reserve สำเร็จ)

# Inventory หลังจากสร้าง Order 2 Laptops, 1 Mouse:
Laptop:   Available=48, Reserved=2
Mouse:    Available=199, Reserved=1
Keyboard: Available=150, Reserved=0
```

### Testing ด้วย Integration Tests

```csharp
// Tests/IntegrationTests/OrderFlowTests.cs
public class OrderFlowTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _orderClient;
    private readonly HttpClient _inventoryClient;
    
    [Fact]
    public async Task CreateOrder_ShouldReserveInventory()
    {
        // Arrange
        var request = new CreateOrderRequest(
            CustomerId: Guid.NewGuid(),
            Items: new List<CreateOrderItemRequest>
            {
                new(
                    ProductId: Guid.Parse("00000000-0000-0000-0000-000000000001"),
                    ProductName: "Laptop",
                    Quantity: 1,
                    UnitPrice: 50000m
                )
            }
        );
        
        // Act - สร้าง Order
        var orderResponse = await _orderClient.PostAsJsonAsync("/api/orders", request);
        orderResponse.EnsureSuccessStatusCode();
        var order = await orderResponse.Content.ReadFromJsonAsync<Order>();
        
        // Wait for message processing
        await Task.Delay(2000);
        
        // Assert - ตรวจสอบ Order status
        var updatedOrderResponse = await _orderClient.GetAsync($"/api/orders/{order!.Id}");
        var updatedOrder = await updatedOrderResponse.Content.ReadFromJsonAsync<Order>();
        
        Assert.Equal(OrderStatus.ConfirmedStock, updatedOrder!.Status);
        
        // Assert - ตรวจสอบ Inventory
        var inventoryResponse = await _inventoryClient.GetAsync(
            $"/api/inventory/00000000-0000-0000-0000-000000000001");
        var inventory = await inventoryResponse.Content.ReadFromJsonAsync<InventoryItem>();
        
        Assert.Equal(49, inventory!.AvailableQuantity);
        Assert.Equal(1, inventory.ReservedQuantity);
    }
}
```

---

## สรุปตาราง Steps 631-640

| Step | หัวข้อ | เทคโนโลยีหลัก | สิ่งที่เรียนรู้ |
|------|--------|--------------|---------------|
| 631 | Microservices Overview | Architecture patterns | Monolith vs Microservices trade-offs, Bounded Context, Patterns |
| 632 | API Gateway with YARP | YARP, Rate Limiting | Routing, Load balancing, Auth at gateway |
| 633 | Service Communication | HttpClientFactory, Polly, gRPC | Resilience patterns, Typed clients, Health checks |
| 634 | Message Bus | MassTransit, RabbitMQ | Publish/Subscribe, Saga, Idempotency |
| 635 | Outbox Pattern | EF Core, Background Worker | Atomic publish, At-least-once delivery |
| 636 | Demo Setup | docker-compose | Multi-service architecture, Contracts |
| 637 | Shared Contracts | C# Records | Event/Command definitions, Versioning |
| 638 | Order Service | MassTransit, REST API | Publisher, Event handler |
| 639 | Inventory Service | MassTransit, REST API | Consumer, Stock management |
| 640 | Integration & Testing | Docker, HTTP | End-to-end flow, Testing async systems |

---

## Checklist ทักษะที่ได้จากบทนี้

- [ ] เข้าใจ trade-offs ระหว่าง Monolith และ Microservices
- [ ] กำหนด Bounded Context และ Service boundaries ได้
- [ ] สร้าง API Gateway ด้วย YARP พร้อม routing, load balancing
- [ ] ใช้ HttpClientFactory สร้าง Typed HTTP clients
- [ ] ติดตั้ง Polly สำหรับ retry, circuit breaker, timeout
- [ ] สร้าง gRPC services สำหรับ high-performance internal communication
- [ ] Setup MassTransit กับ RabbitMQ
- [ ] สร้าง Event consumers ด้วย `IConsumer<T>`
- [ ] Implement Saga/State Machine สำหรับ long-running processes
- [ ] เข้าใจและ implement Outbox Pattern
- [ ] สร้าง multi-service application ด้วย docker-compose

---

## แนวทางปฏิบัติที่ดี (Best Practices)

### 1. Design for Failure

```csharp
// ทุก service-to-service call ต้องมี resilience
services.AddHttpClient<IInventoryClient, InventoryClient>()
    .AddStandardResilienceHandler();  // retry + circuit breaker + timeout
```

### 2. Versioning

```csharp
// ต้อง version API และ message contracts
[Route("api/v1/orders")]
public class OrdersV1Controller : ControllerBase { }

[Route("api/v2/orders")]
public class OrdersV2Controller : ControllerBase { }

// Version message contracts ด้วย interface
public interface IOrderCreated
{
    Guid OrderId { get; }
    DateTime CreatedAt { get; }
}

public record OrderCreatedV1 : IOrderCreated
{
    public Guid OrderId { get; init; }
    public DateTime CreatedAt { get; init; }
    // V1 fields...
}

public record OrderCreatedV2 : IOrderCreated
{
    public Guid OrderId { get; init; }
    public DateTime CreatedAt { get; init; }
    // V2 fields (backward compatible)
    public string? CustomerEmail { get; init; }  // nullable = backward compatible
}
```

### 3. Observability

```csharp
// ใส่ Tracing ในทุก service
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddSource("MassTransit")
            .AddOtlpExporter();
    })
    .WithMetrics(metrics =>
    {
        metrics
            .AddAspNetCoreInstrumentation()
            .AddRuntimeInstrumentation()
            .AddPrometheusExporter();
    });
```

### 4. Correlation IDs

```csharp
// Propagate correlation ID ข้าม services
app.Use(async (context, next) =>
{
    if (!context.Request.Headers.TryGetValue("X-Correlation-Id", out var correlationId))
    {
        correlationId = Guid.NewGuid().ToString();
        context.Request.Headers["X-Correlation-Id"] = correlationId;
    }
    
    context.Response.Headers["X-Correlation-Id"] = correlationId;
    
    using (LogContext.PushProperty("CorrelationId", correlationId.ToString()))
    {
        await next();
    }
});
```

---

## การนำทาง

- **Previous**: [Part 63 - Event Sourcing](./part63-event-sourcing.md)
- **Next**: [Part 65 - ML.NET](./part65-mlnet.md)

---

*จบ Part 64: Microservices Architecture*

> **หมายเหตุ**: Microservices เป็น complex architecture ที่มี trade-offs มาก  
> เริ่มต้นด้วย Modular Monolith แล้วค่อย extract เป็น services เมื่อมีเหตุผลที่ชัดเจน  
> และอย่าลืมลงทุนใน observability (logging, tracing, metrics) ก่อนเสมอ
