# Part 71: Observability & Monitoring (การสังเกตการณ์และการติดตามระบบ)

## ภาพรวม

Observability คือความสามารถในการเข้าใจสถานะภายในของระบบจากผลลัพธ์ภายนอก ในยุค Microservices และ Cloud-native applications การติดตามระบบอย่างมีประสิทธิภาพเป็นสิ่งจำเป็น Part นี้จะครอบคลุมการตั้งค่า Observability แบบครบวงจรสำหรับ .NET applications

---

## Step 701: Three Pillars of Observability และ OpenTelemetry Overview

### สามเสาหลักของ Observability

Observability ที่ดีประกอบด้วย 3 ส่วนสำคัญ:

1. **Logs (บันทึก)** - บันทึกเหตุการณ์ที่เกิดขึ้นในระบบ พร้อม context
2. **Metrics (ตัวชี้วัด)** - ข้อมูลเชิงตัวเลขที่วัดประสิทธิภาพและพฤติกรรมของระบบ
3. **Traces (การติดตาม)** - การติดตาม request ตลอดทั้ง distributed system

### OpenTelemetry คืออะไร

OpenTelemetry (OTel) คือ open-source observability framework ที่ได้รับการรับรองจาก CNCF (Cloud Native Computing Foundation) ช่วยให้ developer สามารถ instrument, generate, collect และ export telemetry data ได้อย่างสม่ำเสมอ

```
┌─────────────────────────────────────────────────────────┐
│                    Application                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐  │
│  │   Logs   │  │ Metrics  │  │       Traces         │  │
│  └────┬─────┘  └────┬─────┘  └──────────┬───────────┘  │
│       └─────────────┴─────────────────────┘             │
│                     OpenTelemetry SDK                    │
└──────────────────────────┬──────────────────────────────┘
                           │
                    OTLP Protocol
                           │
           ┌───────────────┼───────────────┐
           │               │               │
     ┌─────▼─────┐  ┌──────▼──────┐  ┌────▼──────┐
     │ Prometheus │  │    Jaeger   │  │   Seq     │
     │ + Grafana  │  │   Zipkin    │  │   Loki    │
     └───────────┘  └─────────────┘  └───────────┘
```

### การติดตั้ง OpenTelemetry Packages

```bash
# Core packages
dotnet add package OpenTelemetry
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Api

# Instrumentation packages
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.SqlClient
dotnet add package OpenTelemetry.Instrumentation.Runtime

# Exporters
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
dotnet add package OpenTelemetry.Exporter.Console
dotnet add package OpenTelemetry.Exporter.Prometheus.AspNetCore

# Serilog
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.Seq
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Process
dotnet add package Serilog.Enrichers.Thread
```

### โครงสร้างพื้นฐานของ OpenTelemetry Setup

```csharp
// Program.cs - Basic OpenTelemetry Setup
using OpenTelemetry;
using OpenTelemetry.Metrics;
using OpenTelemetry.Trace;
using OpenTelemetry.Logs;
using OpenTelemetry.Resources;

var builder = WebApplication.CreateBuilder(args);

// กำหนด Resource สำหรับ service
var resourceBuilder = ResourceBuilder.CreateDefault()
    .AddService(
        serviceName: "MyWebApi",
        serviceVersion: "1.0.0",
        serviceInstanceId: Environment.MachineName)
    .AddAttributes(new Dictionary<string, object>
    {
        ["deployment.environment"] = builder.Environment.EnvironmentName,
        ["team.name"] = "platform-team"
    });

// เพิ่ม OpenTelemetry
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .SetResourceBuilder(resourceBuilder)
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSqlClientInstrumentation()
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .SetResourceBuilder(resourceBuilder)
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddPrometheusExporter())
    .WithLogging(logging => logging
        .SetResourceBuilder(resourceBuilder)
        .AddOtlpExporter());

var app = builder.Build();

// เปิด Prometheus endpoint
app.MapPrometheusScrapingEndpoint();

app.Run();
```

### ข้อดีของ OpenTelemetry

- **Vendor-neutral**: ไม่ผูกติดกับ vendor ใด vendor หนึ่ง
- **Auto-instrumentation**: ติดตาม framework ยอดนิยมโดยอัตโนมัติ
- **Standardized**: ใช้ standard เดียวกันทั้ง ecosystem
- **Context Propagation**: ส่ง trace context ระหว่าง services อัตโนมัติ

---

## Step 702: Structured Logging ด้วย Serilog

### ทำไมต้องใช้ Structured Logging

Structured logging แตกต่างจาก plain text logging ตรงที่ข้อมูล log ถูกเก็บเป็น structured data (เช่น JSON) แทนที่จะเป็น string ธรรมดา ทำให้ค้นหา filter และวิเคราะห์ได้ง่ายขึ้นมาก

```csharp
// ❌ Plain text logging - ค้นหาและวิเคราะห์ยาก
_logger.LogInformation("User John created order 123 for $99.99");

// ✅ Structured logging - มี properties ที่ query ได้
_logger.LogInformation("User {UserId} created order {OrderId} for {Amount:C}", 
    "john@example.com", 123, 99.99m);
```

### การตั้งค่า Serilog แบบครบวงจร

```csharp
// Program.cs
using Serilog;
using Serilog.Events;
using Serilog.Formatting.Compact;

// สร้าง logger เบื้องต้นสำหรับ startup errors
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    Log.Information("Starting web application");
    
    var builder = WebApplication.CreateBuilder(args);
    
    // ตั้งค่า Serilog จาก configuration
    builder.Host.UseSerilog((context, services, configuration) => configuration
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .Enrich.WithEnvironmentName()
        .Enrich.WithMachineName()
        .Enrich.WithProcessId()
        .Enrich.WithThreadId()
        .Enrich.WithProperty("Application", context.HostingEnvironment.ApplicationName)
        .Enrich.WithProperty("Version", "1.0.0")
        .WriteTo.Console(new CompactJsonFormatter())
        .WriteTo.File(
            new CompactJsonFormatter(),
            path: "logs/app-.log",
            rollingInterval: RollingInterval.Day,
            retainedFileCountLimit: 7,
            fileSizeLimitBytes: 50_000_000)
        .WriteTo.Seq(
            serverUrl: context.Configuration["Seq:ServerUrl"] ?? "http://localhost:5341",
            apiKey: context.Configuration["Seq:ApiKey"])
    );
    
    builder.Services.AddControllers();
    
    var app = builder.Build();
    
    // เพิ่ม request logging middleware
    app.UseSerilogRequestLogging(options =>
    {
        options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
        options.GetLevel = (httpContext, elapsed, ex) => ex != null
            ? LogEventLevel.Error
            : httpContext.Response.StatusCode > 499
                ? LogEventLevel.Error
                : elapsed > 3000
                    ? LogEventLevel.Warning
                    : LogEventLevel.Information;
        options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
        {
            diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
            diagnosticContext.Set("RequestScheme", httpContext.Request.Scheme);
            diagnosticContext.Set("UserAgent", httpContext.Request.Headers.UserAgent);
            
            if (httpContext.User.Identity?.IsAuthenticated == true)
            {
                diagnosticContext.Set("UserId", httpContext.User.FindFirst("sub")?.Value);
            }
        };
    });
    
    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
}
finally
{
    Log.CloseAndFlush();
}
```

### การตั้งค่าใน appsettings.json

```json
{
  "Serilog": {
    "Using": ["Serilog.Sinks.Console", "Serilog.Sinks.File", "Serilog.Sinks.Seq"],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.Hosting.Lifetime": "Information",
        "Microsoft.EntityFrameworkCore": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "formatter": "Serilog.Formatting.Compact.CompactJsonFormatter, Serilog.Formatting.Compact"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/app-.log",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30,
          "formatter": "Serilog.Formatting.Compact.CompactJsonFormatter, Serilog.Formatting.Compact"
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName", "WithProcessId"],
    "Properties": {
      "Application": "MyWebApi",
      "Environment": "Production"
    }
  },
  "Seq": {
    "ServerUrl": "http://localhost:5341",
    "ApiKey": ""
  }
}
```

### Custom Enrichers

```csharp
// CustomEnrichers/CorrelationIdEnricher.cs
using Serilog.Core;
using Serilog.Events;

public class CorrelationIdEnricher : ILogEventEnricher
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public CorrelationIdEnricher(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        var httpContext = _httpContextAccessor.HttpContext;
        if (httpContext == null) return;

        // เพิ่ม Correlation ID
        var correlationId = httpContext.Request.Headers["X-Correlation-ID"].FirstOrDefault()
            ?? httpContext.TraceIdentifier;
        
        logEvent.AddPropertyIfAbsent(
            propertyFactory.CreateProperty("CorrelationId", correlationId));

        // เพิ่ม User ID ถ้า authenticated
        if (httpContext.User.Identity?.IsAuthenticated == true)
        {
            var userId = httpContext.User.FindFirst("sub")?.Value;
            if (userId != null)
            {
                logEvent.AddPropertyIfAbsent(
                    propertyFactory.CreateProperty("UserId", userId));
            }
        }

        // เพิ่ม Request Path
        logEvent.AddPropertyIfAbsent(
            propertyFactory.CreateProperty("RequestPath", httpContext.Request.Path));
    }
}

// TenantEnricher.cs
public class TenantEnricher : ILogEventEnricher
{
    private readonly ITenantContext _tenantContext;

    public TenantEnricher(ITenantContext tenantContext)
    {
        _tenantContext = tenantContext;
    }

    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        if (_tenantContext.CurrentTenant != null)
        {
            logEvent.AddPropertyIfAbsent(
                propertyFactory.CreateProperty("TenantId", _tenantContext.CurrentTenant.Id));
            logEvent.AddPropertyIfAbsent(
                propertyFactory.CreateProperty("TenantName", _tenantContext.CurrentTenant.Name));
        }
    }
}

// ลงทะเบียน enrichers
builder.Host.UseSerilog((context, services, configuration) => configuration
    .Enrich.With<CorrelationIdEnricher>()
    .Enrich.With<TenantEnricher>()
    // ...
);
```

### Logging Context และ Scopes

```csharp
// OrderService.cs
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger)
    {
        _logger = logger;
    }

    public async Task<Order> ProcessOrderAsync(CreateOrderRequest request)
    {
        // สร้าง log scope เพื่อเพิ่ม properties ทุก log ใน scope นี้
        using var scope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["OrderId"] = request.OrderId,
            ["CustomerId"] = request.CustomerId,
            ["OrderAmount"] = request.TotalAmount
        });

        _logger.LogInformation("Starting order processing for customer {CustomerId}", 
            request.CustomerId);

        try
        {
            // Validate
            await ValidateOrderAsync(request);
            _logger.LogDebug("Order validation passed");

            // Process payment
            var payment = await ProcessPaymentAsync(request);
            _logger.LogInformation("Payment processed successfully. TransactionId: {TransactionId}", 
                payment.TransactionId);

            // Create order
            var order = await CreateOrderInDatabaseAsync(request, payment);
            
            // Structured logging with complex object
            _logger.LogInformation(
                "Order {OrderId} created successfully. Items: {ItemCount}, Total: {Amount:C}",
                order.Id, order.Items.Count, order.TotalAmount);

            return order;
        }
        catch (PaymentException ex)
        {
            _logger.LogError(ex, 
                "Payment failed for order {OrderId}. Reason: {FailureReason}",
                request.OrderId, ex.FailureReason);
            throw;
        }
        catch (Exception ex)
        {
            _logger.LogCritical(ex, 
                "Unexpected error processing order {OrderId}",
                request.OrderId);
            throw;
        }
    }
}
```

---

## Step 703: Metrics ด้วย OpenTelemetry และ Prometheus

### ประเภทของ Metrics

OpenTelemetry รองรับ metric instruments หลายประเภท:

| Instrument | คำอธิบาย | ตัวอย่างการใช้ |
|-----------|----------|---------------|
| **Counter** | นับค่าที่เพิ่มขึ้นเรื่อยๆ | จำนวน requests, จำนวน errors |
| **UpDownCounter** | นับค่าที่เพิ่มหรือลดได้ | จำนวน active connections |
| **Histogram** | วัดการกระจายของค่า | latency, request size |
| **ObservableGauge** | ค่าที่วัดในช่วงเวลาหนึ่ง | CPU usage, memory |
| **ObservableCounter** | counter ที่อ่านค่าแบบ callback | total bytes sent |

### การสร้าง Custom Metrics

```csharp
// Metrics/ApplicationMetrics.cs
using System.Diagnostics.Metrics;

public class ApplicationMetrics : IDisposable
{
    private readonly Meter _meter;
    
    // Counter - นับจำนวน orders
    public readonly Counter<long> OrdersCreated;
    
    // Histogram - วัด latency
    public readonly Histogram<double> OrderProcessingDuration;
    
    // UpDownCounter - นับ active sessions
    public readonly UpDownCounter<long> ActiveUsers;
    
    // ObservableGauge - CPU/Memory
    private double _currentMemoryUsageMb;
    public readonly ObservableGauge<double> MemoryUsage;

    public ApplicationMetrics(IMeterFactory meterFactory)
    {
        _meter = meterFactory.Create("MyWebApi.Application", "1.0.0");

        // สร้าง Counter
        OrdersCreated = _meter.CreateCounter<long>(
            name: "orders.created.total",
            unit: "{orders}",
            description: "Total number of orders created");

        // สร้าง Histogram พร้อม custom boundaries
        OrderProcessingDuration = _meter.CreateHistogram<double>(
            name: "orders.processing.duration",
            unit: "ms",
            description: "Duration of order processing in milliseconds");

        // สร้าง UpDownCounter
        ActiveUsers = _meter.CreateUpDownCounter<long>(
            name: "users.active.count",
            unit: "{users}",
            description: "Number of currently active users");

        // สร้าง ObservableGauge
        MemoryUsage = _meter.CreateObservableGauge<double>(
            name: "process.memory.usage",
            unit: "MB",
            description: "Current memory usage",
            observeValue: () =>
            {
                var process = System.Diagnostics.Process.GetCurrentProcess();
                return process.WorkingSet64 / 1024.0 / 1024.0;
            });
    }

    public void Dispose() => _meter.Dispose();
}

// ลงทะเบียน Metrics
builder.Services.AddSingleton<ApplicationMetrics>();
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddMeter("MyWebApi.Application") // เพิ่ม custom meter
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddPrometheusExporter());
```

### การใช้ Metrics ใน Services

```csharp
// Services/OrderService.cs
public class OrderService
{
    private readonly ApplicationMetrics _metrics;
    private readonly ILogger<OrderService> _logger;

    public OrderService(ApplicationMetrics metrics, ILogger<OrderService> logger)
    {
        _metrics = metrics;
        _logger = logger;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        var startTime = Stopwatch.GetTimestamp();
        
        // สร้าง tag list สำหรับ labels
        var tags = new TagList
        {
            { "order.type", request.OrderType.ToString() },
            { "customer.tier", request.CustomerTier.ToString() },
            { "payment.method", request.PaymentMethod.ToString() }
        };

        try
        {
            var order = await ProcessOrderInternalAsync(request);
            
            // บันทึก success counter
            tags.Add("status", "success");
            _metrics.OrdersCreated.Add(1, tags);
            
            // บันทึก duration
            var elapsed = Stopwatch.GetElapsedTime(startTime).TotalMilliseconds;
            _metrics.OrderProcessingDuration.Record(elapsed, tags);
            
            _logger.LogInformation(
                "Order {OrderId} created in {Duration:F2}ms", 
                order.Id, elapsed);
            
            return order;
        }
        catch (Exception ex)
        {
            // บันทึก failure counter
            tags.Add("status", "error");
            tags.Add("error.type", ex.GetType().Name);
            _metrics.OrdersCreated.Add(1, tags);
            
            var elapsed = Stopwatch.GetElapsedTime(startTime).TotalMilliseconds;
            _metrics.OrderProcessingDuration.Record(elapsed, tags);
            
            throw;
        }
    }
}
```

### การตั้งค่า Prometheus Exporter

```csharp
// Program.cs
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddPrometheusExporter(options =>
        {
            // ปิด default metrics ที่ไม่ต้องการ
            options.DisableTotalNameSuffixForCounters = false;
        }));

// เพิ่ม scraping endpoint
app.MapPrometheusScrapingEndpoint("/metrics");
// หรือใช้ default path /metrics
app.MapPrometheusScrapingEndpoint();
```

### ตัวอย่าง Prometheus Metrics Output

```
# HELP orders_created_total Total number of orders created
# TYPE orders_created_total counter
orders_created_total{order_type="online",customer_tier="premium",payment_method="card",status="success"} 1523
orders_created_total{order_type="online",customer_tier="standard",payment_method="card",status="success"} 8921
orders_created_total{order_type="online",customer_tier="standard",payment_method="card",status="error"} 45

# HELP orders_processing_duration_ms Duration of order processing in milliseconds
# TYPE orders_processing_duration_ms histogram
orders_processing_duration_ms_bucket{le="0"} 0
orders_processing_duration_ms_bucket{le="5"} 1200
orders_processing_duration_ms_bucket{le="10"} 3500
orders_processing_duration_ms_bucket{le="25"} 7800
orders_processing_duration_ms_bucket{le="50"} 9100
orders_processing_duration_ms_bucket{le="75"} 9600
orders_processing_duration_ms_bucket{le="100"} 9850
orders_processing_duration_ms_bucket{le="+Inf"} 10444
orders_processing_duration_ms_sum 89234.5
orders_processing_duration_ms_count 10444
```

### Built-in Runtime Metrics

```csharp
// เพิ่ม Runtime Instrumentation
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddRuntimeInstrumentation() // GC, Threads, Exception metrics
        .AddProcessInstrumentation()); // CPU, Memory metrics

// Runtime metrics ที่ได้รับโดยอัตโนมัติ:
// process.runtime.dotnet.gc.collections.count
// process.runtime.dotnet.gc.objects.size
// process.runtime.dotnet.threadpool.threads.count
// process.runtime.dotnet.threadpool.queue.length
// process.runtime.dotnet.exceptions.count
```

---

## Step 704: Distributed Tracing ด้วย OpenTelemetry

### ความเข้าใจเรื่อง Distributed Tracing

ใน distributed system request หนึ่งอาจผ่าน services หลายตัว Distributed Tracing ช่วยให้เราติดตาม request ทั้งหมดตลอดเส้นทาง

```
Browser → API Gateway → Order Service → Payment Service
                                     → Inventory Service
                                     → Notification Service
```

แต่ละ service สร้าง **Span** ซึ่งรวมกันเป็น **Trace** ทั้งหมด

### Activity และ Span ใน .NET

```csharp
// Tracing/OrderTracer.cs
using System.Diagnostics;

public static class OrderTracer
{
    // สร้าง ActivitySource สำหรับ service
    public static readonly ActivitySource Source = 
        new ActivitySource("MyWebApi.Orders", "1.0.0");
}

// การใช้ Custom Traces ใน Service
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        // เริ่ม span หลัก
        using var activity = OrderTracer.Source.StartActivity(
            "CreateOrder",
            ActivityKind.Internal);

        // เพิ่ม attributes เข้า span
        activity?.SetTag("order.id", request.OrderId.ToString());
        activity?.SetTag("customer.id", request.CustomerId.ToString());
        activity?.SetTag("order.amount", request.TotalAmount);
        activity?.SetTag("order.item_count", request.Items.Count);

        try
        {
            // Validate - สร้าง child span
            using (var validateSpan = OrderTracer.Source.StartActivity("ValidateOrder"))
            {
                validateSpan?.SetTag("validation.rules_count", 10);
                await ValidateOrderAsync(request);
                validateSpan?.SetTag("validation.result", "passed");
            }

            // Payment - สร้าง child span สำหรับ external call
            Order order;
            using (var paymentSpan = OrderTracer.Source.StartActivity(
                "ProcessPayment", ActivityKind.Client))
            {
                paymentSpan?.SetTag("payment.provider", "stripe");
                paymentSpan?.SetTag("payment.amount", request.TotalAmount);
                
                var payment = await _paymentService.ChargeAsync(request);
                
                paymentSpan?.SetTag("payment.transaction_id", payment.TransactionId);
                paymentSpan?.SetStatus(ActivityStatusCode.Ok);

                order = await SaveOrderAsync(request, payment);
            }

            // เพิ่ม event เข้า span
            activity?.AddEvent(new ActivityEvent("OrderSaved", 
                tags: new ActivityTagsCollection
                {
                    { "order.id", order.Id.ToString() },
                    { "db.table", "orders" }
                }));

            activity?.SetStatus(ActivityStatusCode.Ok);
            activity?.SetTag("order.created_id", order.Id.ToString());
            
            return order;
        }
        catch (Exception ex)
        {
            // บันทึก error ใน span
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            throw;
        }
    }
}
```

### Baggage - ส่ง Context ระหว่าง Services

```csharp
// การส่ง Baggage ใน parent service
public async Task ProcessRequestAsync(HttpContext httpContext)
{
    // เพิ่ม Baggage สำหรับส่งต่อไปยัง downstream services
    Baggage.SetBaggage("tenant.id", httpContext.User.FindFirst("tenant")?.Value ?? "");
    Baggage.SetBaggage("user.id", httpContext.User.FindFirst("sub")?.Value ?? "");
    Baggage.SetBaggage("request.priority", "high");
    
    // เรียก downstream service - baggage จะถูกส่งอัตโนมัติผ่าน W3C TraceContext header
    await _orderService.CreateOrderAsync(request);
}

// การอ่าน Baggage ใน child service
public async Task ProcessInternalAsync()
{
    // อ่าน baggage ที่ได้รับจาก parent
    var tenantId = Baggage.GetBaggage("tenant.id");
    var userId = Baggage.GetBaggage("user.id");
    var priority = Baggage.GetBaggage("request.priority");
    
    using var activity = OrderTracer.Source.StartActivity("ProcessInternal");
    activity?.SetTag("tenant.id", tenantId);
    activity?.SetTag("user.id", userId);
    activity?.SetTag("request.priority", priority);
    
    // ... process logic
}
```

### Context Propagation

```csharp
// การ configure tracing พร้อม propagators
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .SetResourceBuilder(resourceBuilder)
        .AddSource("MyWebApi.Orders")        // เพิ่ม custom ActivitySource
        .AddSource("MyWebApi.Payments")
        .AddAspNetCoreInstrumentation(options =>
        {
            options.RecordException = true;
            options.EnrichWithHttpRequest = (activity, request) =>
            {
                activity.SetTag("http.client_ip", request.HttpContext.Connection.RemoteIpAddress?.ToString());
                activity.SetTag("http.user_agent", request.Headers.UserAgent);
            };
            options.EnrichWithHttpResponse = (activity, response) =>
            {
                activity.SetTag("http.response_content_length", response.ContentLength);
            };
        })
        .AddHttpClientInstrumentation(options =>
        {
            options.RecordException = true;
            options.EnrichWithHttpRequestMessage = (activity, request) =>
            {
                activity.SetTag("http.request.target_service", 
                    request.RequestUri?.Host);
            };
        })
        .AddJaegerExporter(options =>
        {
            options.AgentHost = "localhost";
            options.AgentPort = 6831;
        })
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri("http://localhost:4317");
        }));
```

---

## Step 705: Health Checks พร้อม UI Dashboard

### การตั้งค่า Health Checks

Health Checks ช่วยให้ monitoring systems รู้ว่า application ยังทำงานได้ถูกต้อง

```bash
dotnet add package Microsoft.Extensions.Diagnostics.HealthChecks
dotnet add package AspNetCore.HealthChecks.UI
dotnet add package AspNetCore.HealthChecks.UI.Client
dotnet add package AspNetCore.HealthChecks.UI.InMemory.Storage
dotnet add package AspNetCore.HealthChecks.SqlServer
dotnet add package AspNetCore.HealthChecks.Redis
dotnet add package AspNetCore.HealthChecks.Rabbitmq
dotnet add package AspNetCore.HealthChecks.Uris
```

```csharp
// Program.cs
builder.Services
    .AddHealthChecks()
    // ตรวจสอบ Database
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "sql-server",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "db", "sql" })
    // ตรวจสอบ Redis
    .AddRedis(
        redisConnectionString: builder.Configuration.GetConnectionString("Redis")!,
        name: "redis",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "cache", "redis" })
    // ตรวจสอบ External API
    .AddUrlGroup(
        uri: new Uri("https://api.payment-provider.com/health"),
        name: "payment-api",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "external", "payment" })
    // Custom Health Check
    .AddCheck<DatabaseConnectionHealthCheck>(
        name: "db-connection-pool",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "db" })
    .AddCheck<DiskSpaceHealthCheck>(
        name: "disk-space",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "infrastructure" });

// Health Checks UI
builder.Services.AddHealthChecksUI(settings =>
{
    settings.SetEvaluationTimeInSeconds(30);    // ตรวจทุก 30 วินาที
    settings.SetMinimumSecondsBetweenFailureNotifications(60);
    settings.MaximumHistoryEntriesPerEndpoint(50);
    settings.AddHealthCheckEndpoint("API", "/healthz");
}).AddInMemoryStorage();
```

### Custom Health Checks

```csharp
// HealthChecks/DatabaseConnectionHealthCheck.cs
public class DatabaseConnectionHealthCheck : IHealthCheck
{
    private readonly IDbConnectionFactory _connectionFactory;
    private readonly ILogger<DatabaseConnectionHealthCheck> _logger;

    public DatabaseConnectionHealthCheck(
        IDbConnectionFactory connectionFactory,
        ILogger<DatabaseConnectionHealthCheck> logger)
    {
        _connectionFactory = connectionFactory;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            await using var connection = _connectionFactory.CreateConnection();
            await connection.OpenAsync(cancellationToken);
            
            using var command = connection.CreateCommand();
            command.CommandText = "SELECT 1";
            command.CommandTimeout = 5;
            
            await command.ExecuteScalarAsync(cancellationToken);
            
            return HealthCheckResult.Healthy("Database connection is healthy", 
                data: new Dictionary<string, object>
                {
                    ["server"] = connection.DataSource,
                    ["database"] = connection.Database,
                    ["response_time_ms"] = 0 // วัด actual time
                });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Database health check failed");
            return HealthCheckResult.Unhealthy(
                "Database connection failed", 
                exception: ex);
        }
    }
}

// HealthChecks/DiskSpaceHealthCheck.cs
public class DiskSpaceHealthCheck : IHealthCheck
{
    private readonly long _minimumFreeMegabytes;

    public DiskSpaceHealthCheck(long minimumFreeMegabytes = 500)
    {
        _minimumFreeMegabytes = minimumFreeMegabytes;
    }

    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var drives = DriveInfo.GetDrives()
            .Where(d => d.IsReady)
            .Select(d => new
            {
                d.Name,
                FreeMB = d.AvailableFreeSpace / 1024 / 1024,
                TotalMB = d.TotalSize / 1024 / 1024,
                UsagePercent = 100 - (d.AvailableFreeSpace * 100 / d.TotalSize)
            })
            .ToList();

        var criticalDrives = drives.Where(d => d.FreeMB < _minimumFreeMegabytes).ToList();
        var data = drives.ToDictionary(
            d => d.Name,
            d => (object)$"{d.FreeMB:N0} MB free ({d.UsagePercent}% used)");

        if (criticalDrives.Any())
        {
            return Task.FromResult(HealthCheckResult.Degraded(
                $"Low disk space on: {string.Join(", ", criticalDrives.Select(d => d.Name))}",
                data: data));
        }

        return Task.FromResult(HealthCheckResult.Healthy("Disk space is sufficient", data: data));
    }
}
```

### Health Check Endpoints

```csharp
// Program.cs - ตั้งค่า endpoints
app.MapHealthChecks("/healthz", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
    Predicate = _ => true // แสดงทุก check
});

// แยก endpoints ตาม tag
app.MapHealthChecks("/healthz/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("readiness"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/healthz/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("liveness"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

// Health Checks UI
app.MapHealthChecksUI(options =>
{
    options.UIPath = "/health-ui";       // URL สำหรับเข้าดู UI
    options.ApiPath = "/health-api";
});
```

---

## Step 706: Application Performance Monitoring (APM) Integration

### APM Solutions ที่นิยม

APM tools ช่วย profile และวิเคราะห์ performance ของ application อย่างละเอียด

```csharp
// การ integrate กับ Elastic APM
// dotnet add package Elastic.Apm.NetCoreAll

// Program.cs
builder.Services.AddAllElasticApm();

// appsettings.json
{
  "ElasticApm": {
    "ServiceName": "MyWebApi",
    "ServiceVersion": "1.0.0",
    "Environment": "production",
    "ServerUrl": "http://apm-server:8200",
    "SecretToken": "your-secret-token",
    "LogLevel": "Warning",
    "TransactionSampleRate": 1.0,
    "CaptureBody": "all",
    "CaptureHeaders": true
  }
}
```

### Performance Profiling Middleware

```csharp
// Middleware/PerformanceProfilingMiddleware.cs
public class PerformanceProfilingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<PerformanceProfilingMiddleware> _logger;
    private readonly ApplicationMetrics _metrics;
    private static readonly double SlowRequestThresholdMs = 1000;

    public PerformanceProfilingMiddleware(
        RequestDelegate next,
        ILogger<PerformanceProfilingMiddleware> logger,
        ApplicationMetrics metrics)
    {
        _next = next;
        _logger = logger;
        _metrics = metrics;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var startTime = Stopwatch.GetTimestamp();
        var originalBodyStream = context.Response.Body;

        try
        {
            await _next(context);
        }
        finally
        {
            var elapsed = Stopwatch.GetElapsedTime(startTime).TotalMilliseconds;

            // บันทึก metrics
            var tags = new TagList
            {
                { "http.method", context.Request.Method },
                { "http.route", context.GetEndpoint()?.DisplayName ?? "unknown" },
                { "http.status_code", context.Response.StatusCode }
            };

            _metrics.HttpRequestDuration.Record(elapsed, tags);

            // Log slow requests
            if (elapsed > SlowRequestThresholdMs)
            {
                _logger.LogWarning(
                    "Slow request detected: {Method} {Path} took {Duration:F2}ms (threshold: {Threshold}ms)",
                    context.Request.Method,
                    context.Request.Path,
                    elapsed,
                    SlowRequestThresholdMs);
            }
        }
    }
}
```

### Database Query Performance Tracking

```csharp
// Data/ObservableDbContext.cs
public class ObservableDbContext : DbContext
{
    private readonly ILogger<ObservableDbContext> _logger;
    private readonly ApplicationMetrics _metrics;

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.AddInterceptors(new QueryPerformanceInterceptor(_logger, _metrics));
    }
}

// Interceptors/QueryPerformanceInterceptor.cs
public class QueryPerformanceInterceptor : DbCommandInterceptor
{
    private readonly ILogger _logger;
    private readonly ApplicationMetrics _metrics;

    public override ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result,
        CancellationToken cancellationToken = default)
    {
        TrackQueryPerformance(command, eventData);
        return new ValueTask<DbDataReader>(result);
    }

    private void TrackQueryPerformance(DbCommand command, CommandExecutedEventData eventData)
    {
        var duration = eventData.Duration.TotalMilliseconds;
        var queryType = GetQueryType(command.CommandText);

        var tags = new TagList
        {
            { "db.operation", queryType },
            { "db.table", ExtractTableName(command.CommandText) }
        };

        _metrics.DatabaseQueryDuration.Record(duration, tags);

        // Log slow queries
        if (duration > 500)
        {
            _logger.LogWarning(
                "Slow query ({Duration:F2}ms): {Query}",
                duration,
                command.CommandText[..Math.Min(200, command.CommandText.Length)]);
        }
    }

    private static string GetQueryType(string sql)
    {
        var trimmed = sql.TrimStart().ToUpperInvariant();
        return trimmed switch
        {
            var s when s.StartsWith("SELECT") => "SELECT",
            var s when s.StartsWith("INSERT") => "INSERT",
            var s when s.StartsWith("UPDATE") => "UPDATE",
            var s when s.StartsWith("DELETE") => "DELETE",
            _ => "OTHER"
        };
    }
}
```

---

## Step 707: Custom Metrics และ Dashboards

### สร้าง Business Metrics ที่มีความหมาย

```csharp
// Metrics/BusinessMetrics.cs
public class BusinessMetrics
{
    private readonly Meter _meter;

    // Revenue metrics
    public readonly Counter<double> RevenueTotal;
    public readonly Histogram<double> OrderValue;

    // User engagement metrics
    public readonly Counter<long> UserLogins;
    public readonly Counter<long> FeatureUsage;
    public readonly UpDownCounter<long> ActiveSessions;

    // Error rate metrics
    public readonly Counter<long> ValidationErrors;
    public readonly Counter<long> PaymentDeclines;
    public readonly Counter<long> ApiCallErrors;

    // SLA metrics
    public readonly Histogram<double> ApiResponseTime;
    public readonly ObservableGauge<double> SlaCompliancePercent;

    private double _currentSlaCompliance = 99.5;

    public BusinessMetrics(IMeterFactory meterFactory)
    {
        _meter = meterFactory.Create("MyWebApi.Business", "1.0.0");

        RevenueTotal = _meter.CreateCounter<double>(
            "business.revenue.total",
            unit: "THB",
            description: "Total revenue generated");

        OrderValue = _meter.CreateHistogram<double>(
            "business.order.value",
            unit: "THB",
            description: "Distribution of order values");

        UserLogins = _meter.CreateCounter<long>(
            "business.user.logins.total",
            description: "Total user login events");

        FeatureUsage = _meter.CreateCounter<long>(
            "business.feature.usage.total",
            description: "Feature usage tracking");

        ActiveSessions = _meter.CreateUpDownCounter<long>(
            "business.sessions.active",
            unit: "{sessions}",
            description: "Current active user sessions");

        ValidationErrors = _meter.CreateCounter<long>(
            "business.errors.validation.total",
            description: "Total validation errors");

        PaymentDeclines = _meter.CreateCounter<long>(
            "business.payment.declines.total",
            description: "Total payment declines");

        ApiCallErrors = _meter.CreateCounter<long>(
            "business.api.errors.total",
            description: "Total API call errors");

        ApiResponseTime = _meter.CreateHistogram<double>(
            "business.api.response_time",
            unit: "ms",
            description: "API response time distribution");

        SlaCompliancePercent = _meter.CreateObservableGauge<double>(
            "business.sla.compliance_percent",
            unit: "%",
            description: "Current SLA compliance percentage",
            observeValue: () => _currentSlaCompliance);
    }

    public void UpdateSlaCompliance(double percentage)
    {
        _currentSlaCompliance = percentage;
    }
}
```

### Grafana Dashboard Configuration (JSON)

```json
{
  "dashboard": {
    "id": null,
    "title": "MyWebApi - Application Dashboard",
    "tags": ["application", "api", "business"],
    "timezone": "Asia/Bangkok",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "graph",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "rate(http_server_duration_milliseconds_count[5m])",
            "legendFormat": "{{http_route}} - {{http_status_code}}"
          }
        ]
      },
      {
        "id": 2,
        "title": "Error Rate",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 12, "y": 0},
        "targets": [
          {
            "expr": "sum(rate(http_server_duration_milliseconds_count{http_status_code=~'5..'}[5m])) / sum(rate(http_server_duration_milliseconds_count[5m])) * 100",
            "legendFormat": "Error Rate %"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                {"color": "green", "value": 0},
                {"color": "yellow", "value": 1},
                {"color": "red", "value": 5}
              ]
            }
          }
        }
      },
      {
        "id": 3,
        "title": "P95 Latency",
        "type": "graph",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8},
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_server_duration_milliseconds_bucket[5m]))",
            "legendFormat": "P95 {{http_route}}"
          },
          {
            "expr": "histogram_quantile(0.99, rate(http_server_duration_milliseconds_bucket[5m]))",
            "legendFormat": "P99 {{http_route}}"
          }
        ]
      }
    ]
  }
}
```

---

## Step 708: Alerting Patterns

### การสร้าง Alert Rules ใน Prometheus

```yaml
# prometheus-alerts.yaml
groups:
  - name: application_alerts
    interval: 30s
    rules:
      # Alert เมื่อ error rate สูงกว่า 5%
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_duration_milliseconds_count{http_status_code=~"5.."}[5m]))
          /
          sum(rate(http_server_duration_milliseconds_count[5m]))
          > 0.05
        for: 2m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"
          runbook_url: "https://wiki.internal/runbooks/high-error-rate"
          
      # Alert เมื่อ latency สูงเกิน 2 วินาที
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            rate(http_server_duration_milliseconds_bucket[5m])
          ) > 2000
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High API latency detected"
          description: "P95 latency is {{ $value }}ms for {{ $labels.http_route }}"
          
      # Alert เมื่อ memory สูงเกิน 80%
      - alert: HighMemoryUsage
        expr: process_memory_usage_MB > 800
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value }}MB"
          
      # Alert เมื่อ health check ล้มเหลว
      - alert: HealthCheckFailing
        expr: health_checks_status{status="Unhealthy"} == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Health check failing: {{ $labels.check_name }}"
```

### Alert Manager Configuration

```yaml
# alertmanager.yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'

route:
  receiver: 'default'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  routes:
    - matchers:
        - severity = critical
      receiver: 'critical-alerts'
      repeat_interval: 1h
    - matchers:
        - severity = warning
      receiver: 'warning-alerts'

receivers:
  - name: 'critical-alerts'
    slack_configs:
      - channel: '#alerts-critical'
        title: '🚨 Critical Alert: {{ .GroupLabels.alertname }}'
        text: |
          *Summary*: {{ range .Alerts }}{{ .Annotations.summary }}{{ end }}
          *Description*: {{ range .Alerts }}{{ .Annotations.description }}{{ end }}
        send_resolved: true
    email_configs:
      - to: 'oncall-team@company.com'
        subject: '🚨 CRITICAL: {{ .GroupLabels.alertname }}'
        
  - name: 'warning-alerts'
    slack_configs:
      - channel: '#alerts-warning'
        title: '⚠️ Warning: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
```

### Programmatic Alerting ใน C#

```csharp
// Services/AlertingService.cs
public interface IAlertingService
{
    Task SendAlertAsync(Alert alert, CancellationToken cancellationToken = default);
}

public record Alert(
    string Title,
    string Description,
    AlertSeverity Severity,
    string? RunbookUrl = null,
    Dictionary<string, string>? Labels = null);

public enum AlertSeverity { Info, Warning, Error, Critical }

public class SlackAlertingService : IAlertingService
{
    private readonly HttpClient _httpClient;
    private readonly string _webhookUrl;
    private readonly ILogger<SlackAlertingService> _logger;

    public async Task SendAlertAsync(Alert alert, CancellationToken cancellationToken = default)
    {
        var emoji = alert.Severity switch
        {
            AlertSeverity.Info => "ℹ️",
            AlertSeverity.Warning => "⚠️",
            AlertSeverity.Error => "🔴",
            AlertSeverity.Critical => "🚨",
            _ => "📢"
        };

        var payload = new
        {
            attachments = new[]
            {
                new
                {
                    color = GetColor(alert.Severity),
                    title = $"{emoji} {alert.Title}",
                    text = alert.Description,
                    footer = $"Severity: {alert.Severity}",
                    ts = DateTimeOffset.UtcNow.ToUnixTimeSeconds()
                }
            }
        };

        var json = JsonSerializer.Serialize(payload);
        var content = new StringContent(json, Encoding.UTF8, "application/json");

        try
        {
            var response = await _httpClient.PostAsync(_webhookUrl, content, cancellationToken);
            response.EnsureSuccessStatusCode();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to send alert: {AlertTitle}", alert.Title);
        }
    }

    private static string GetColor(AlertSeverity severity) => severity switch
    {
        AlertSeverity.Info => "good",
        AlertSeverity.Warning => "warning",
        AlertSeverity.Error => "danger",
        AlertSeverity.Critical => "#FF0000",
        _ => "#808080"
    };
}

// Background service สำหรับตรวจสอบ metrics และส่ง alert
public class MetricsAlertingBackgroundService : BackgroundService
{
    private readonly IAlertingService _alertingService;
    private readonly ApplicationMetrics _metrics;
    private readonly ILogger<MetricsAlertingBackgroundService> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await CheckMetricsAndAlertAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in metrics alerting service");
            }

            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }

    private async Task CheckMetricsAndAlertAsync(CancellationToken cancellationToken)
    {
        // ตรวจสอบ error rate
        var errorRate = _metrics.GetCurrentErrorRate();
        if (errorRate > 0.05)
        {
            await _alertingService.SendAlertAsync(new Alert(
                Title: "High Error Rate Detected",
                Description: $"Current error rate: {errorRate:P2} (threshold: 5%)",
                Severity: AlertSeverity.Critical,
                RunbookUrl: "https://wiki.internal/runbooks/error-rate"
            ), cancellationToken);
        }
    }
}
```

---

## Step 709: Log Aggregation (Seq, ELK, Azure Monitor)

### การตั้งค่า Seq

Seq คือ structured log server ที่ใช้งานง่ายและเหมาะสำหรับ development และ small-scale production

```csharp
// การตั้งค่า Serilog ให้ส่งไปยัง Seq
builder.Host.UseSerilog((context, services, configuration) => configuration
    .WriteTo.Seq(
        serverUrl: "http://seq-server:5341",
        apiKey: context.Configuration["Seq:ApiKey"],
        restrictedToMinimumLevel: LogEventLevel.Information,
        batchPostingLimit: 1000,
        period: TimeSpan.FromSeconds(2),
        controlLevelSwitch: new LoggingLevelSwitch())
);
```

### การตั้งค่า ELK Stack

```csharp
// dotnet add package Serilog.Sinks.Elasticsearch
builder.Host.UseSerilog((context, services, configuration) => configuration
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(
        new Uri("http://elasticsearch:9200"))
    {
        AutoRegisterTemplate = true,
        AutoRegisterTemplateVersion = AutoRegisterTemplateVersion.ESv7,
        IndexFormat = $"mywebapi-{DateTime.UtcNow:yyyy-MM}",
        NumberOfShards = 2,
        NumberOfReplicas = 1,
        MinimumLogEventLevel = LogEventLevel.Information,
        BatchAction = ElasticOpType.Create,
        ModifyConnectionSettings = conn =>
            conn.BasicAuthentication("elastic", "password")
    })
);
```

### การตั้งค่า Azure Monitor

```csharp
// dotnet add package Azure.Monitor.OpenTelemetry.AspNetCore

// Program.cs
builder.Services.AddOpenTelemetry()
    .UseAzureMonitor(options =>
    {
        options.ConnectionString = builder.Configuration["AzureMonitor:ConnectionString"];
        options.SamplingRatio = 1.0f; // เก็บ 100% ของ traces
    });

// appsettings.json
{
  "AzureMonitor": {
    "ConnectionString": "InstrumentationKey=your-key;IngestionEndpoint=https://eastasia-0.in.applicationinsights.azure.com/"
  }
}
```

### Log Aggregation Patterns

```csharp
// LogAggregation/LogAggregationService.cs
public class LogAggregationService
{
    private readonly ILogger<LogAggregationService> _logger;
    
    // ใช้ structured logging เพื่อ enable aggregation
    public void LogOrderEvent(OrderEvent orderEvent)
    {
        _logger.LogInformation(
            "OrderEvent {@OrderEvent}",
            new
            {
                EventType = orderEvent.Type,
                OrderId = orderEvent.OrderId,
                CustomerId = orderEvent.CustomerId,
                Amount = orderEvent.Amount,
                Currency = orderEvent.Currency,
                Timestamp = orderEvent.Timestamp,
                ProcessingTime = orderEvent.ProcessingTimeMs,
                Status = orderEvent.Status,
                Region = orderEvent.Region
            });
    }
    
    // Correlation ID สำหรับ trace ระหว่าง services
    public IDisposable BeginCorrelationScope(string correlationId)
    {
        return LogContext.PushProperty("CorrelationId", correlationId);
    }
}

// Middleware/CorrelationIdMiddleware.cs
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;
    private const string CorrelationIdHeader = "X-Correlation-ID";

    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString("N");

        context.Response.Headers[CorrelationIdHeader] = correlationId;
        
        using (LogContext.PushProperty("CorrelationId", correlationId))
        using (LogContext.PushProperty("RequestId", context.TraceIdentifier))
        {
            await _next(context);
        }
    }
}
```

### Log Query Examples

```
// Seq Queries
// หา orders ที่ fail ในชั่วโมงที่ผ่านมา
EventType = 'OrderFailed' && @Timestamp > Now() - 1h

// หา slow requests (>1000ms)
RequestDuration > 1000 && @Level = 'Warning'

// Kibana (ELK) Queries  
// หา errors ตาม service
GET /mywebapi-*/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "level": "Error" } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "aggs": {
    "errors_by_service": {
      "terms": { "field": "fields.Application.keyword" }
    }
  }
}
```

---

## Step 710: Complete Observability Setup สำหรับ ASP.NET Core App

### สรุปการตั้งค่าแบบครบวงจร

```csharp
// ObservabilityExtensions.cs - Extension method สำหรับ setup ทั้งหมด
using OpenTelemetry;
using OpenTelemetry.Metrics;
using OpenTelemetry.Trace;
using OpenTelemetry.Logs;
using OpenTelemetry.Resources;
using Serilog;
using Serilog.Events;

public static class ObservabilityExtensions
{
    public static WebApplicationBuilder AddObservability(
        this WebApplicationBuilder builder,
        ObservabilityOptions? options = null)
    {
        options ??= new ObservabilityOptions();
        
        // 1. ตั้งค่า Serilog
        builder.Host.UseSerilog((context, services, configuration) =>
        {
            configuration
                .MinimumLevel.Information()
                .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
                .MinimumLevel.Override("System", LogEventLevel.Warning)
                .Enrich.FromLogContext()
                .Enrich.WithEnvironmentName()
                .Enrich.WithMachineName()
                .Enrich.WithProcessId()
                .Enrich.WithProperty("Application", options.ServiceName)
                .Enrich.WithProperty("Version", options.ServiceVersion);

            // Console output
            if (options.EnableConsoleLogging)
            {
                configuration.WriteTo.Console(
                    outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties}{NewLine}{Exception}");
            }

            // File output
            if (!string.IsNullOrEmpty(options.LogFilePath))
            {
                configuration.WriteTo.File(
                    path: options.LogFilePath,
                    rollingInterval: RollingInterval.Day,
                    retainedFileCountLimit: 7);
            }

            // Seq output
            if (!string.IsNullOrEmpty(options.SeqServerUrl))
            {
                configuration.WriteTo.Seq(options.SeqServerUrl);
            }
        });

        // 2. ตั้งค่า Resource
        var resourceBuilder = ResourceBuilder.CreateDefault()
            .AddService(
                serviceName: options.ServiceName,
                serviceVersion: options.ServiceVersion,
                serviceInstanceId: Environment.MachineName)
            .AddAttributes(new Dictionary<string, object>
            {
                ["deployment.environment"] = builder.Environment.EnvironmentName
            });

        // 3. ตั้งค่า OpenTelemetry
        builder.Services.AddOpenTelemetry()
            .WithTracing(tracing =>
            {
                tracing
                    .SetResourceBuilder(resourceBuilder)
                    .AddAspNetCoreInstrumentation(opts =>
                    {
                        opts.RecordException = true;
                    })
                    .AddHttpClientInstrumentation(opts =>
                    {
                        opts.RecordException = true;
                    })
                    .AddSqlClientInstrumentation(opts =>
                    {
                        opts.RecordException = true;
                        opts.SetDbStatementForText = true;
                    });

                // เพิ่ม custom sources
                foreach (var source in options.AdditionalActivitySources)
                    tracing.AddSource(source);

                if (!string.IsNullOrEmpty(options.OtlpEndpoint))
                    tracing.AddOtlpExporter(otlp =>
                        otlp.Endpoint = new Uri(options.OtlpEndpoint));
            })
            .WithMetrics(metrics =>
            {
                metrics
                    .SetResourceBuilder(resourceBuilder)
                    .AddAspNetCoreInstrumentation()
                    .AddHttpClientInstrumentation()
                    .AddRuntimeInstrumentation();

                // เพิ่ม custom meters
                foreach (var meter in options.AdditionalMeters)
                    metrics.AddMeter(meter);

                if (options.EnablePrometheus)
                    metrics.AddPrometheusExporter();

                if (!string.IsNullOrEmpty(options.OtlpEndpoint))
                    metrics.AddOtlpExporter(otlp =>
                        otlp.Endpoint = new Uri(options.OtlpEndpoint));
            })
            .WithLogging(logging =>
            {
                logging.SetResourceBuilder(resourceBuilder);

                if (!string.IsNullOrEmpty(options.OtlpEndpoint))
                    logging.AddOtlpExporter(otlp =>
                        otlp.Endpoint = new Uri(options.OtlpEndpoint));
            });

        // 4. ตั้งค่า Health Checks
        var healthChecks = builder.Services.AddHealthChecks();
        options.HealthCheckConfigurator?.Invoke(healthChecks);

        if (options.EnableHealthCheckUI)
        {
            builder.Services
                .AddHealthChecksUI(settings =>
                {
                    settings.SetEvaluationTimeInSeconds(30);
                    settings.AddHealthCheckEndpoint("API", "/healthz");
                })
                .AddInMemoryStorage();
        }

        // 5. ลงทะเบียน custom metrics
        builder.Services.AddSingleton<ApplicationMetrics>();
        builder.Services.AddSingleton<BusinessMetrics>();
        
        // 6. ลงทะเบียน alerting
        builder.Services.AddScoped<IAlertingService, SlackAlertingService>();

        return builder;
    }

    public static WebApplication UseObservability(
        this WebApplication app,
        ObservabilityOptions? options = null)
    {
        options ??= new ObservabilityOptions();

        // Serilog request logging
        app.UseSerilogRequestLogging(opts =>
        {
            opts.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000}ms";
            opts.GetLevel = (ctx, elapsed, ex) =>
                ex != null || ctx.Response.StatusCode >= 500
                    ? LogEventLevel.Error
                    : elapsed > 1000
                        ? LogEventLevel.Warning
                        : LogEventLevel.Information;
        });

        // Prometheus endpoint
        if (options.EnablePrometheus)
        {
            app.MapPrometheusScrapingEndpoint("/metrics");
        }

        // Health check endpoints
        app.MapHealthChecks("/healthz", new HealthCheckOptions
        {
            ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
        });

        app.MapHealthChecks("/healthz/live", new HealthCheckOptions
        {
            Predicate = check => check.Tags.Contains("liveness"),
            ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
        });

        app.MapHealthChecks("/healthz/ready", new HealthCheckOptions
        {
            Predicate = check => check.Tags.Contains("readiness"),
            ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
        });

        if (options.EnableHealthCheckUI)
        {
            app.MapHealthChecksUI(uiOptions =>
            {
                uiOptions.UIPath = "/health-ui";
            });
        }

        return app;
    }
}

// ObservabilityOptions.cs
public class ObservabilityOptions
{
    public string ServiceName { get; set; } = "MyWebApi";
    public string ServiceVersion { get; set; } = "1.0.0";
    public bool EnableConsoleLogging { get; set; } = true;
    public string? LogFilePath { get; set; }
    public string? SeqServerUrl { get; set; }
    public string? OtlpEndpoint { get; set; }
    public bool EnablePrometheus { get; set; } = true;
    public bool EnableHealthCheckUI { get; set; } = true;
    public List<string> AdditionalActivitySources { get; set; } = [];
    public List<string> AdditionalMeters { get; set; } = [];
    public Action<IHealthChecksBuilder>? HealthCheckConfigurator { get; set; }
}
```

### การใช้งาน Extension Methods

```csharp
// Program.cs - Clean and Complete Setup
var builder = WebApplication.CreateBuilder(args);

// เพิ่ม Observability ทั้งหมดในครั้งเดียว
builder.AddObservability(new ObservabilityOptions
{
    ServiceName = "OrderManagementApi",
    ServiceVersion = "2.1.0",
    LogFilePath = "logs/app-.log",
    SeqServerUrl = builder.Configuration["Seq:ServerUrl"],
    OtlpEndpoint = builder.Configuration["OpenTelemetry:Endpoint"],
    EnablePrometheus = true,
    EnableHealthCheckUI = true,
    AdditionalActivitySources = ["OrderManagementApi.Orders", "OrderManagementApi.Payments"],
    AdditionalMeters = ["OrderManagementApi.Business"],
    HealthCheckConfigurator = checks =>
    {
        checks.AddSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")!);
        checks.AddRedis(builder.Configuration.GetConnectionString("Redis")!);
    }
});

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

var app = builder.Build();

// เปิดใช้งาน Observability middleware
app.UseObservability();
app.MapControllers();
app.Run();
```

### Docker Compose สำหรับ Complete Observability Stack

```yaml
# docker-compose.observability.yaml
version: '3.8'

services:
  # =========================================
  # Application
  # =========================================
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=sqlserver;Database=OrderDb;User=sa;Password=Pass123!
      - ConnectionStrings__Redis=redis:6379
      - Seq__ServerUrl=http://seq:5341
      - OpenTelemetry__Endpoint=http://otel-collector:4317
    depends_on:
      - sqlserver
      - redis
      - seq
      - otel-collector

  # =========================================
  # Logging - Seq
  # =========================================
  seq:
    image: datalust/seq:latest
    ports:
      - "5341:5341"   # API
      - "5342:80"     # UI
    environment:
      ACCEPT_EULA: "Y"
    volumes:
      - seq-data:/data

  # =========================================
  # Metrics - Prometheus + Grafana
  # =========================================
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yaml:/etc/prometheus/prometheus.yml
      - ./monitoring/alerts.yaml:/etc/prometheus/alerts.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./monitoring/alertmanager.yaml:/etc/alertmanager/alertmanager.yml

  # =========================================
  # Tracing - Jaeger
  # =========================================
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI
      - "14250:14250"  # gRPC
      - "6831:6831/udp" # UDP
    environment:
      COLLECTOR_ZIPKIN_HOST_PORT: "9411"

  # =========================================
  # OpenTelemetry Collector
  # =========================================
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    ports:
      - "4317:4317"    # gRPC
      - "4318:4318"    # HTTP
      - "8888:8888"    # Prometheus metrics
    volumes:
      - ./monitoring/otel-collector.yaml:/etc/otelcol-contrib/config.yaml
    depends_on:
      - jaeger
      - prometheus

  # =========================================
  # Infrastructure
  # =========================================
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    ports:
      - "1433:1433"
    environment:
      SA_PASSWORD: "Pass123!"
      ACCEPT_EULA: "Y"
    volumes:
      - sqlserver-data:/var/opt/mssql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  seq-data:
  prometheus-data:
  grafana-data:
  sqlserver-data:
```

### OpenTelemetry Collector Configuration

```yaml
# monitoring/otel-collector.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024
  
  memory_limiter:
    limit_mib: 512
    spike_limit_mib: 128
    check_interval: 5s
  
  resource:
    attributes:
      - key: environment
        value: production
        action: upsert

exporters:
  # ส่ง traces ไป Jaeger
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
  
  # ส่ง metrics ไป Prometheus
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: mywebapi
  
  # ส่ง logs ไป Elasticsearch
  elasticsearch:
    endpoints: ["http://elasticsearch:9200"]
    index: otel-logs
  
  # Debug output
  logging:
    loglevel: info

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource]
      exporters: [jaeger, logging]
    
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
    
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [logging]
```

### Prometheus Configuration

```yaml
# monitoring/prometheus.yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'production'
    region: 'asia-southeast1'

rule_files:
  - "alerts.yaml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  # Scrape application metrics
  - job_name: 'mywebapi'
    static_configs:
      - targets: ['api:8080']
    metrics_path: '/metrics'
    scrape_interval: 10s
  
  # Scrape OTel Collector metrics
  - job_name: 'otel-collector'
    static_configs:
      - targets: ['otel-collector:8888', 'otel-collector:8889']
  
  # Scrape Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### Complete Controller with Observability

```csharp
// Controllers/OrdersController.cs
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private static readonly ActivitySource ActivitySource = 
        new ActivitySource("OrderManagementApi.Orders");
    
    private readonly IOrderService _orderService;
    private readonly ApplicationMetrics _appMetrics;
    private readonly BusinessMetrics _bizMetrics;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(
        IOrderService orderService,
        ApplicationMetrics appMetrics,
        BusinessMetrics bizMetrics,
        ILogger<OrdersController> logger)
    {
        _orderService = orderService;
        _appMetrics = appMetrics;
        _bizMetrics = bizMetrics;
        _logger = logger;
    }

    [HttpPost]
    public async Task<ActionResult<OrderResponse>> CreateOrder(
        [FromBody] CreateOrderRequest request)
    {
        using var activity = ActivitySource.StartActivity("CreateOrder");
        activity?.SetTag("customer.id", request.CustomerId);
        activity?.SetTag("order.amount", request.TotalAmount);

        using var logScope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["CustomerId"] = request.CustomerId,
            ["OrderAmount"] = request.TotalAmount
        });

        _logger.LogInformation("Received order creation request from customer {CustomerId}", 
            request.CustomerId);

        try
        {
            var order = await _orderService.CreateOrderAsync(request);
            
            // บันทึก business metrics
            _bizMetrics.RevenueTotal.Add(
                (double)order.TotalAmount,
                new TagList { { "currency", "THB" }, { "channel", "api" } });
            
            _bizMetrics.OrderValue.Record(
                (double)order.TotalAmount,
                new TagList { { "customer.tier", request.CustomerTier } });

            activity?.SetTag("order.id", order.Id.ToString());
            activity?.SetStatus(ActivityStatusCode.Ok);

            _logger.LogInformation(
                "Order {OrderId} created successfully for customer {CustomerId}",
                order.Id, request.CustomerId);

            return CreatedAtAction(
                nameof(GetOrder), 
                new { id = order.Id }, 
                MapToResponse(order));
        }
        catch (ValidationException ex)
        {
            _bizMetrics.ValidationErrors.Add(1, 
                new TagList { { "entity", "order" } });
            
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            
            _logger.LogWarning("Order validation failed: {ValidationErrors}",
                string.Join(", ", ex.Errors));
            
            return BadRequest(new ProblemDetails
            {
                Title = "Validation Failed",
                Detail = ex.Message,
                Status = StatusCodes.Status400BadRequest
            });
        }
        catch (PaymentDeclinedException ex)
        {
            _bizMetrics.PaymentDeclines.Add(1, 
                new TagList { { "decline.reason", ex.DeclineCode } });
            
            activity?.SetStatus(ActivityStatusCode.Error, "Payment declined");
            activity?.RecordException(ex);
            
            _logger.LogWarning(
                "Payment declined for customer {CustomerId}. Reason: {DeclineCode}",
                request.CustomerId, ex.DeclineCode);
            
            return UnprocessableEntity(new ProblemDetails
            {
                Title = "Payment Declined",
                Detail = ex.Message,
                Status = StatusCodes.Status422UnprocessableEntity
            });
        }
        catch (Exception ex)
        {
            _appMetrics.UnhandledErrors.Add(1, 
                new TagList { { "error.type", ex.GetType().Name } });
            
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            
            _logger.LogError(ex, 
                "Unhandled error creating order for customer {CustomerId}",
                request.CustomerId);
            
            throw;
        }
    }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<OrderResponse>> GetOrder(int id)
    {
        using var activity = ActivitySource.StartActivity("GetOrder");
        activity?.SetTag("order.id", id);

        var order = await _orderService.GetByIdAsync(id);
        
        if (order is null)
        {
            activity?.SetStatus(ActivityStatusCode.Error, "Order not found");
            return NotFound();
        }
        
        activity?.SetStatus(ActivityStatusCode.Ok);
        return Ok(MapToResponse(order));
    }

    private static OrderResponse MapToResponse(Order order) =>
        new(order.Id, order.CustomerId, order.TotalAmount, order.Status, order.CreatedAt);
}
```

### Testing Observability

```csharp
// Tests/ObservabilityTests.cs
public class ObservabilityTests
{
    [Fact]
    public async Task CreateOrder_ShouldEmitCorrectMetrics()
    {
        // Arrange
        var meterListener = new MeterListener();
        var measurements = new List<(string Name, double Value, KeyValuePair<string, object?>[] Tags)>();
        
        meterListener.InstrumentPublished = (instrument, listener) =>
        {
            if (instrument.Meter.Name == "OrderManagementApi.Business")
                listener.EnableMeasurementEvents(instrument);
        };
        
        meterListener.SetMeasurementEventCallback<double>((instrument, measurement, tags, _) =>
        {
            measurements.Add((instrument.Name, measurement, tags.ToArray()));
        });
        
        meterListener.Start();
        
        // Act
        using var app = CreateTestApp();
        var client = app.CreateClient();
        
        var response = await client.PostAsJsonAsync("/api/orders", new CreateOrderRequest
        {
            CustomerId = "test-customer",
            TotalAmount = 1000m,
            CustomerTier = "premium"
        });
        
        // Assert
        response.EnsureSuccessStatusCode();
        
        var revenueMetric = measurements.FirstOrDefault(m => m.Name == "business.revenue.total");
        Assert.Equal(1000, revenueMetric.Value);
        
        meterListener.Dispose();
    }

    [Fact]
    public async Task CreateOrder_ShouldCreateTraceSpan()
    {
        // Arrange
        var activities = new List<Activity>();
        
        using var listener = new ActivityListener
        {
            ShouldListenTo = source => source.Name == "OrderManagementApi.Orders",
            Sample = (ref ActivityCreationOptions<ActivityContext> _) => ActivitySamplingResult.AllData,
            ActivityStopped = activity => activities.Add(activity)
        };
        
        ActivitySource.AddActivityListener(listener);
        
        // Act
        using var app = CreateTestApp();
        var client = app.CreateClient();
        await client.PostAsJsonAsync("/api/orders", CreateValidOrderRequest());
        
        // Assert
        var createOrderActivity = activities.FirstOrDefault(a => a.OperationName == "CreateOrder");
        Assert.NotNull(createOrderActivity);
        Assert.Equal(ActivityStatusCode.Ok, createOrderActivity.Status);
        Assert.NotNull(createOrderActivity.GetTagItem("customer.id"));
    }
}
```

---

## สรุป

ในส่วนนี้เราได้เรียนรู้การสร้าง Observability ที่ครบวงจรสำหรับ ASP.NET Core:

| หัวข้อ | เครื่องมือที่ใช้ | วัตถุประสงค์ |
|--------|---------------|------------|
| **Logs** | Serilog + Seq/ELK | บันทึกเหตุการณ์ด้วย structure |
| **Metrics** | OpenTelemetry + Prometheus + Grafana | วัดประสิทธิภาพเชิงตัวเลข |
| **Traces** | OpenTelemetry + Jaeger | ติดตาม request ระหว่าง services |
| **Health Checks** | AspNetCore.HealthChecks UI | แสดงสถานะของ dependencies |
| **Alerting** | Prometheus AlertManager | แจ้งเตือนเมื่อมีปัญหา |

### Best Practices

1. **เริ่มต้นด้วย USE method** - Utilization, Saturation, Errors สำหรับ infrastructure
2. **ใช้ RED method** - Rate, Errors, Duration สำหรับ API services
3. **Structured logging เสมอ** - ใช้ properties แทน string concatenation
4. **ตั้ง SLO (Service Level Objectives)** ก่อน แล้วค่อย alert ตาม SLO
5. **Correlation ID ทุก request** - ทำให้ trace ระหว่าง services ง่ายขึ้น
6. **Sample traces อย่างเหมาะสม** - ใน production อาจ sample 10-20%
7. **เก็บ business metrics ด้วย** ไม่ใช่แค่ technical metrics

---

**ก่อนหน้า → [Part 70: Caching](../part61-70/part70-caching.md)**
**ต่อไป → [Part 72: Resilience Patterns](part72-resilience.md)**
