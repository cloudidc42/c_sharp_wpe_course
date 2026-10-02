# Part 91: Final Capstone - Complete System Integration
## ขั้นตอนที่ 901-910: การรวมระบบ ShopThai ทั้งหมด

> **ระดับ**: ขั้นสูง (Advanced) | **เวลาเรียน**: ~10 ชั่วโมง | **ข้อกำหนด**: Part 81-90 ทั้งหมด

---

## ภาพรวมของ Part นี้

ใน Part สุดท้ายนี้ เราจะนำทุกสิ่งที่เรียนมาตลอด 90 Part มารวมกันเป็นระบบ **ShopThai** ที่สมบูรณ์และพร้อมใช้งานในระดับ Production เราจะสร้าง API Gateway, ตั้งค่า Service Discovery, ใช้ Distributed Tracing, เขียน Docker Compose สำหรับทุก Service และทดสอบ End-to-End

```
┌─────────────────────────────────────────────────────────────────┐
│                    ShopThai System Overview                      │
│                                                                 │
│  Client ──► API Gateway (YARP) ──► Catalog Service              │
│                    │            ──► Order Service               │
│                    │            ──► Payment Service             │
│                    │            ──► Notification Service        │
│                    │                                            │
│             Rate Limiting                                       │
│             Auth Middleware                                     │
│             Load Balancing                                      │
│                                                                 │
│  Infrastructure:                                                │
│  PostgreSQL | Redis | RabbitMQ | Elasticsearch | Jaeger | SEQ   │
└─────────────────────────────────────────────────────────────────┘
```

---

## ขั้นตอนที่ 901: Integration Architecture - โครงสร้างการรวมระบบ ShopThai

### แนวคิดสถาปัตยกรรม (Architecture Concepts)

ระบบ ShopThai ใช้รูปแบบ **Microservices Architecture** โดยมีส่วนประกอบหลักดังนี้:

```
┌──────────────────────────────────────────────────────────────────────┐
│                        ShopThai Architecture                          │
│                                                                      │
│  ┌──────────┐    ┌─────────────────────────────────────────────────┐ │
│  │  Client  │───►│              API Gateway (YARP)                 │ │
│  │  (Web/   │    │  ┌──────────┐ ┌──────────┐ ┌─────────────────┐ │ │
│  │   App)   │    │  │  Auth    │ │  Rate    │ │   Load Balance  │ │ │
│  └──────────┘    │  │Middleware│ │ Limiting │ │   & Routing     │ │ │
│                  │  └──────────┘ └──────────┘ └─────────────────┘ │ │
│                  └──────────────────┬────────────────────────────┘ │
│                                     │                               │
│            ┌──────────┬─────────────┼────────────┬──────────┐      │
│            ▼          ▼             ▼            ▼          ▼      │
│     ┌──────────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │
│     │ Catalog  │ │  Order  │ │ Payment  │ │Notif.    │ │User    │ │
│     │ Service  │ │ Service │ │ Service  │ │Service   │ │Service │ │
│     └────┬─────┘ └────┬────┘ └────┬─────┘ └────┬─────┘ └───┬────┘ │
│          │            │           │             │            │     │
│  ┌───────┴────────────┴───────────┴─────────────┴────────────┴───┐ │
│  │                     Message Bus (RabbitMQ)                    │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌────────────┐ ┌─────────┐ ┌──────────────┐ ┌───────────────────┐ │
│  │ PostgreSQL │ │  Redis  │ │Elasticsearch │ │Jaeger + SEQ (Obs) │ │
│  └────────────┘ └─────────┘ └──────────────┘ └───────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

### โครงสร้างโปรเจกต์ (Project Structure)

```
ShopThai/
├── src/
│   ├── ApiGateway/
│   │   ├── ShopThai.ApiGateway.csproj
│   │   ├── Program.cs
│   │   ├── appsettings.json
│   │   └── Middleware/
│   │       ├── AuthenticationMiddleware.cs
│   │       └── RateLimitingMiddleware.cs
│   ├── Services/
│   │   ├── CatalogService/
│   │   │   ├── ShopThai.CatalogService.csproj
│   │   │   ├── Program.cs
│   │   │   ├── Controllers/
│   │   │   ├── Domain/
│   │   │   └── Infrastructure/
│   │   ├── OrderService/
│   │   ├── PaymentService/
│   │   └── NotificationService/
│   └── Shared/
│       ├── ShopThai.Shared.Contracts/
│       ├── ShopThai.Shared.Infrastructure/
│       └── ShopThai.Shared.Events/
├── tests/
│   ├── IntegrationTests/
│   └── E2ETests/
├── docker/
│   ├── docker-compose.yml
│   ├── docker-compose.override.yml
│   └── scripts/
│       ├── init-db.sql
│       └── seed-data.sql
└── docs/
    └── architecture.md
```

### Shared Contracts - สัญญาร่วมระหว่าง Services

```csharp
// src/Shared/ShopThai.Shared.Contracts/Events/OrderEvents.cs
namespace ShopThai.Shared.Contracts.Events;

public record OrderCreatedEvent
{
    public Guid OrderId { get; init; }
    public Guid CustomerId { get; init; }
    public List<OrderItemDto> Items { get; init; } = [];
    public decimal TotalAmount { get; init; }
    public string Currency { get; init; } = "THB";
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
    public string CorrelationId { get; init; } = Guid.NewGuid().ToString();
}

public record OrderItemDto
{
    public Guid ProductId { get; init; }
    public string ProductName { get; init; } = string.Empty;
    public int Quantity { get; init; }
    public decimal UnitPrice { get; init; }
}

public record PaymentProcessedEvent
{
    public Guid PaymentId { get; init; }
    public Guid OrderId { get; init; }
    public decimal Amount { get; init; }
    public PaymentStatus Status { get; init; }
    public string TransactionId { get; init; } = string.Empty;
    public DateTime ProcessedAt { get; init; } = DateTime.UtcNow;
    public string CorrelationId { get; init; } = string.Empty;
}

public record InventoryReservedEvent
{
    public Guid OrderId { get; init; }
    public List<ReservedItemDto> ReservedItems { get; init; } = [];
    public bool Success { get; init; }
    public string? FailureReason { get; init; }
    public string CorrelationId { get; init; } = string.Empty;
}

public record ReservedItemDto
{
    public Guid ProductId { get; init; }
    public int Quantity { get; init; }
    public int WarehouseId { get; init; }
}

public record ShipmentCreatedEvent
{
    public Guid ShipmentId { get; init; }
    public Guid OrderId { get; init; }
    public string TrackingNumber { get; init; } = string.Empty;
    public string Carrier { get; init; } = string.Empty;
    public DateTime EstimatedDelivery { get; init; }
    public string CorrelationId { get; init; } = string.Empty;
}

public enum PaymentStatus
{
    Pending,
    Authorized,
    Captured,
    Failed,
    Refunded
}
```

---

## ขั้นตอนที่ 902: API Gateway Setup (YARP) - การตั้งค่า API Gateway

### ติดตั้ง YARP (Yet Another Reverse Proxy)

```xml
<!-- src/ApiGateway/ShopThai.ApiGateway.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Yarp.ReverseProxy" Version="2.1.0" />
    <PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="8.0.0" />
    <PackageReference Include="AspNetCoreRateLimit" Version="5.0.0" />
    <PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.7.0" />
    <PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.7.0" />
    <PackageReference Include="OpenTelemetry.Exporter.Jaeger" Version="1.5.1" />
  </ItemGroup>
</Project>
```

### Program.cs สำหรับ API Gateway

```csharp
// src/ApiGateway/Program.cs
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.RateLimiting;
using Microsoft.IdentityModel.Tokens;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// ============================================================
// 1. YARP - Reverse Proxy Configuration
// ============================================================
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddTransforms(context =>
    {
        // เพิ่ม Correlation ID ให้กับทุก Request ที่ส่งไปยัง downstream services
        context.AddRequestTransform(async transformContext =>
        {
            if (!transformContext.HttpContext.Request.Headers.ContainsKey("X-Correlation-ID"))
            {
                transformContext.ProxyRequest.Headers.Add(
                    "X-Correlation-ID",
                    Guid.NewGuid().ToString());
            }
        });

        // เพิ่ม X-Forwarded-For header
        context.UseDefaultForwarders();
    });

// ============================================================
// 2. JWT Authentication
// ============================================================
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Secret"]
                    ?? throw new InvalidOperationException("JWT Secret not configured"))),
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidateAudience = true,
            ValidAudience = builder.Configuration["Jwt:Audience"],
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };

        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = context =>
            {
                context.Response.Headers.Append("X-Auth-Error", "Token validation failed");
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();

// ============================================================
// 3. Rate Limiting
// ============================================================
builder.Services.AddRateLimiter(options =>
{
    // Global Rate Limit - จำกัด 1000 requests ต่อนาทีสำหรับทุก IP
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
    {
        var clientIp = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        return RateLimitPartition.GetFixedWindowLimiter(clientIp, _ =>
            new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 1000,
                Window = TimeSpan.FromMinutes(1)
            });
    });

    // API Rate Limit สำหรับ Authenticated Users - 5000 requests ต่อนาที
    options.AddPolicy("authenticated", context =>
    {
        var userId = context.User.FindFirst("sub")?.Value ?? "anonymous";
        return RateLimitPartition.GetSlidingWindowLimiter(userId, _ =>
            new SlidingWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 5000,
                Window = TimeSpan.FromMinutes(1),
                SegmentsPerWindow = 6
            });
    });

    // Strict Rate Limit สำหรับ Payment Endpoints - 10 requests ต่อนาที
    options.AddPolicy("payment-strict", context =>
    {
        var userId = context.User.FindFirst("sub")?.Value ?? "anonymous";
        return RateLimitPartition.GetTokenBucketLimiter($"payment-{userId}", _ =>
            new TokenBucketRateLimiterOptions
            {
                AutoReplenishment = true,
                TokenLimit = 10,
                TokensPerPeriod = 10,
                ReplenishmentPeriod = TimeSpan.FromMinutes(1)
            });
    });

    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        context.HttpContext.Response.Headers["Retry-After"] = "60";
        await context.HttpContext.Response.WriteAsJsonAsync(new
        {
            error = "Rate limit exceeded",
            message = "คุณส่ง request มากเกินไป กรุณารอสักครู่แล้วลองใหม่",
            retryAfter = 60
        }, token);
    };
});

// ============================================================
// 4. OpenTelemetry - Distributed Tracing
// ============================================================
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource =>
        resource.AddService("ShopThai.ApiGateway"))
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation(options =>
            {
                options.RecordException = true;
                options.Filter = ctx =>
                    !ctx.Request.Path.StartsWithSegments("/health");
            })
            .AddHttpClientInstrumentation()
            .AddJaegerExporter(options =>
            {
                options.AgentHost = builder.Configuration["Jaeger:Host"] ?? "jaeger";
                options.AgentPort = int.Parse(builder.Configuration["Jaeger:Port"] ?? "6831");
            });
    });

// ============================================================
// 5. Health Checks
// ============================================================
builder.Services.AddHealthChecks()
    .AddUrlGroup(new Uri("http://catalog-service/health"), "catalog-service")
    .AddUrlGroup(new Uri("http://order-service/health"), "order-service")
    .AddUrlGroup(new Uri("http://payment-service/health"), "payment-service")
    .AddUrlGroup(new Uri("http://notification-service/health"), "notification-service");

// ============================================================
// 6. CORS
// ============================================================
builder.Services.AddCors(options =>
{
    options.AddPolicy("ShopThaiCors", policy =>
    {
        policy.WithOrigins(
                builder.Configuration.GetSection("AllowedOrigins").Get<string[]>()
                    ?? ["http://localhost:3000"])
            .AllowAnyHeader()
            .AllowAnyMethod()
            .AllowCredentials()
            .WithExposedHeaders("X-Correlation-ID", "X-RateLimit-Remaining");
    });
});

var app = builder.Build();

// Middleware Pipeline
app.UseHttpsRedirection();
app.UseCors("ShopThaiCors");
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();

// Health Check Endpoints
app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready", new Microsoft.AspNetCore.Diagnostics.HealthChecks.HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

// YARP - Route all traffic
app.MapReverseProxy(proxyPipeline =>
{
    // Middleware เฉพาะสำหรับ Proxy requests
    proxyPipeline.UseSessionAffinity();
    proxyPipeline.UseLoadBalancing();
    proxyPipeline.UsePassiveHealthChecks();
});

app.Run();
```

### appsettings.json สำหรับ YARP Routing

```json
{
  "ReverseProxy": {
    "Routes": {
      "catalog-route": {
        "ClusterId": "catalog-cluster",
        "Match": {
          "Path": "/api/catalog/{**catch-all}"
        },
        "Transforms": [
          { "PathPattern": "/api/{**catch-all}" }
        ]
      },
      "order-route": {
        "ClusterId": "order-cluster",
        "Match": {
          "Path": "/api/orders/{**catch-all}"
        },
        "AuthorizationPolicy": "default",
        "RateLimiterPolicy": "authenticated",
        "Transforms": [
          { "PathPattern": "/api/{**catch-all}" }
        ]
      },
      "payment-route": {
        "ClusterId": "payment-cluster",
        "Match": {
          "Path": "/api/payments/{**catch-all}"
        },
        "AuthorizationPolicy": "default",
        "RateLimiterPolicy": "payment-strict",
        "Transforms": [
          { "PathPattern": "/api/{**catch-all}" }
        ]
      },
      "notification-route": {
        "ClusterId": "notification-cluster",
        "Match": {
          "Path": "/api/notifications/{**catch-all}"
        },
        "AuthorizationPolicy": "default"
      }
    },
    "Clusters": {
      "catalog-cluster": {
        "LoadBalancingPolicy": "RoundRobin",
        "HealthCheck": {
          "Passive": {
            "Enabled": true,
            "ReactivationPeriod": "00:00:30"
          },
          "Active": {
            "Enabled": true,
            "Interval": "00:00:10",
            "Timeout": "00:00:05",
            "Policy": "ConsecutiveFailures",
            "Path": "/health"
          }
        },
        "Destinations": {
          "destination1": {
            "Address": "http://catalog-service:8080"
          }
        }
      },
      "order-cluster": {
        "LoadBalancingPolicy": "LeastRequests",
        "Destinations": {
          "destination1": {
            "Address": "http://order-service:8080"
          }
        }
      },
      "payment-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "http://payment-service:8080"
          }
        }
      },
      "notification-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "http://notification-service:8080"
          }
        }
      }
    }
  },
  "Jwt": {
    "Secret": "ShopThai-Super-Secret-Key-2024-Min32Chars!",
    "Issuer": "ShopThai",
    "Audience": "ShopThai-Clients"
  },
  "Jaeger": {
    "Host": "jaeger",
    "Port": "6831"
  },
  "AllowedOrigins": ["http://localhost:3000", "https://shopthai.com"]
}
```

---

## ขั้นตอนที่ 903: Service Discovery และ Health Aggregation

### Shared Infrastructure สำหรับ Health Checks

```csharp
// src/Shared/ShopThai.Shared.Infrastructure/Health/ServiceHealthExtensions.cs
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Diagnostics.HealthChecks;
using System.Text.Json;

namespace ShopThai.Shared.Infrastructure.Health;

public static class ServiceHealthExtensions
{
    /// <summary>
    /// เพิ่ม Health Checks มาตรฐานสำหรับทุก ShopThai Service
    /// </summary>
    public static IHealthChecksBuilder AddShopThaiHealthChecks(
        this IHealthChecksBuilder builder,
        string serviceName,
        string connectionString,
        string redisConnection)
    {
        return builder
            .AddNpgsql(connectionString,
                name: $"{serviceName}-postgres",
                tags: ["database", "ready"])
            .AddRedis(redisConnection,
                name: $"{serviceName}-redis",
                tags: ["cache", "ready"])
            .AddCheck<SystemResourcesHealthCheck>(
                $"{serviceName}-resources",
                tags: ["system"]);
    }

    /// <summary>
    /// ตั้งค่า Health Endpoints มาตรฐาน
    /// </summary>
    public static WebApplication MapShopThaiHealthEndpoints(this WebApplication app)
    {
        // Liveness Probe - ตรวจสอบว่า Service ยังทำงานอยู่
        app.MapHealthChecks("/health/live", new HealthCheckOptions
        {
            Predicate = _ => false,
            ResponseWriter = WriteHealthResponse
        });

        // Readiness Probe - ตรวจสอบว่า Service พร้อมรับ Traffic
        app.MapHealthChecks("/health/ready", new HealthCheckOptions
        {
            Predicate = check => check.Tags.Contains("ready"),
            ResponseWriter = WriteHealthResponse
        });

        // Startup Probe - ตรวจสอบว่า Service เริ่มต้นสำเร็จ
        app.MapHealthChecks("/health/startup", new HealthCheckOptions
        {
            Predicate = check => check.Tags.Contains("startup"),
            ResponseWriter = WriteHealthResponse
        });

        // Detailed Health Check สำหรับ Internal Use
        app.MapHealthChecks("/health", new HealthCheckOptions
        {
            ResponseWriter = WriteDetailedHealthResponse
        });

        return app;
    }

    private static async Task WriteHealthResponse(
        HttpContext context,
        HealthReport report)
    {
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = report.Status == HealthStatus.Healthy
            ? StatusCodes.Status200OK
            : StatusCodes.Status503ServiceUnavailable;

        var response = new
        {
            status = report.Status.ToString(),
            duration = report.TotalDuration.TotalMilliseconds
        };

        await JsonSerializer.SerializeAsync(context.Response.Body, response);
    }

    private static async Task WriteDetailedHealthResponse(
        HttpContext context,
        HealthReport report)
    {
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = report.Status == HealthStatus.Healthy
            ? StatusCodes.Status200OK
            : StatusCodes.Status503ServiceUnavailable;

        var response = new
        {
            status = report.Status.ToString(),
            totalDurationMs = report.TotalDuration.TotalMilliseconds,
            checks = report.Entries.Select(entry => new
            {
                name = entry.Key,
                status = entry.Value.Status.ToString(),
                description = entry.Value.Description,
                durationMs = entry.Value.Duration.TotalMilliseconds,
                tags = entry.Value.Tags,
                exception = entry.Value.Exception?.Message
            })
        };

        await JsonSerializer.SerializeAsync(context.Response.Body, response,
            new JsonSerializerOptions { WriteIndented = true });
    }
}

// Custom Health Check สำหรับตรวจสอบ System Resources
public class SystemResourcesHealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var memoryInfo = GC.GetGCMemoryInfo();
        var totalMemoryMb = memoryInfo.TotalAvailableMemoryBytes / (1024 * 1024);
        var usedMemoryMb = GC.GetTotalMemory(false) / (1024 * 1024);
        var memoryUsagePercent = (double)usedMemoryMb / totalMemoryMb * 100;

        var data = new Dictionary<string, object>
        {
            ["totalMemoryMB"] = totalMemoryMb,
            ["usedMemoryMB"] = usedMemoryMb,
            ["memoryUsagePercent"] = Math.Round(memoryUsagePercent, 2),
            ["processorCount"] = Environment.ProcessorCount,
            ["threadCount"] = System.Diagnostics.Process.GetCurrentProcess().Threads.Count
        };

        return memoryUsagePercent switch
        {
            > 95 => Task.FromResult(HealthCheckResult.Unhealthy(
                "หน่วยความจำใกล้เต็มแล้ว - System memory critical", data: data)),
            > 80 => Task.FromResult(HealthCheckResult.Degraded(
                "หน่วยความจำใช้งานสูง - High memory usage", data: data)),
            _ => Task.FromResult(HealthCheckResult.Healthy(
                "ระบบทำงานปกติ - System resources normal", data: data))
        };
    }
}
```

### Health Aggregator Service

```csharp
// src/ApiGateway/Services/HealthAggregatorService.cs
using System.Collections.Concurrent;
using System.Net.Http.Json;

namespace ShopThai.ApiGateway.Services;

public class ServiceHealthStatus
{
    public string ServiceName { get; set; } = string.Empty;
    public string Status { get; set; } = "Unknown";
    public double ResponseTimeMs { get; set; }
    public DateTime LastChecked { get; set; }
    public Dictionary<string, string> Details { get; set; } = [];
}

public class HealthAggregatorService : BackgroundService
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<HealthAggregatorService> _logger;
    private readonly ConcurrentDictionary<string, ServiceHealthStatus> _healthCache = new();

    private readonly Dictionary<string, string> _serviceEndpoints = new()
    {
        ["catalog-service"] = "http://catalog-service:8080/health",
        ["order-service"] = "http://order-service:8080/health",
        ["payment-service"] = "http://payment-service:8080/health",
        ["notification-service"] = "http://notification-service:8080/health"
    };

    public HealthAggregatorService(
        IHttpClientFactory httpClientFactory,
        ILogger<HealthAggregatorService> logger)
    {
        _httpClient = httpClientFactory.CreateClient("health-check");
        _httpClient.Timeout = TimeSpan.FromSeconds(5);
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await CheckAllServicesAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
    }

    private async Task CheckAllServicesAsync(CancellationToken cancellationToken)
    {
        var tasks = _serviceEndpoints.Select(async kvp =>
        {
            var (serviceName, endpoint) = kvp;
            var stopwatch = System.Diagnostics.Stopwatch.StartNew();

            try
            {
                var response = await _httpClient.GetAsync(endpoint, cancellationToken);
                stopwatch.Stop();

                var status = new ServiceHealthStatus
                {
                    ServiceName = serviceName,
                    Status = response.IsSuccessStatusCode ? "Healthy" : "Unhealthy",
                    ResponseTimeMs = stopwatch.ElapsedMilliseconds,
                    LastChecked = DateTime.UtcNow
                };

                if (response.IsSuccessStatusCode)
                {
                    var healthData = await response.Content.ReadFromJsonAsync<Dictionary<string, object>>(
                        cancellationToken: cancellationToken);
                    status.Details = healthData?.ToDictionary(k => k.Key, v => v.Value?.ToString() ?? "")
                        ?? [];
                }

                _healthCache[serviceName] = status;
            }
            catch (Exception ex)
            {
                stopwatch.Stop();
                _logger.LogWarning(ex, "ไม่สามารถตรวจสอบ Health ของ {ServiceName} ได้", serviceName);

                _healthCache[serviceName] = new ServiceHealthStatus
                {
                    ServiceName = serviceName,
                    Status = "Unreachable",
                    ResponseTimeMs = stopwatch.ElapsedMilliseconds,
                    LastChecked = DateTime.UtcNow,
                    Details = new() { ["error"] = ex.Message }
                };
            }
        });

        await Task.WhenAll(tasks);
    }

    public IReadOnlyDictionary<string, ServiceHealthStatus> GetAllHealth()
        => _healthCache.AsReadOnly();

    public ServiceHealthStatus? GetServiceHealth(string serviceName)
        => _healthCache.TryGetValue(serviceName, out var status) ? status : null;
}
```

---

## ขั้นตอนที่ 904: Distributed Tracing ข้ามทุก Services

### OpenTelemetry Setup สำหรับทุก Service

```csharp
// src/Shared/ShopThai.Shared.Infrastructure/Telemetry/TelemetryExtensions.cs
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using OpenTelemetry;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using OpenTelemetry.Metrics;
using System.Diagnostics;

namespace ShopThai.Shared.Infrastructure.Telemetry;

public static class TelemetryExtensions
{
    // ActivitySource สำหรับ Custom Tracing
    public static readonly ActivitySource ShopThaiActivitySource =
        new("ShopThai.Services", "1.0.0");

    /// <summary>
    /// ตั้งค่า OpenTelemetry สำหรับ ShopThai Services
    /// </summary>
    public static IServiceCollection AddShopThaiTelemetry(
        this IServiceCollection services,
        IConfiguration configuration,
        string serviceName)
    {
        var jaegerHost = configuration["Jaeger:Host"] ?? "jaeger";
        var jaegerPort = int.Parse(configuration["Jaeger:Port"] ?? "6831");
        var serviceVersion = configuration["Service:Version"] ?? "1.0.0";

        services.AddOpenTelemetry()
            .ConfigureResource(resource =>
            {
                resource.AddService(
                    serviceName: serviceName,
                    serviceVersion: serviceVersion,
                    serviceInstanceId: Environment.MachineName);
                resource.AddAttributes(new Dictionary<string, object>
                {
                    ["deployment.environment"] = configuration["ASPNETCORE_ENVIRONMENT"] ?? "Production",
                    ["host.name"] = Environment.MachineName
                });
            })
            .WithTracing(tracing =>
            {
                tracing
                    .AddSource(ShopThaiActivitySource.Name)
                    .AddAspNetCoreInstrumentation(options =>
                    {
                        options.RecordException = true;
                        options.EnrichWithHttpRequest = (activity, request) =>
                        {
                            activity.SetTag("http.client_ip",
                                request.HttpContext.Connection.RemoteIpAddress?.ToString());
                            if (request.Headers.TryGetValue("X-Correlation-ID", out var correlationId))
                            {
                                activity.SetTag("correlation_id", correlationId.ToString());
                            }
                        };
                    })
                    .AddHttpClientInstrumentation(options =>
                    {
                        options.RecordException = true;
                    })
                    .AddEntityFrameworkCoreInstrumentation(options =>
                    {
                        options.SetDbStatementForText = true;
                        options.SetDbStatementForStoredProcedure = true;
                    })
                    .AddRedisInstrumentation(options =>
                    {
                        options.SetVerboseDatabaseStatements = true;
                    })
                    .AddJaegerExporter(options =>
                    {
                        options.AgentHost = jaegerHost;
                        options.AgentPort = jaegerPort;
                    });
            })
            .WithMetrics(metrics =>
            {
                metrics
                    .AddAspNetCoreInstrumentation()
                    .AddHttpClientInstrumentation()
                    .AddRuntimeInstrumentation()
                    .AddPrometheusExporter();
            });

        return services;
    }
}

// Tracing Helper สำหรับ Business Operations
public static class TracingHelper
{
    private static readonly ActivitySource Source =
        TelemetryExtensions.ShopThaiActivitySource;

    /// <summary>
    /// สร้าง Activity สำหรับ Order Processing
    /// </summary>
    public static Activity? StartOrderProcessing(string orderId, string customerId)
    {
        var activity = Source.StartActivity("ProcessOrder",
            ActivityKind.Internal);

        activity?.SetTag("order.id", orderId);
        activity?.SetTag("customer.id", customerId);
        activity?.SetTag("operation", "process_order");

        return activity;
    }

    /// <summary>
    /// สร้าง Activity สำหรับ Payment Processing
    /// </summary>
    public static Activity? StartPaymentProcessing(string paymentId, decimal amount)
    {
        var activity = Source.StartActivity("ProcessPayment",
            ActivityKind.Internal);

        activity?.SetTag("payment.id", paymentId);
        activity?.SetTag("payment.amount", amount);
        activity?.SetTag("payment.currency", "THB");

        return activity;
    }

    /// <summary>
    /// บันทึก Error ลงใน Activity
    /// </summary>
    public static void RecordException(Activity? activity, Exception exception)
    {
        activity?.SetStatus(ActivityStatusCode.Error, exception.Message);
        activity?.RecordException(exception);
    }
}
```

### Correlation ID Middleware

```csharp
// src/Shared/ShopThai.Shared.Infrastructure/Middleware/CorrelationIdMiddleware.cs
namespace ShopThai.Shared.Infrastructure.Middleware;

public class CorrelationIdMiddleware
{
    private const string CorrelationIdHeader = "X-Correlation-ID";
    private readonly RequestDelegate _next;
    private readonly ILogger<CorrelationIdMiddleware> _logger;

    public CorrelationIdMiddleware(
        RequestDelegate next,
        ILogger<CorrelationIdMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // อ่าน Correlation ID จาก Header หรือสร้างใหม่
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        // เพิ่ม Correlation ID ใน Response Header
        context.Response.Headers[CorrelationIdHeader] = correlationId;

        // เพิ่ม Correlation ID ใน Activity Tags
        var activity = System.Diagnostics.Activity.Current;
        activity?.SetTag("correlation_id", correlationId);

        // เพิ่ม Correlation ID ใน Log Context
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["CorrelationId"] = correlationId
        }))
        {
            await _next(context);
        }
    }
}
```

---

## ขั้นตอนที่ 905: Complete Docker Compose Configuration

### docker-compose.yml - ทุก Services รวมกัน

```yaml
# docker/docker-compose.yml
version: '3.9'

# ============================================================
# Networks
# ============================================================
networks:
  shopthai-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

# ============================================================
# Volumes
# ============================================================
volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local
  rabbitmq-data:
    driver: local
  elasticsearch-data:
    driver: local
  seq-data:
    driver: local

# ============================================================
# Services
# ============================================================
services:

  # ----------------------------------------------------------
  # Infrastructure Services
  # ----------------------------------------------------------

  postgres:
    image: postgres:16-alpine
    container_name: shopthai-postgres
    environment:
      POSTGRES_USER: shopthai
      POSTGRES_PASSWORD: ShopThai@2024
      POSTGRES_DB: shopthai
      PGDATA: /data/postgres
    volumes:
      - postgres-data:/data/postgres
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
      - ./scripts/seed-data.sql:/docker-entrypoint-initdb.d/02-seed.sql:ro
    ports:
      - "5432:5432"
    networks:
      - shopthai-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U shopthai -d shopthai"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: shopthai-redis
    command: >
      redis-server
      --requirepass ShopThaiRedis@2024
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --save 900 1
      --save 300 10
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    networks:
      - shopthai-network
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "ShopThaiRedis@2024", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    restart: unless-stopped

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    container_name: shopthai-rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: shopthai
      RABBITMQ_DEFAULT_PASS: ShopThaiMQ@2024
      RABBITMQ_DEFAULT_VHOST: shopthai
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
      - ./config/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
    ports:
      - "5672:5672"
      - "15672:15672"
    networks:
      - shopthai-network
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 30s
      timeout: 30s
      retries: 3
    restart: unless-stopped

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    container_name: shopthai-elasticsearch
    environment:
      - node.name=shopthai-es
      - cluster.name=shopthai-cluster
      - discovery.type=single-node
      - ELASTIC_PASSWORD=ShopThaiES@2024
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - shopthai-network
    healthcheck:
      test: >
        curl -sf http://elastic:ShopThaiES@2024@localhost:9200/_cluster/health
        | grep -vq '"status":"red"'
      interval: 20s
      timeout: 10s
      retries: 5
      start_period: 60s
    restart: unless-stopped

  seq:
    image: datalust/seq:2024
    container_name: shopthai-seq
    environment:
      ACCEPT_EULA: "Y"
      SEQ_FIRSTRUN_ADMINPASSWORDHASH: ""
    volumes:
      - seq-data:/data
    ports:
      - "5341:80"
    networks:
      - shopthai-network
    restart: unless-stopped

  jaeger:
    image: jaegertracing/all-in-one:1.54
    container_name: shopthai-jaeger
    environment:
      COLLECTOR_ZIPKIN_HOST_PORT: ":9411"
      COLLECTOR_OTLP_ENABLED: "true"
    ports:
      - "6831:6831/udp"
      - "16686:16686"
      - "14268:14268"
      - "4317:4317"
      - "4318:4318"
    networks:
      - shopthai-network
    restart: unless-stopped

  # ----------------------------------------------------------
  # Application Services
  # ----------------------------------------------------------

  catalog-service:
    build:
      context: ../src
      dockerfile: Services/CatalogService/Dockerfile
    container_name: shopthai-catalog
    environment:
      ASPNETCORE_ENVIRONMENT: Production
      ConnectionStrings__DefaultConnection: >
        Host=postgres;Database=shopthai_catalog;
        Username=shopthai;Password=ShopThai@2024
      ConnectionStrings__Redis: "redis:6379,password=ShopThaiRedis@2024"
      RabbitMQ__Host: rabbitmq
      RabbitMQ__Username: shopthai
      RabbitMQ__Password: ShopThaiMQ@2024
      RabbitMQ__VirtualHost: shopthai
      Elasticsearch__Uri: "http://elasticsearch:9200"
      Elasticsearch__Username: elastic
      Elasticsearch__Password: ShopThaiES@2024
      Jaeger__Host: jaeger
      Jaeger__Port: "6831"
      SEQ__ServerUrl: "http://seq:80"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks:
      - shopthai-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  order-service:
    build:
      context: ../src
      dockerfile: Services/OrderService/Dockerfile
    container_name: shopthai-order
    environment:
      ASPNETCORE_ENVIRONMENT: Production
      ConnectionStrings__DefaultConnection: >
        Host=postgres;Database=shopthai_orders;
        Username=shopthai;Password=ShopThai@2024
      ConnectionStrings__Redis: "redis:6379,password=ShopThaiRedis@2024"
      RabbitMQ__Host: rabbitmq
      RabbitMQ__Username: shopthai
      RabbitMQ__Password: ShopThaiMQ@2024
      RabbitMQ__VirtualHost: shopthai
      Jaeger__Host: jaeger
      Jaeger__Port: "6831"
      SEQ__ServerUrl: "http://seq:80"
      Services__CatalogServiceUrl: "http://catalog-service:8080"
      Services__PaymentServiceUrl: "http://payment-service:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks:
      - shopthai-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  payment-service:
    build:
      context: ../src
      dockerfile: Services/PaymentService/Dockerfile
    container_name: shopthai-payment
    environment:
      ASPNETCORE_ENVIRONMENT: Production
      ConnectionStrings__DefaultConnection: >
        Host=postgres;Database=shopthai_payments;
        Username=shopthai;Password=ShopThai@2024
      ConnectionStrings__Redis: "redis:6379,password=ShopThaiRedis@2024"
      RabbitMQ__Host: rabbitmq
      RabbitMQ__Username: shopthai
      RabbitMQ__Password: ShopThaiMQ@2024
      RabbitMQ__VirtualHost: shopthai
      Jaeger__Host: jaeger
      Jaeger__Port: "6831"
      SEQ__ServerUrl: "http://seq:80"
      PaymentGateway__OmiseKey: "${OMISE_KEY}"
      PaymentGateway__OmiseSecretKey: "${OMISE_SECRET_KEY}"
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks:
      - shopthai-network
    restart: unless-stopped

  notification-service:
    build:
      context: ../src
      dockerfile: Services/NotificationService/Dockerfile
    container_name: shopthai-notification
    environment:
      ASPNETCORE_ENVIRONMENT: Production
      ConnectionStrings__DefaultConnection: >
        Host=postgres;Database=shopthai_notifications;
        Username=shopthai;Password=ShopThai@2024
      RabbitMQ__Host: rabbitmq
      RabbitMQ__Username: shopthai
      RabbitMQ__Password: ShopThaiMQ@2024
      RabbitMQ__VirtualHost: shopthai
      Smtp__Host: mailhog
      Smtp__Port: "1025"
      Firebase__ServiceAccountKey: "${FIREBASE_SERVICE_ACCOUNT}"
      Jaeger__Host: jaeger
      SEQ__ServerUrl: "http://seq:80"
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks:
      - shopthai-network
    restart: unless-stopped

  api-gateway:
    build:
      context: ../src
      dockerfile: ApiGateway/Dockerfile
    container_name: shopthai-gateway
    environment:
      ASPNETCORE_ENVIRONMENT: Production
      Jwt__Secret: "${JWT_SECRET}"
      Jwt__Issuer: ShopThai
      Jwt__Audience: ShopThai-Clients
      Jaeger__Host: jaeger
    ports:
      - "8080:8080"
      - "8443:8443"
    depends_on:
      catalog-service:
        condition: service_healthy
      order-service:
        condition: service_healthy
      payment-service:
        condition: service_started
      notification-service:
        condition: service_started
    networks:
      - shopthai-network
    restart: unless-stopped

  # ----------------------------------------------------------
  # Development Tools
  # ----------------------------------------------------------

  mailhog:
    image: mailhog/mailhog:v1.0.1
    container_name: shopthai-mailhog
    ports:
      - "1025:1025"
      - "8025:8025"
    networks:
      - shopthai-network
    profiles:
      - dev

  adminer:
    image: adminer:4
    container_name: shopthai-adminer
    ports:
      - "8081:8080"
    networks:
      - shopthai-network
    profiles:
      - dev
```

---

## ขั้นตอนที่ 906: Database Migration Strategy - กลยุทธ์การ Migrate Database

### Master Migration Script

```csharp
// src/Shared/ShopThai.Shared.Infrastructure/Migrations/MigrationRunner.cs
using DbUp;
using DbUp.Engine;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;

namespace ShopThai.Shared.Infrastructure.Migrations;

public class MigrationRunner
{
    private readonly ILogger<MigrationRunner> _logger;

    public MigrationRunner(ILogger<MigrationRunner> logger)
    {
        _logger = logger;
    }

    /// <summary>
    /// รัน Migration สำหรับ Service ที่กำหนด
    /// </summary>
    public bool RunMigrations(string connectionString, string serviceName)
    {
        _logger.LogInformation("เริ่มต้น Migration สำหรับ {ServiceName}", serviceName);

        // ตรวจสอบและสร้าง Database ถ้ายังไม่มี
        EnsureDatabase.For.PostgresqlDatabase(connectionString);

        var upgrader = DeployChanges.To
            .PostgresqlDatabase(connectionString)
            .WithScriptsEmbeddedInAssembly(
                typeof(MigrationRunner).Assembly,
                script => script.Contains($"Migrations.{serviceName}"))
            .WithTransaction()
            .LogToConsole()
            .WithVariablesDisabled()
            .Build();

        if (upgrader.IsUpgradeRequired())
        {
            _logger.LogInformation(
                "พบ Migration ที่ต้องทำสำหรับ {ServiceName}", serviceName);

            var result = upgrader.PerformUpgrade();

            if (!result.Successful)
            {
                _logger.LogError(result.Error,
                    "Migration ล้มเหลวสำหรับ {ServiceName}", serviceName);
                return false;
            }

            _logger.LogInformation(
                "Migration สำเร็จสำหรับ {ServiceName} - {Count} scripts",
                serviceName,
                result.Scripts.Count());
        }
        else
        {
            _logger.LogInformation(
                "ไม่มี Migration ที่ต้องทำสำหรับ {ServiceName}", serviceName);
        }

        return true;
    }
}
```

### SQL Migration Scripts

```sql
-- docker/scripts/init-db.sql
-- สร้าง Databases สำหรับทุก Service

-- Catalog Service Database
CREATE DATABASE shopthai_catalog
    WITH OWNER = shopthai
    ENCODING = 'UTF8'
    LC_COLLATE = 'th_TH.UTF-8'
    LC_CTYPE = 'th_TH.UTF-8';

-- Order Service Database
CREATE DATABASE shopthai_orders
    WITH OWNER = shopthai
    ENCODING = 'UTF8';

-- Payment Service Database
CREATE DATABASE shopthai_payments
    WITH OWNER = shopthai
    ENCODING = 'UTF8';

-- Notification Service Database
CREATE DATABASE shopthai_notifications
    WITH OWNER = shopthai
    ENCODING = 'UTF8';

-- User Service Database
CREATE DATABASE shopthai_users
    WITH OWNER = shopthai
    ENCODING = 'UTF8';

-- เชื่อมต่อและสร้าง Schema สำหรับ Catalog Service
\connect shopthai_catalog

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

CREATE TABLE IF NOT EXISTS categories (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(200) NOT NULL,
    name_th VARCHAR(200) NOT NULL,
    slug VARCHAR(200) UNIQUE NOT NULL,
    parent_id UUID REFERENCES categories(id),
    description TEXT,
    image_url VARCHAR(500),
    is_active BOOLEAN NOT NULL DEFAULT true,
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS products (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    sku VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(500) NOT NULL,
    name_th VARCHAR(500) NOT NULL,
    slug VARCHAR(500) UNIQUE NOT NULL,
    description TEXT,
    category_id UUID REFERENCES categories(id),
    price DECIMAL(18,2) NOT NULL,
    original_price DECIMAL(18,2),
    currency VARCHAR(3) NOT NULL DEFAULT 'THB',
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    reserved_quantity INTEGER NOT NULL DEFAULT 0,
    weight_grams INTEGER,
    is_active BOOLEAN NOT NULL DEFAULT true,
    is_featured BOOLEAN NOT NULL DEFAULT false,
    metadata JSONB,
    search_vector TSVECTOR,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- สร้าง Index สำหรับ Full-text Search
CREATE INDEX IF NOT EXISTS idx_products_search ON products USING GIN(search_vector);
CREATE INDEX IF NOT EXISTS idx_products_category ON products(category_id);
CREATE INDEX IF NOT EXISTS idx_products_active ON products(is_active) WHERE is_active = true;
CREATE INDEX IF NOT EXISTS idx_products_featured ON products(is_featured) WHERE is_featured = true;

-- Trigger สำหรับอัปเดต search_vector อัตโนมัติ
CREATE OR REPLACE FUNCTION update_product_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', COALESCE(NEW.name, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.description, '')), 'B') ||
        setweight(to_tsvector('simple', COALESCE(NEW.sku, '')), 'C');
    NEW.updated_at := NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_product_search_vector
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION update_product_search_vector();

-- เชื่อมต่อ Order Service Database
\connect shopthai_orders

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE IF NOT EXISTS orders (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    order_number VARCHAR(50) UNIQUE NOT NULL,
    customer_id UUID NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'Pending',
    total_amount DECIMAL(18,2) NOT NULL,
    subtotal DECIMAL(18,2) NOT NULL,
    tax_amount DECIMAL(18,2) NOT NULL DEFAULT 0,
    shipping_amount DECIMAL(18,2) NOT NULL DEFAULT 0,
    discount_amount DECIMAL(18,2) NOT NULL DEFAULT 0,
    currency VARCHAR(3) NOT NULL DEFAULT 'THB',
    shipping_address JSONB NOT NULL,
    payment_method VARCHAR(50),
    notes TEXT,
    correlation_id VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS order_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id UUID NOT NULL,
    product_name VARCHAR(500) NOT NULL,
    sku VARCHAR(100) NOT NULL,
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(18,2) NOT NULL,
    total_price DECIMAL(18,2) NOT NULL,
    metadata JSONB
);

CREATE TABLE IF NOT EXISTS order_status_history (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    from_status VARCHAR(50),
    to_status VARCHAR(50) NOT NULL,
    reason TEXT,
    created_by VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_orders_customer ON orders(customer_id);
CREATE INDEX IF NOT EXISTS idx_orders_status ON orders(status);
CREATE INDEX IF NOT EXISTS idx_orders_created ON orders(created_at DESC);
CREATE INDEX IF NOT EXISTS idx_order_items_order ON order_items(order_id);
```

---

## ขั้นตอนที่ 907: Seed Data สำหรับการทดสอบ

### Seed Data SQL

```sql
-- docker/scripts/seed-data.sql
-- ข้อมูลตัวอย่างสำหรับการทดสอบ

\connect shopthai_catalog;

-- หมวดหมู่สินค้า
INSERT INTO categories (id, name, name_th, slug, description, is_active, sort_order) VALUES
    ('11111111-1111-1111-1111-111111111111', 'Electronics', 'อิเล็กทรอนิกส์', 'electronics',
     'สินค้าอิเล็กทรอนิกส์ทุกประเภท', true, 1),
    ('22222222-2222-2222-2222-222222222222', 'Fashion', 'แฟชั่น', 'fashion',
     'เสื้อผ้าและเครื่องประดับ', true, 2),
    ('33333333-3333-3333-3333-333333333333', 'Food & Beverage', 'อาหารและเครื่องดื่ม', 'food-beverage',
     'อาหารและเครื่องดื่มหลากหลายประเภท', true, 3),
    ('44444444-4444-4444-4444-444444444444', 'Books', 'หนังสือ', 'books',
     'หนังสือและสื่อการเรียนรู้', true, 4)
ON CONFLICT (slug) DO NOTHING;

-- สินค้าตัวอย่าง
INSERT INTO products (
    id, sku, name, name_th, slug, description, category_id,
    price, original_price, stock_quantity, is_active, is_featured
) VALUES
    ('aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa',
     'IPHONE-15-PRO-128', 'iPhone 15 Pro 128GB',
     'ไอโฟน 15 โปร 128 กิกะไบต์',
     'iphone-15-pro-128',
     'สมาร์ทโฟนชั้นนำจาก Apple พร้อม Dynamic Island และ USB-C',
     '11111111-1111-1111-1111-111111111111',
     42900.00, 44900.00, 50, true, true),

    ('bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb',
     'SAMSUNG-S24-256', 'Samsung Galaxy S24 256GB',
     'ซัมซุง กาแล็กซี เอส24 256 กิกะไบต์',
     'samsung-galaxy-s24-256',
     'สมาร์ทโฟน Android ระดับ Flagship พร้อม AI Features',
     '11111111-1111-1111-1111-111111111111',
     32900.00, 34900.00, 75, true, true),

    ('cccccccc-cccc-cccc-cccc-cccccccccccc',
     'MACBOOK-AIR-M3', 'MacBook Air M3 13"',
     'แมคบุ๊ก แอร์ เอ็ม3 13 นิ้ว',
     'macbook-air-m3-13',
     'แล็ปท็อปน้ำหนักเบาพร้อม Apple M3 Chip ประสิทธิภาพสูง',
     '11111111-1111-1111-1111-111111111111',
     52900.00, 54900.00, 30, true, false),

    ('dddddddd-dddd-dddd-dddd-dddddddddddd',
     'THAI-SILK-DRESS-M', 'Thai Silk Dress - Medium',
     'ชุดผ้าไหมไทย ไซส์ M',
     'thai-silk-dress-medium',
     'ชุดผ้าไหมไทยแท้ ทอมือ สวมใส่สบาย เหมาะสำหรับทุกโอกาส',
     '22222222-2222-2222-2222-222222222222',
     2500.00, 3000.00, 100, true, true)
ON CONFLICT (sku) DO NOTHING;
```

### C# Seeder Class

```csharp
// src/Shared/ShopThai.Shared.Infrastructure/Seeding/DataSeeder.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;

namespace ShopThai.Shared.Infrastructure.Seeding;

public abstract class DataSeeder<TContext> where TContext : DbContext
{
    protected readonly TContext Context;
    protected readonly ILogger Logger;

    protected DataSeeder(TContext context, ILogger logger)
    {
        Context = context;
        Logger = logger;
    }

    public async Task SeedAsync(CancellationToken cancellationToken = default)
    {
        Logger.LogInformation("เริ่ม Seed ข้อมูลสำหรับ {Context}", typeof(TContext).Name);

        try
        {
            await SeedInternalAsync(cancellationToken);
            await Context.SaveChangesAsync(cancellationToken);
            Logger.LogInformation("Seed ข้อมูลสำเร็จ");
        }
        catch (Exception ex)
        {
            Logger.LogError(ex, "เกิดข้อผิดพลาดขณะ Seed ข้อมูล");
            throw;
        }
    }

    protected abstract Task SeedInternalAsync(CancellationToken cancellationToken);

    protected bool IsProduction =>
        Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") == "Production";
}

// ตัวอย่าง Catalog Data Seeder
public class CatalogDataSeeder : DataSeeder<CatalogDbContext>
{
    public CatalogDataSeeder(CatalogDbContext context, ILogger<CatalogDataSeeder> logger)
        : base(context, logger) { }

    protected override async Task SeedInternalAsync(CancellationToken cancellationToken)
    {
        // ใช้ข้อมูลจาก SQL Script หรือสร้างใหม่
        if (!IsProduction && !await Context.Products.AnyAsync(cancellationToken))
        {
            Logger.LogInformation("เพิ่ม Sample Products สำหรับ Development");

            var electronics = new Category
            {
                Id = Guid.Parse("11111111-1111-1111-1111-111111111111"),
                Name = "Electronics",
                NameTh = "อิเล็กทรอนิกส์",
                Slug = "electronics",
                IsActive = true,
                SortOrder = 1
            };

            await Context.Categories.AddAsync(electronics, cancellationToken);

            var products = GenerateSampleProducts(electronics.Id);
            await Context.Products.AddRangeAsync(products, cancellationToken);
        }
    }

    private static List<Product> GenerateSampleProducts(Guid categoryId)
    {
        return
        [
            new Product
            {
                Id = Guid.NewGuid(),
                Sku = $"TEST-PRODUCT-{DateTime.Now.Ticks}",
                Name = "Test iPhone 15",
                NameTh = "ไอโฟน 15 ทดสอบ",
                Slug = $"test-iphone-{DateTime.Now.Ticks}",
                Price = 42900,
                StockQuantity = 100,
                CategoryId = categoryId,
                IsActive = true
            }
        ];
    }
}

// Placeholder classes to make code compile (actual implementations in respective services)
public class CatalogDbContext : DbContext
{
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Product> Products => Set<Product>();
}

public class Category
{
    public Guid Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string NameTh { get; set; } = string.Empty;
    public string Slug { get; set; } = string.Empty;
    public bool IsActive { get; set; }
    public int SortOrder { get; set; }
}

public class Product
{
    public Guid Id { get; set; }
    public string Sku { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
    public string NameTh { get; set; } = string.Empty;
    public string Slug { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public Guid CategoryId { get; set; }
    public bool IsActive { get; set; }
}
```

---

## ขั้นตอนที่ 908: End-to-End Test - ทดสอบ Flow ทั้งหมด

### E2E Test: Place Order → Payment → Inventory → Shipping → Notification

```csharp
// tests/E2ETests/OrderFlowE2ETests.cs
using System.Net.Http.Json;
using Microsoft.AspNetCore.Mvc.Testing;
using Xunit;
using FluentAssertions;

namespace ShopThai.Tests.E2E;

[Collection("E2E")]
public class OrderFlowE2ETests : IAsyncLifetime
{
    private readonly HttpClient _client;
    private readonly string _baseUrl;
    private string? _authToken;
    private Guid _testCustomerId;

    public OrderFlowE2ETests()
    {
        _baseUrl = Environment.GetEnvironmentVariable("API_GATEWAY_URL")
            ?? "http://localhost:8080";
        _client = new HttpClient { BaseAddress = new Uri(_baseUrl) };
        _testCustomerId = Guid.NewGuid();
    }

    public async Task InitializeAsync()
    {
        // ขอ Token สำหรับ Test User
        _authToken = await GetTestTokenAsync();
        _client.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", _authToken);
    }

    public Task DisposeAsync()
    {
        _client.Dispose();
        return Task.CompletedTask;
    }

    /// <summary>
    /// ทดสอบ Flow หลัก: สั่งสินค้า → ชำระเงิน → จอง Inventory → จัดส่ง → แจ้งเตือน
    /// </summary>
    [Fact]
    [Trait("Category", "E2E")]
    public async Task CompleteOrderFlow_ShouldSucceed()
    {
        // ============================================================
        // Step 1: ดึงข้อมูลสินค้า
        // ============================================================
        var productResponse = await _client.GetAsync("/api/catalog/products?page=1&limit=5");
        productResponse.IsSuccessStatusCode.Should().BeTrue(
            "ต้องดึงข้อมูลสินค้าได้");

        var products = await productResponse.Content.ReadFromJsonAsync<ProductListResponse>();
        products.Should().NotBeNull();
        products!.Items.Should().NotBeEmpty("ต้องมีสินค้าในระบบ");

        var selectedProduct = products.Items.First();

        // ============================================================
        // Step 2: สร้าง Order
        // ============================================================
        var createOrderRequest = new CreateOrderRequest
        {
            CustomerId = _testCustomerId,
            Items =
            [
                new OrderItemRequest
                {
                    ProductId = selectedProduct.Id,
                    Quantity = 2
                }
            ],
            ShippingAddress = new AddressDto
            {
                FullName = "สมชาย ใจดี",
                Phone = "0812345678",
                AddressLine1 = "123 ถนนสุขุมวิท",
                District = "คลองตัน",
                Province = "กรุงเทพมหานคร",
                PostalCode = "10110"
            },
            PaymentMethod = "credit_card"
        };

        var createOrderResponse = await _client.PostAsJsonAsync(
            "/api/orders", createOrderRequest);
        createOrderResponse.IsSuccessStatusCode.Should().BeTrue(
            "ต้องสร้าง Order ได้สำเร็จ");

        var createdOrder = await createOrderResponse.Content
            .ReadFromJsonAsync<OrderResponse>();
        createdOrder.Should().NotBeNull();
        createdOrder!.Status.Should().Be("Pending");

        var orderId = createdOrder.Id;

        // ============================================================
        // Step 3: ชำระเงิน
        // ============================================================
        var paymentRequest = new ProcessPaymentRequest
        {
            OrderId = orderId,
            Amount = createdOrder.TotalAmount,
            Currency = "THB",
            PaymentMethod = "credit_card",
            CardToken = "test_token_visa_4111" // Test token
        };

        var paymentResponse = await _client.PostAsJsonAsync(
            "/api/payments/process", paymentRequest);
        paymentResponse.IsSuccessStatusCode.Should().BeTrue(
            "ต้องชำระเงินได้สำเร็จ");

        var payment = await paymentResponse.Content.ReadFromJsonAsync<PaymentResponse>();
        payment.Should().NotBeNull();
        payment!.Status.Should().BeOneOf("Authorized", "Captured");

        // ============================================================
        // Step 4: รอให้ Event Processing เสร็จ (Inventory + Shipping)
        // ============================================================
        await WaitForOrderStatusAsync(orderId, "Processing", maxWaitSeconds: 30);

        // ============================================================
        // Step 5: ตรวจสอบสถานะ Order
        // ============================================================
        var orderResponse = await _client.GetAsync($"/api/orders/{orderId}");
        orderResponse.IsSuccessStatusCode.Should().BeTrue();

        var finalOrder = await orderResponse.Content.ReadFromJsonAsync<OrderResponse>();
        finalOrder.Should().NotBeNull();
        finalOrder!.Status.Should().BeOneOf("Processing", "Shipped", "Confirmed");

        // ============================================================
        // Step 6: ตรวจสอบว่ามีการแจ้งเตือน
        // ============================================================
        await Task.Delay(2000); // รอให้ Notification ส่ง

        var notifResponse = await _client.GetAsync(
            $"/api/notifications?customerId={_testCustomerId}&orderId={orderId}");
        notifResponse.IsSuccessStatusCode.Should().BeTrue();

        var notifications = await notifResponse.Content
            .ReadFromJsonAsync<NotificationListResponse>();
        notifications.Should().NotBeNull();
        notifications!.Items.Should().NotBeEmpty(
            "ต้องมีการแจ้งเตือนหลังจากสั่งสินค้าสำเร็จ");
    }

    /// <summary>
    /// ทดสอบการยกเลิก Order เมื่อ Payment ล้มเหลว
    /// </summary>
    [Fact]
    [Trait("Category", "E2E")]
    public async Task OrderFlow_WhenPaymentFails_ShouldCancelOrder()
    {
        // สร้าง Order
        var createOrderRequest = CreateTestOrderRequest();
        var createOrderResponse = await _client.PostAsJsonAsync(
            "/api/orders", createOrderRequest);
        var order = await createOrderResponse.Content.ReadFromJsonAsync<OrderResponse>();

        // ลอง Payment ด้วย Card ที่จะ Fail
        var paymentRequest = new ProcessPaymentRequest
        {
            OrderId = order!.Id,
            Amount = order.TotalAmount,
            Currency = "THB",
            PaymentMethod = "credit_card",
            CardToken = "test_token_declined" // Token ที่จะถูก Decline
        };

        var paymentResponse = await _client.PostAsJsonAsync(
            "/api/payments/process", paymentRequest);

        // Payment อาจคืน 400 หรือ 200 พร้อม status=Failed
        var payment = await paymentResponse.Content.ReadFromJsonAsync<PaymentResponse>();

        // รอให้ Order ถูกยกเลิก
        await WaitForOrderStatusAsync(order.Id, "Cancelled", maxWaitSeconds: 20);

        var cancelledOrder = await _client.GetAsync($"/api/orders/{order.Id}");
        var finalOrder = await cancelledOrder.Content.ReadFromJsonAsync<OrderResponse>();

        finalOrder!.Status.Should().Be("Cancelled",
            "Order ต้องถูกยกเลิกเมื่อ Payment ล้มเหลว");
    }

    private async Task WaitForOrderStatusAsync(
        Guid orderId,
        string expectedStatus,
        int maxWaitSeconds = 30)
    {
        var deadline = DateTime.UtcNow.AddSeconds(maxWaitSeconds);

        while (DateTime.UtcNow < deadline)
        {
            var response = await _client.GetAsync($"/api/orders/{orderId}");
            if (response.IsSuccessStatusCode)
            {
                var order = await response.Content.ReadFromJsonAsync<OrderResponse>();
                if (order?.Status == expectedStatus)
                    return;
            }

            await Task.Delay(1000);
        }

        throw new TimeoutException(
            $"Order {orderId} ยังไม่เปลี่ยนสถานะเป็น {expectedStatus} " +
            $"ภายใน {maxWaitSeconds} วินาที");
    }

    private async Task<string> GetTestTokenAsync()
    {
        var loginRequest = new
        {
            email = "test@shopthai.com",
            password = "TestPass@2024"
        };

        var response = await _client.PostAsJsonAsync("/api/auth/login", loginRequest);
        var result = await response.Content.ReadFromJsonAsync<LoginResponse>();
        return result?.AccessToken ?? throw new Exception("ไม่สามารถขอ Token ได้");
    }

    private CreateOrderRequest CreateTestOrderRequest() => new()
    {
        CustomerId = _testCustomerId,
        Items =
        [
            new OrderItemRequest
            {
                ProductId = Guid.Parse("aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"),
                Quantity = 1
            }
        ],
        ShippingAddress = new AddressDto
        {
            FullName = "ทดสอบ ระบบ",
            Phone = "0899999999",
            AddressLine1 = "999 ถนนทดสอบ",
            District = "ทดสอบ",
            Province = "กรุงเทพมหานคร",
            PostalCode = "10100"
        },
        PaymentMethod = "credit_card"
    };
}

// DTOs สำหรับ E2E Tests
public record CreateOrderRequest
{
    public Guid CustomerId { get; init; }
    public List<OrderItemRequest> Items { get; init; } = [];
    public AddressDto ShippingAddress { get; init; } = new();
    public string PaymentMethod { get; init; } = string.Empty;
}

public record OrderItemRequest
{
    public Guid ProductId { get; init; }
    public int Quantity { get; init; }
}

public record AddressDto
{
    public string FullName { get; init; } = string.Empty;
    public string Phone { get; init; } = string.Empty;
    public string AddressLine1 { get; init; } = string.Empty;
    public string District { get; init; } = string.Empty;
    public string Province { get; init; } = string.Empty;
    public string PostalCode { get; init; } = string.Empty;
}

public record ProcessPaymentRequest
{
    public Guid OrderId { get; init; }
    public decimal Amount { get; init; }
    public string Currency { get; init; } = "THB";
    public string PaymentMethod { get; init; } = string.Empty;
    public string CardToken { get; init; } = string.Empty;
}

public record OrderResponse
{
    public Guid Id { get; init; }
    public string OrderNumber { get; init; } = string.Empty;
    public string Status { get; init; } = string.Empty;
    public decimal TotalAmount { get; init; }
    public DateTime CreatedAt { get; init; }
}

public record PaymentResponse
{
    public Guid Id { get; init; }
    public string Status { get; init; } = string.Empty;
    public string TransactionId { get; init; } = string.Empty;
}

public record ProductListResponse
{
    public List<ProductDto> Items { get; init; } = [];
    public int Total { get; init; }
}

public record ProductDto
{
    public Guid Id { get; init; }
    public string Name { get; init; } = string.Empty;
    public decimal Price { get; init; }
}

public record NotificationListResponse
{
    public List<NotificationDto> Items { get; init; } = [];
}

public record NotificationDto
{
    public Guid Id { get; init; }
    public string Type { get; init; } = string.Empty;
    public string Message { get; init; } = string.Empty;
    public DateTime CreatedAt { get; init; }
}

public record LoginResponse
{
    public string AccessToken { get; init; } = string.Empty;
    public string RefreshToken { get; init; } = string.Empty;
}
```

---

## ขั้นตอนที่ 909: Feature Flags ด้วย Microsoft.FeatureManagement

### ติดตั้งและตั้งค่า Feature Management

```xml
<!-- เพิ่มใน .csproj ของทุก Service -->
<PackageReference Include="Microsoft.FeatureManagement.AspNetCore" Version="3.3.0" />
```

### Feature Flags Configuration

```json
// appsettings.json
{
  "FeatureManagement": {
    "NewCheckoutFlow": true,
    "AIProductRecommendations": false,
    "AdvancedSearch": {
      "EnabledFor": [
        {
          "Name": "Percentage",
          "Parameters": {
            "Value": 25
          }
        }
      ]
    },
    "BetaFeatures": {
      "EnabledFor": [
        {
          "Name": "CustomFilter",
          "Parameters": {
            "AllowedUsers": ["beta@shopthai.com", "admin@shopthai.com"]
          }
        }
      ]
    },
    "FlashSale": {
      "EnabledFor": [
        {
          "Name": "TimeWindow",
          "Parameters": {
            "Start": "2024-12-25T00:00:00",
            "End": "2024-12-26T23:59:59"
          }
        }
      ]
    }
  }
}
```

### Feature Flags Service Implementation

```csharp
// src/Shared/ShopThai.Shared.Infrastructure/Features/FeatureNames.cs
namespace ShopThai.Shared.Infrastructure.Features;

/// <summary>
/// รายชื่อ Feature Flags ทั้งหมดของ ShopThai
/// </summary>
public static class FeatureNames
{
    // ฟีเจอร์หน้าแรก
    public const string NewCheckoutFlow = nameof(NewCheckoutFlow);
    public const string AIProductRecommendations = nameof(AIProductRecommendations);
    public const string AdvancedSearch = nameof(AdvancedSearch);
    public const string BetaFeatures = nameof(BetaFeatures);
    public const string FlashSale = nameof(FlashSale);

    // ฟีเจอร์ Payment
    public const string PromptPayPayment = nameof(PromptPayPayment);
    public const string CryptoPayment = nameof(CryptoPayment);
    public const string InstallmentPayment = nameof(InstallmentPayment);

    // ฟีเจอร์ Notification
    public const string PushNotifications = nameof(PushNotifications);
    public const string LineNotification = nameof(LineNotification);
    public const string SmsNotification = nameof(SmsNotification);

    // ฟีเจอร์ Maintenance
    public const string MaintenanceMode = nameof(MaintenanceMode);
    public const string ReadOnlyMode = nameof(ReadOnlyMode);
}
```

```csharp
// src/Shared/ShopThai.Shared.Infrastructure/Features/FeatureFlagService.cs
using Microsoft.FeatureManagement;
using Microsoft.Extensions.Logging;

namespace ShopThai.Shared.Infrastructure.Features;

public interface IFeatureFlagService
{
    Task<bool> IsEnabledAsync(string featureName);
    Task<bool> IsEnabledForUserAsync(string featureName, string userId);
    Task<T> GetFeatureValueAsync<T>(string featureName, T defaultValue);
}

public class FeatureFlagService : IFeatureFlagService
{
    private readonly IFeatureManager _featureManager;
    private readonly ILogger<FeatureFlagService> _logger;

    public FeatureFlagService(
        IFeatureManager featureManager,
        ILogger<FeatureFlagService> logger)
    {
        _featureManager = featureManager;
        _logger = logger;
    }

    public async Task<bool> IsEnabledAsync(string featureName)
    {
        try
        {
            var isEnabled = await _featureManager.IsEnabledAsync(featureName);
            _logger.LogDebug("Feature {FeatureName}: {Status}",
                featureName, isEnabled ? "Enabled" : "Disabled");
            return isEnabled;
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "ไม่สามารถตรวจสอบ Feature Flag {FeatureName} ได้ - ใช้ค่า Default (false)",
                featureName);
            return false; // Fail-safe: ถ้าตรวจสอบไม่ได้ ให้ปิด Feature
        }
    }

    public async Task<bool> IsEnabledForUserAsync(string featureName, string userId)
    {
        try
        {
            return await _featureManager.IsEnabledAsync(featureName,
                new UserContext(userId));
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex,
                "ไม่สามารถตรวจสอบ Feature Flag {FeatureName} สำหรับผู้ใช้ {UserId}",
                featureName, userId);
            return false;
        }
    }

    public async Task<T> GetFeatureValueAsync<T>(string featureName, T defaultValue)
    {
        if (!await IsEnabledAsync(featureName))
            return defaultValue;

        return defaultValue; // ในระบบจริงอาจดึง Configuration เพิ่มเติม
    }
}

public class UserContext : ITargetingContext
{
    public string UserId { get; }
    public IEnumerable<string> Groups { get; } = [];

    public UserContext(string userId) => UserId = userId;
}

public interface ITargetingContext
{
    string UserId { get; }
    IEnumerable<string> Groups { get; }
}
```

```csharp
// ตัวอย่างการใช้งาน Feature Flags ใน Controller
// src/Services/OrderService/Controllers/OrderController.cs
using Microsoft.AspNetCore.Mvc;
using Microsoft.FeatureManagement;
using Microsoft.FeatureManagement.Mvc;
using ShopThai.Shared.Infrastructure.Features;

namespace ShopThai.OrderService.Controllers;

[ApiController]
[Route("api/orders")]
public class OrderController : ControllerBase
{
    private readonly IFeatureFlagService _featureFlags;
    private readonly ILogger<OrderController> _logger;

    public OrderController(
        IFeatureFlagService featureFlags,
        ILogger<OrderController> logger)
    {
        _featureFlags = featureFlags;
        _logger = logger;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderCommand command)
    {
        // ตรวจสอบ Maintenance Mode
        if (await _featureFlags.IsEnabledAsync(FeatureNames.MaintenanceMode))
        {
            return StatusCode(503, new
            {
                error = "ขณะนี้ระบบอยู่ในโหมดปรับปรุง กรุณาลองใหม่อีกครั้งในภายหลัง",
                retryAfter = 3600
            });
        }

        // ตรวจสอบ Read-Only Mode
        if (await _featureFlags.IsEnabledAsync(FeatureNames.ReadOnlyMode))
        {
            return StatusCode(503, new
            {
                error = "ระบบอยู่ในโหมด Read-Only ชั่วคราว ไม่สามารถสร้าง Order ได้"
            });
        }

        // ใช้ New Checkout Flow ถ้าเปิดใช้งาน
        if (await _featureFlags.IsEnabledAsync(FeatureNames.NewCheckoutFlow))
        {
            _logger.LogInformation("ใช้ New Checkout Flow สำหรับ Order");
            return await ProcessWithNewCheckoutFlowAsync(command);
        }

        return await ProcessWithLegacyFlowAsync(command);
    }

    [HttpGet("{id}/recommendations")]
    [FeatureGate(FeatureNames.AIProductRecommendations)]
    public async Task<IActionResult> GetRecommendations(Guid id)
    {
        // Endpoint นี้จะใช้งานได้เฉพาะเมื่อ Feature Flag เปิดอยู่
        return Ok(new { message = "AI Recommendations feature is enabled", orderId = id });
    }

    private async Task<IActionResult> ProcessWithNewCheckoutFlowAsync(
        CreateOrderCommand command)
    {
        // Logic สำหรับ New Checkout Flow
        await Task.Delay(10); // Simulate processing
        return Ok(new { message = "Order created with new flow", orderId = Guid.NewGuid() });
    }

    private async Task<IActionResult> ProcessWithLegacyFlowAsync(
        CreateOrderCommand command)
    {
        // Logic สำหรับ Legacy Flow
        await Task.Delay(10); // Simulate processing
        return Ok(new { message = "Order created with legacy flow", orderId = Guid.NewGuid() });
    }
}

public record CreateOrderCommand
{
    public Guid CustomerId { get; init; }
    public List<object> Items { get; init; } = [];
}
```

### ตั้งค่า Feature Management ใน Program.cs

```csharp
// เพิ่มใน Program.cs ของทุก Service
using Microsoft.FeatureManagement;

// ลงทะเบียน Feature Management
builder.Services.AddFeatureManagement()
    .AddFeatureFilter<Microsoft.FeatureManagement.FeatureFilters.PercentageFilter>()
    .AddFeatureFilter<Microsoft.FeatureManagement.FeatureFilters.TimeWindowFilter>()
    .AddFeatureFilter<Microsoft.FeatureManagement.FeatureFilters.TargetingFilter>();

// ลงทะเบียน Custom Services
builder.Services.AddScoped<IFeatureFlagService, FeatureFlagService>();
```

---

## ขั้นตอนที่ 910: Graceful Shutdown Handling - การปิดระบบอย่างสง่างาม

### Graceful Shutdown สำหรับ Background Services

```csharp
// src/Shared/ShopThai.Shared.Infrastructure/Hosting/GracefulShutdownExtensions.cs
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace ShopThai.Shared.Infrastructure.Hosting;

public static class GracefulShutdownExtensions
{
    /// <summary>
    /// ตั้งค่า Graceful Shutdown สำหรับ ShopThai Services
    /// </summary>
    public static IServiceCollection AddShopThaiGracefulShutdown(
        this IServiceCollection services,
        int shutdownTimeoutSeconds = 30)
    {
        services.Configure<HostOptions>(options =>
        {
            // เวลา Timeout สำหรับการปิดระบบ
            options.ShutdownTimeout = TimeSpan.FromSeconds(shutdownTimeoutSeconds);

            // จัดการ Exception ใน Background Services
            options.BackgroundServiceExceptionBehavior =
                BackgroundServiceExceptionBehavior.StopHost;
        });

        // ลงทะเบียน Shutdown Handler
        services.AddSingleton<IShutdownHandler, DefaultShutdownHandler>();

        return services;
    }

    /// <summary>
    /// ตั้งค่า Middleware สำหรับ Graceful Shutdown
    /// </summary>
    public static WebApplication UseShopThaiGracefulShutdown(this WebApplication app)
    {
        var lifetime = app.Services.GetRequiredService<IHostApplicationLifetime>();
        var logger = app.Services.GetRequiredService<ILogger<WebApplication>>();
        var shutdownHandler = app.Services.GetRequiredService<IShutdownHandler>();

        // ตรวจสอบเมื่อเริ่มต้นระบบ
        lifetime.ApplicationStarted.Register(() =>
        {
            logger.LogInformation(
                "ShopThai Service เริ่มทำงานแล้ว - {Time}",
                DateTime.UtcNow.ToString("yyyy-MM-dd HH:mm:ss UTC"));
        });

        // ตรวจสอบเมื่อเริ่มกระบวนการปิดระบบ
        lifetime.ApplicationStopping.Register(async () =>
        {
            logger.LogWarning(
                "กำลังปิด ShopThai Service - เริ่มกระบวนการ Graceful Shutdown...");

            try
            {
                // รอให้ Requests ที่กำลังประมวลผลเสร็จสิ้น
                await shutdownHandler.HandleShutdownAsync();
                logger.LogInformation("Graceful Shutdown สำเร็จ");
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "เกิดข้อผิดพลาดขณะ Graceful Shutdown");
            }
        });

        // ตรวจสอบเมื่อปิดระบบเสร็จสิ้น
        lifetime.ApplicationStopped.Register(() =>
        {
            logger.LogInformation(
                "ShopThai Service ปิดทำงานแล้ว - {Time}",
                DateTime.UtcNow.ToString("yyyy-MM-dd HH:mm:ss UTC"));
        });

        return app;
    }
}

public interface IShutdownHandler
{
    Task HandleShutdownAsync(CancellationToken cancellationToken = default);
}

public class DefaultShutdownHandler : IShutdownHandler
{
    private readonly ILogger<DefaultShutdownHandler> _logger;
    private readonly IEnumerable<IShutdownTask> _shutdownTasks;

    public DefaultShutdownHandler(
        ILogger<DefaultShutdownHandler> logger,
        IEnumerable<IShutdownTask> shutdownTasks)
    {
        _logger = logger;
        _shutdownTasks = shutdownTasks;
    }

    public async Task HandleShutdownAsync(CancellationToken cancellationToken = default)
    {
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
        cts.CancelAfter(TimeSpan.FromSeconds(25)); // Timeout 25 วินาที

        var tasks = _shutdownTasks
            .OrderBy(t => t.Priority)
            .Select(task => ExecuteShutdownTaskAsync(task, cts.Token))
            .ToList();

        await Task.WhenAll(tasks);
    }

    private async Task ExecuteShutdownTaskAsync(
        IShutdownTask task,
        CancellationToken cancellationToken)
    {
        try
        {
            _logger.LogInformation("กำลังรัน Shutdown Task: {TaskName}", task.Name);
            await task.ExecuteAsync(cancellationToken);
            _logger.LogInformation("Shutdown Task สำเร็จ: {TaskName}", task.Name);
        }
        catch (OperationCanceledException)
        {
            _logger.LogWarning("Shutdown Task ถูกยกเลิก (Timeout): {TaskName}", task.Name);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Shutdown Task ล้มเหลว: {TaskName}", task.Name);
        }
    }
}

public interface IShutdownTask
{
    string Name { get; }
    int Priority { get; } // ตัวเลขน้อย = รันก่อน
    Task ExecuteAsync(CancellationToken cancellationToken = default);
}
```

### Shutdown Tasks สำหรับ RabbitMQ และ Database

```csharp
// src/Shared/ShopThai.Shared.Infrastructure/Hosting/ShutdownTasks.cs
using Microsoft.Extensions.Logging;
using RabbitMQ.Client;
using Microsoft.EntityFrameworkCore;

namespace ShopThai.Shared.Infrastructure.Hosting;

/// <summary>
/// Shutdown Task สำหรับ RabbitMQ - รอให้ Messages ที่กำลังประมวลผลเสร็จสิ้น
/// </summary>
public class RabbitMQShutdownTask : IShutdownTask
{
    private readonly IConnection _connection;
    private readonly ILogger<RabbitMQShutdownTask> _logger;

    public string Name => "RabbitMQ Connection Cleanup";
    public int Priority => 1; // รันก่อน

    public RabbitMQShutdownTask(
        IConnection connection,
        ILogger<RabbitMQShutdownTask> logger)
    {
        _connection = connection;
        _logger = logger;
    }

    public async Task ExecuteAsync(CancellationToken cancellationToken = default)
    {
        // รอให้ Consumer Channels ปิดก่อน
        await Task.Delay(2000, cancellationToken);

        if (_connection.IsOpen)
        {
            _logger.LogInformation("กำลังปิด RabbitMQ Connection...");
            _connection.Close(TimeSpan.FromSeconds(5));
        }
    }
}

/// <summary>
/// Shutdown Task สำหรับ Database - ปิด Connection Pool อย่างถูกต้อง
/// </summary>
public class DatabaseShutdownTask : IShutdownTask
{
    private readonly DbContext _dbContext;
    private readonly ILogger<DatabaseShutdownTask> _logger;

    public string Name => "Database Connection Pool Cleanup";
    public int Priority => 2; // รันหลัง RabbitMQ

    public DatabaseShutdownTask(
        DbContext dbContext,
        ILogger<DatabaseShutdownTask> logger)
    {
        _dbContext = dbContext;
        _logger = logger;
    }

    public async Task ExecuteAsync(CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("กำลังปิด Database Connection...");
        await _dbContext.DisposeAsync();
    }
}

/// <summary>
/// Shutdown Task สำหรับ Cache Flush
/// </summary>
public class CacheShutdownTask : IShutdownTask
{
    private readonly ILogger<CacheShutdownTask> _logger;

    public string Name => "Cache Flush and Cleanup";
    public int Priority => 3;

    public CacheShutdownTask(ILogger<CacheShutdownTask> logger)
    {
        _logger = logger;
    }

    public async Task ExecuteAsync(CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("กำลัง Flush Cache...");
        // Flush in-memory caches ถ้ามี
        await Task.Delay(500, cancellationToken);
    }
}
```

### Program.cs Template สมบูรณ์สำหรับทุก Service

```csharp
// Template สำหรับ Program.cs ของทุก ShopThai Service
using Microsoft.FeatureManagement;
using ShopThai.Shared.Infrastructure.Features;
using ShopThai.Shared.Infrastructure.Health;
using ShopThai.Shared.Infrastructure.Hosting;
using ShopThai.Shared.Infrastructure.Middleware;
using ShopThai.Shared.Infrastructure.Telemetry;

var builder = WebApplication.CreateBuilder(args);

// ============================================================
// Core Services
// ============================================================
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// ============================================================
// Database
// ============================================================
// builder.Services.AddDbContext<ServiceDbContext>(options =>
//     options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

// ============================================================
// Cache
// ============================================================
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "ShopThai:";
});

// ============================================================
// Feature Management
// ============================================================
builder.Services.AddFeatureManagement()
    .AddFeatureFilter<Microsoft.FeatureManagement.FeatureFilters.PercentageFilter>()
    .AddFeatureFilter<Microsoft.FeatureManagement.FeatureFilters.TimeWindowFilter>();
builder.Services.AddScoped<IFeatureFlagService, FeatureFlagService>();

// ============================================================
// Telemetry
// ============================================================
builder.Services.AddShopThaiTelemetry(
    builder.Configuration,
    "ShopThai.ServiceName");

// ============================================================
// Health Checks
// ============================================================
builder.Services.AddHealthChecks()
    .AddShopThaiHealthChecks(
        "service-name",
        builder.Configuration.GetConnectionString("DefaultConnection")!,
        builder.Configuration.GetConnectionString("Redis")!);

// ============================================================
// Graceful Shutdown
// ============================================================
builder.Services.AddShopThaiGracefulShutdown(shutdownTimeoutSeconds: 30);
builder.Services.AddSingleton<IShutdownTask, CacheShutdownTask>();

// ============================================================
// Logging (Serilog + SEQ)
// ============================================================
builder.Host.UseSerilog((context, config) =>
{
    config
        .ReadFrom.Configuration(context.Configuration)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithEnvironmentName()
        .WriteTo.Console(outputTemplate:
            "[{Timestamp:HH:mm:ss} {Level:u3}] {SourceContext}: {Message:lj}{NewLine}{Exception}")
        .WriteTo.Seq(context.Configuration["SEQ:ServerUrl"] ?? "http://seq:80");
});

var app = builder.Build();

// ============================================================
// Middleware Pipeline
// ============================================================
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseMiddleware<CorrelationIdMiddleware>();
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.MapShopThaiHealthEndpoints();
app.UseShopThaiGracefulShutdown();

// ============================================================
// Database Migration ตอน Startup
// ============================================================
// using (var scope = app.Services.CreateScope())
// {
//     var migrationRunner = scope.ServiceProvider.GetRequiredService<MigrationRunner>();
//     migrationRunner.RunMigrations(
//         app.Configuration.GetConnectionString("DefaultConnection")!,
//         "ServiceName");
// }

app.Run();
```

### Dockerfile มาตรฐานสำหรับทุก Service

```dockerfile
# docker/Dockerfile.template
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy project files
COPY ["Services/OrderService/ShopThai.OrderService.csproj", "Services/OrderService/"]
COPY ["Shared/ShopThai.Shared.Contracts/ShopThai.Shared.Contracts.csproj", "Shared/ShopThai.Shared.Contracts/"]
COPY ["Shared/ShopThai.Shared.Infrastructure/ShopThai.Shared.Infrastructure.csproj", "Shared/ShopThai.Shared.Infrastructure/"]

# Restore dependencies
RUN dotnet restore "Services/OrderService/ShopThai.OrderService.csproj"

# Copy source code
COPY . .

# Build
WORKDIR "/src/Services/OrderService"
RUN dotnet build "ShopThai.OrderService.csproj" -c Release -o /app/build

# Stage 2: Publish
FROM build AS publish
RUN dotnet publish "ShopThai.OrderService.csproj" \
    -c Release \
    -o /app/publish \
    --no-restore \
    /p:UseAppHost=false

# Stage 3: Final
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app

# ติดตั้ง curl สำหรับ Health Check
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# สร้าง Non-root User เพื่อความปลอดภัย
RUN addgroup --gid 1001 shopthai && \
    adduser --uid 1001 --gid 1001 --disabled-password --gecos "" shopthai

COPY --from=publish /app/publish .
RUN chown -R shopthai:shopthai /app

USER shopthai
EXPOSE 8080
EXPOSE 8081

ENTRYPOINT ["dotnet", "ShopThai.OrderService.dll"]
```

---

## สรุปการเรียนรู้ของ Part 91

### สิ่งที่คุณได้เรียนรู้ใน Part นี้

| ขั้นตอน | หัวข้อ | ทักษะที่ได้ |
|---------|---------|------------|
| 901 | Integration Architecture | การออกแบบ Microservices ให้ทำงานร่วมกัน |
| 902 | API Gateway (YARP) | Reverse Proxy, Rate Limiting, Auth Middleware |
| 903 | Service Discovery | Health Aggregation, Service Registration |
| 904 | Distributed Tracing | OpenTelemetry, Jaeger, Correlation IDs |
| 905 | Docker Compose | Container Orchestration, Infrastructure as Code |
| 906 | Database Migrations | DbUp, Migration Strategy, SQL Scripts |
| 907 | Seed Data | Test Data Management, Development Setup |
| 908 | End-to-End Testing | Integration Testing, Async Flow Testing |
| 909 | Feature Flags | Microsoft.FeatureManagement, A/B Testing |
| 910 | Graceful Shutdown | Process Lifecycle, Resource Cleanup |

### เครื่องมือและเทคโนโลยีที่ใช้

```
Infrastructure:
├── YARP (Yet Another Reverse Proxy) - API Gateway
├── OpenTelemetry + Jaeger - Distributed Tracing
├── Serilog + SEQ - Structured Logging
├── PostgreSQL + EF Core - Database
├── Redis - Caching
├── RabbitMQ - Message Broker
└── Elasticsearch - Search Engine

Testing:
├── xUnit - Unit Testing
├── FluentAssertions - Assertions
└── TestContainers - Integration Testing

DevOps:
├── Docker + Docker Compose
├── Health Checks
└── Feature Flags (Microsoft.FeatureManagement)
```

### ขั้นตอนต่อไป

หลังจาก Part 91 แล้ว คุณมีระบบ **ShopThai** ที่สมบูรณ์พร้อม:
- ✅ Microservices Architecture
- ✅ API Gateway with Rate Limiting
- ✅ Distributed Tracing
- ✅ Containerized with Docker
- ✅ Database with Migrations
- ✅ E2E Tests
- ✅ Feature Flags
- ✅ Graceful Shutdown

---

## แบบฝึกหัด (Exercises)

1. **เพิ่ม Circuit Breaker** ใน API Gateway โดยใช้ Polly สำหรับป้องกันการล้มเหลวแบบ Cascade
2. **เพิ่ม GraphQL Gateway** โดยใช้ Hot Chocolate เพื่อรวม APIs ต่างๆ
3. **ทำ Blue-Green Deployment** โดยใช้ Docker Compose กับ NGINX
4. **เพิ่ม API Versioning** ใน YARP Routing เพื่อรองรับ Multiple API Versions
5. **สร้าง Dashboard** ใน Grafana สำหรับ Monitor ระบบ ShopThai ทั้งหมด

---

## การนำทาง (Navigation)

← [Part 90: AI/ML Integration](./part90-capstone-ai-ml.md) | [Part 92: Advanced Patterns & Best Practices](./part92-advanced-patterns.md) →

---

*Part นี้เป็นส่วนหนึ่งของหลักสูตร C# จาก Beginner สู่ Professional 1000 ขั้นตอน*
*สร้างด้วยความรักสำหรับนักพัฒนา C# ชาวไทย 🇹🇭*
