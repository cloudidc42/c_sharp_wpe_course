# Part 68: gRPC Services ใน C#/.NET

## ภาพรวม

gRPC (Google Remote Procedure Call) คือ framework สำหรับ Remote Procedure Call ประสิทธิภาพสูงที่พัฒนาโดย Google โดยใช้ HTTP/2 เป็น transport layer และ Protocol Buffers เป็น interface definition language ทำให้การสื่อสารระหว่าง services เร็วและมีประสิทธิภาพสูงกว่า REST API แบบดั้งเดิม

ใน Part นี้จะครอบคลุม Steps 671-680 ซึ่งจะพาไปเรียนรู้ gRPC ตั้งแต่พื้นฐานจนถึงการสร้าง service จริง

---

## Step 671: gRPC Overview, Protocol Buffers และการตั้งค่า Server

### ทำความรู้จัก gRPC

gRPC มีข้อดีเหนือ REST API หลายอย่าง:

- **ประสิทธิภาพสูง**: ใช้ HTTP/2 ที่รองรับ multiplexing และ header compression
- **Type-safe**: มี contract ที่ชัดเจนผ่าน `.proto` files
- **รองรับ streaming**: ทั้ง server streaming, client streaming และ bidirectional streaming
- **Multi-language**: สร้าง client/server ใน C#, Go, Python, Java ฯลฯ จาก `.proto` เดียวกัน
- **Strongly typed**: ลด bug ที่เกิดจาก serialization/deserialization

### Protocol Buffers (Protobuf)

Protocol Buffers คือ language-neutral, platform-neutral mechanism สำหรับ serializing structured data เร็วกว่า JSON และ XML มาก

#### สร้าง .proto file แรก

```protobuf
// product.proto
syntax = "proto3";

option csharp_namespace = "GrpcProductService";

package product;

// ประกาศ service
service ProductService {
  // Unary RPC
  rpc GetProduct (GetProductRequest) returns (ProductResponse);
  rpc CreateProduct (CreateProductRequest) returns (ProductResponse);
  rpc UpdateProduct (UpdateProductRequest) returns (ProductResponse);
  rpc DeleteProduct (DeleteProductRequest) returns (DeleteProductResponse);
  
  // Server streaming
  rpc ListProducts (ListProductsRequest) returns (stream ProductResponse);
  
  // Client streaming
  rpc BulkCreateProducts (stream CreateProductRequest) returns (BulkCreateResponse);
  
  // Bidirectional streaming
  rpc StreamProductUpdates (stream ProductUpdateRequest) returns (stream ProductUpdateResponse);
}

// Messages
message GetProductRequest {
  string product_id = 1;
}

message CreateProductRequest {
  string name = 1;
  string description = 2;
  double price = 3;
  int32 stock_quantity = 4;
  string category = 5;
}

message UpdateProductRequest {
  string product_id = 1;
  string name = 2;
  double price = 3;
  int32 stock_quantity = 4;
}

message DeleteProductRequest {
  string product_id = 1;
}

message DeleteProductResponse {
  bool success = 1;
  string message = 2;
}

message ProductResponse {
  string product_id = 1;
  string name = 2;
  string description = 3;
  double price = 4;
  int32 stock_quantity = 5;
  string category = 6;
  google.protobuf.Timestamp created_at = 7;
  google.protobuf.Timestamp updated_at = 8;
}

message ListProductsRequest {
  string category = 1;
  int32 page_size = 2;
  string page_token = 3;
}

message BulkCreateResponse {
  int32 created_count = 1;
  repeated string product_ids = 2;
  repeated string failed_items = 3;
}

message ProductUpdateRequest {
  string product_id = 1;
  int32 quantity_change = 2;
}

message ProductUpdateResponse {
  string product_id = 1;
  int32 new_quantity = 2;
  bool success = 3;
}
```

### ตั้งค่า gRPC Server Project

```bash
# สร้าง project ใหม่
dotnet new grpc -n GrpcProductService
cd GrpcProductService

# หรือเพิ่ม gRPC ใน project ที่มีอยู่
dotnet add package Grpc.AspNetCore
dotnet add package Google.Protobuf
dotnet add package Grpc.Tools
```

#### ตั้งค่า .csproj

```xml
<!-- GrpcProductService.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <!-- กำหนด .proto files และ role ของ project (Server) -->
    <Protobuf Include="Protos\product.proto" GrpcServices="Server" />
    <Protobuf Include="Protos\order.proto" GrpcServices="Server" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Grpc.AspNetCore" Version="2.57.0" />
    <PackageReference Include="Google.Protobuf" Version="3.25.0" />
    <PackageReference Include="Grpc.Tools" Version="2.57.0">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
  </ItemGroup>
</Project>
```

#### ตั้งค่า Program.cs

```csharp
// Program.cs
using GrpcProductService.Services;
using GrpcProductService.Interceptors;

var builder = WebApplication.CreateBuilder(args);

// เพิ่ม gRPC services
builder.Services.AddGrpc(options =>
{
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
    options.MaxReceiveMessageSize = 16 * 1024 * 1024; // 16 MB
    options.MaxSendMessageSize = 16 * 1024 * 1024;    // 16 MB
    options.Interceptors.Add<LoggingInterceptor>();
    options.Interceptors.Add<AuthenticationInterceptor>();
});

// เพิ่ม gRPC Reflection (สำหรับ development)
builder.Services.AddGrpcReflection();

// เพิ่ม services อื่น ๆ
builder.Services.AddSingleton<IProductRepository, InMemoryProductRepository>();

var app = builder.Build();

// Map gRPC services
app.MapGrpcService<GrpcProductServiceImpl>();
app.MapGrpcService<GrpcOrderService>();

// Enable reflection ใน development
if (app.Environment.IsDevelopment())
{
    app.MapGrpcReflectionService();
}

// Health check endpoint
app.MapGet("/", () => "gRPC Product Service is running. Use gRPC client to connect.");

app.Run();
```

#### appsettings.json สำหรับ gRPC

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Grpc": "Debug"
    }
  },
  "AllowedHosts": "*",
  "Kestrel": {
    "Endpoints": {
      "Grpc": {
        "Url": "https://localhost:7042",
        "Protocols": "Http2"
      },
      "Http": {
        "Url": "http://localhost:5042",
        "Protocols": "Http1AndHttp2"
      }
    }
  }
}
```

---

## Step 672: Unary RPC (Request/Response)

### ทำความเข้าใจ Unary RPC

Unary RPC คือรูปแบบที่ง่ายที่สุด เหมือน HTTP request/response ปกติ client ส่ง request หนึ่งอัน และ server ตอบกลับหนึ่งอัน

### สร้าง Service Implementation

```csharp
// Services/GrpcProductServiceImpl.cs
using Grpc.Core;
using GrpcProductService.Repositories;
using Google.Protobuf.WellKnownTypes;

namespace GrpcProductService.Services;

public class GrpcProductServiceImpl : ProductService.ProductServiceBase
{
    private readonly IProductRepository _repository;
    private readonly ILogger<GrpcProductServiceImpl> _logger;

    public GrpcProductServiceImpl(
        IProductRepository repository,
        ILogger<GrpcProductServiceImpl> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    // Unary RPC: ดึงสินค้าตาม ID
    public override async Task<ProductResponse> GetProduct(
        GetProductRequest request,
        ServerCallContext context)
    {
        _logger.LogInformation("GetProduct called with ID: {ProductId}", request.ProductId);

        // ตรวจสอบ input
        if (string.IsNullOrEmpty(request.ProductId))
        {
            throw new RpcException(new Status(
                StatusCode.InvalidArgument,
                "Product ID cannot be empty"));
        }

        var product = await _repository.GetByIdAsync(request.ProductId);

        if (product == null)
        {
            throw new RpcException(new Status(
                StatusCode.NotFound,
                $"Product with ID '{request.ProductId}' was not found"));
        }

        return MapToResponse(product);
    }

    // Unary RPC: สร้างสินค้าใหม่
    public override async Task<ProductResponse> CreateProduct(
        CreateProductRequest request,
        ServerCallContext context)
    {
        _logger.LogInformation("CreateProduct called: {ProductName}", request.Name);

        // Validation
        if (string.IsNullOrEmpty(request.Name))
        {
            throw new RpcException(new Status(
                StatusCode.InvalidArgument,
                "Product name is required"));
        }

        if (request.Price <= 0)
        {
            throw new RpcException(new Status(
                StatusCode.InvalidArgument,
                "Price must be greater than 0"));
        }

        var product = new ProductEntity
        {
            Id = Guid.NewGuid().ToString(),
            Name = request.Name,
            Description = request.Description,
            Price = request.Price,
            StockQuantity = request.StockQuantity,
            Category = request.Category,
            CreatedAt = DateTime.UtcNow,
            UpdatedAt = DateTime.UtcNow
        };

        try
        {
            await _repository.CreateAsync(product);
            _logger.LogInformation("Product created successfully: {ProductId}", product.Id);
            return MapToResponse(product);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating product");
            throw new RpcException(new Status(
                StatusCode.Internal,
                "An error occurred while creating the product"));
        }
    }

    // Unary RPC: อัปเดตสินค้า
    public override async Task<ProductResponse> UpdateProduct(
        UpdateProductRequest request,
        ServerCallContext context)
    {
        var existing = await _repository.GetByIdAsync(request.ProductId)
            ?? throw new RpcException(new Status(
                StatusCode.NotFound,
                $"Product '{request.ProductId}' not found"));

        existing.Name = request.Name.Length > 0 ? request.Name : existing.Name;
        existing.Price = request.Price > 0 ? request.Price : existing.Price;
        existing.StockQuantity = request.StockQuantity >= 0
            ? request.StockQuantity
            : existing.StockQuantity;
        existing.UpdatedAt = DateTime.UtcNow;

        await _repository.UpdateAsync(existing);
        return MapToResponse(existing);
    }

    // Unary RPC: ลบสินค้า
    public override async Task<DeleteProductResponse> DeleteProduct(
        DeleteProductRequest request,
        ServerCallContext context)
    {
        var deleted = await _repository.DeleteAsync(request.ProductId);

        if (!deleted)
        {
            throw new RpcException(new Status(
                StatusCode.NotFound,
                $"Product '{request.ProductId}' not found"));
        }

        return new DeleteProductResponse
        {
            Success = true,
            Message = $"Product '{request.ProductId}' deleted successfully"
        };
    }

    private static ProductResponse MapToResponse(ProductEntity product) => new()
    {
        ProductId = product.Id,
        Name = product.Name,
        Description = product.Description,
        Price = product.Price,
        StockQuantity = product.StockQuantity,
        Category = product.Category,
        CreatedAt = Timestamp.FromDateTime(product.CreatedAt),
        UpdatedAt = Timestamp.FromDateTime(product.UpdatedAt)
    };
}
```

### gRPC Client สำหรับ Unary RPC

```csharp
// GrpcProductClient/Program.cs
using Grpc.Core;
using Grpc.Net.Client;
using GrpcProductService;

// สร้าง channel เชื่อมต่อ server
using var channel = GrpcChannel.ForAddress("https://localhost:7042");
var client = new ProductService.ProductServiceClient(channel);

// ==================
// เรียก Unary RPC
// ==================

// 1. สร้างสินค้า
Console.WriteLine("=== Creating Product ===");
try
{
    var createRequest = new CreateProductRequest
    {
        Name = "Laptop Pro X1",
        Description = "High-performance laptop for professionals",
        Price = 45000.00,
        StockQuantity = 100,
        Category = "Electronics"
    };

    var created = await client.CreateProductAsync(createRequest);
    Console.WriteLine($"Created: {created.ProductId} - {created.Name} @ {created.Price:C}");

    // 2. ดึงสินค้า
    Console.WriteLine("\n=== Getting Product ===");
    var getRequest = new GetProductRequest { ProductId = created.ProductId };
    var product = await client.GetProductAsync(getRequest);
    Console.WriteLine($"Got: {product.Name} (Stock: {product.StockQuantity})");

    // 3. อัปเดตราคา
    Console.WriteLine("\n=== Updating Product ===");
    var updateRequest = new UpdateProductRequest
    {
        ProductId = created.ProductId,
        Price = 42000.00,
        StockQuantity = 95
    };
    var updated = await client.UpdateProductAsync(updateRequest);
    Console.WriteLine($"Updated price: {updated.Price:C}");

    // 4. ลบสินค้า
    Console.WriteLine("\n=== Deleting Product ===");
    var deleteRequest = new DeleteProductRequest { ProductId = created.ProductId };
    var deleteResult = await client.DeleteProductAsync(deleteRequest);
    Console.WriteLine($"Delete result: {deleteResult.Message}");
}
catch (RpcException ex) when (ex.StatusCode == StatusCode.NotFound)
{
    Console.WriteLine($"Product not found: {ex.Status.Detail}");
}
catch (RpcException ex) when (ex.StatusCode == StatusCode.InvalidArgument)
{
    Console.WriteLine($"Invalid request: {ex.Status.Detail}");
}
catch (RpcException ex)
{
    Console.WriteLine($"gRPC Error [{ex.StatusCode}]: {ex.Status.Detail}");
}

// การตั้ง Deadline (timeout)
Console.WriteLine("\n=== With Deadline ===");
try
{
    var deadline = DateTime.UtcNow.AddSeconds(5); // timeout 5 วินาที
    var product = await client.GetProductAsync(
        new GetProductRequest { ProductId = "some-id" },
        deadline: deadline);
    Console.WriteLine($"Got product: {product.Name}");
}
catch (RpcException ex) when (ex.StatusCode == StatusCode.DeadlineExceeded)
{
    Console.WriteLine("Request timed out!");
}

// การยกเลิก request
Console.WriteLine("\n=== With Cancellation ===");
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(3));
try
{
    var product = await client.GetProductAsync(
        new GetProductRequest { ProductId = "some-id" },
        cancellationToken: cts.Token);
    Console.WriteLine($"Got product: {product.Name}");
}
catch (RpcException ex) when (ex.StatusCode == StatusCode.Cancelled)
{
    Console.WriteLine("Request was cancelled.");
}
```

---

## Step 673: Server Streaming RPC

### ทำความเข้าใจ Server Streaming

Server Streaming RPC คือ client ส่ง request เดียว แต่ server ส่งกลับ stream ของ responses หลายอัน เหมาะสำหรับการดึงข้อมูลจำนวนมาก หรือส่งข้อมูล real-time

### Server Implementation

```csharp
// ใน GrpcProductServiceImpl.cs เพิ่ม method นี้

// Server Streaming RPC: ส่งรายการสินค้าแบบ stream
public override async Task ListProducts(
    ListProductsRequest request,
    IServerStreamWriter<ProductResponse> responseStream,
    ServerCallContext context)
{
    _logger.LogInformation(
        "ListProducts stream started - Category: {Category}",
        request.Category);

    var products = await _repository.GetByCategoryAsync(
        request.Category,
        request.PageSize > 0 ? request.PageSize : 100);

    var batchNumber = 0;
    foreach (var product in products)
    {
        // ตรวจสอบว่า client ยังเชื่อมต่ออยู่
        if (context.CancellationToken.IsCancellationRequested)
        {
            _logger.LogInformation("Client cancelled the stream");
            break;
        }

        // ส่งแต่ละ product
        await responseStream.WriteAsync(MapToResponse(product));

        batchNumber++;
        _logger.LogDebug("Sent product {BatchNumber}: {ProductId}", batchNumber, product.Id);

        // Simulate processing delay
        await Task.Delay(10, context.CancellationToken);
    }

    _logger.LogInformation(
        "ListProducts stream completed - Sent {Count} products", batchNumber);
}

// Server Streaming RPC: ส่งการแจ้งเตือนราคาสินค้าแบบ real-time
public override async Task WatchPriceChanges(
    WatchPriceRequest request,
    IServerStreamWriter<PriceChangeNotification> responseStream,
    ServerCallContext context)
{
    _logger.LogInformation("Starting price watch for {Count} products", request.ProductIds.Count);

    // ลงทะเบียนรับการเปลี่ยนแปลง
    var channel = _priceChangeService.Subscribe(request.ProductIds.ToList());

    try
    {
        await foreach (var change in channel.ReadAllAsync(context.CancellationToken))
        {
            await responseStream.WriteAsync(new PriceChangeNotification
            {
                ProductId = change.ProductId,
                OldPrice = change.OldPrice,
                NewPrice = change.NewPrice,
                ChangedAt = Timestamp.FromDateTime(change.ChangedAt)
            });
        }
    }
    catch (OperationCanceledException)
    {
        _logger.LogInformation("Price watch stream cancelled by client");
    }
    finally
    {
        _priceChangeService.Unsubscribe(channel);
    }
}
```

### Client ที่รับ Server Streaming

```csharp
// Client อ่าน server streaming
Console.WriteLine("=== Server Streaming: List Products ===");

var listRequest = new ListProductsRequest
{
    Category = "Electronics",
    PageSize = 50
};

using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));

// รับ stream จาก server
using var call = client.ListProducts(listRequest, cancellationToken: cts.Token);

var productCount = 0;
await foreach (var product in call.ResponseStream.ReadAllAsync(cts.Token))
{
    productCount++;
    Console.WriteLine($"  [{productCount}] {product.Name} - {product.Price:C} (Stock: {product.StockQuantity})");
}

Console.WriteLine($"Total products received: {productCount}");

// ==========================================
// Real-time price watch streaming
// ==========================================
Console.WriteLine("\n=== Real-time Price Watch ===");

var watchRequest = new WatchPriceRequest();
watchRequest.ProductIds.AddRange(new[] { "prod-001", "prod-002", "prod-003" });

using var watchCts = new CancellationTokenSource();

// รันใน background task
var watchTask = Task.Run(async () =>
{
    using var watchCall = client.WatchPriceChanges(watchRequest, cancellationToken: watchCts.Token);
    
    try
    {
        await foreach (var notification in watchCall.ResponseStream.ReadAllAsync(watchCts.Token))
        {
            Console.WriteLine($"  Price change: {notification.ProductId} " +
                $"{notification.OldPrice:C} -> {notification.NewPrice:C}");
        }
    }
    catch (RpcException ex) when (ex.StatusCode == StatusCode.Cancelled)
    {
        Console.WriteLine("Price watch cancelled.");
    }
});

// รอ 10 วินาทีแล้วยกเลิก
await Task.Delay(10000);
watchCts.Cancel();
await watchTask;
```

---

## Step 674: Client Streaming RPC

### ทำความเข้าใจ Client Streaming

Client Streaming RPC คือ client ส่ง stream ของ requests หลายอัน แล้ว server ตอบกลับครั้งเดียวหลังจากได้รับทั้งหมด เหมาะสำหรับ batch operations หรือการอัปโหลดข้อมูลจำนวนมาก

### Server Implementation

```csharp
// ใน GrpcProductServiceImpl.cs

// Client Streaming RPC: รับสินค้าหลายรายการในครั้งเดียว
public override async Task<BulkCreateResponse> BulkCreateProducts(
    IAsyncStreamReader<CreateProductRequest> requestStream,
    ServerCallContext context)
{
    _logger.LogInformation("BulkCreateProducts stream started");

    var createdIds = new List<string>();
    var failedItems = new List<string>();
    var processedCount = 0;

    await foreach (var request in requestStream.ReadAllAsync(context.CancellationToken))
    {
        processedCount++;

        try
        {
            // Validate
            if (string.IsNullOrEmpty(request.Name) || request.Price <= 0)
            {
                failedItems.Add($"Item {processedCount}: Invalid data (Name: {request.Name})");
                continue;
            }

            var product = new ProductEntity
            {
                Id = Guid.NewGuid().ToString(),
                Name = request.Name,
                Description = request.Description,
                Price = request.Price,
                StockQuantity = request.StockQuantity,
                Category = request.Category,
                CreatedAt = DateTime.UtcNow,
                UpdatedAt = DateTime.UtcNow
            };

            await _repository.CreateAsync(product);
            createdIds.Add(product.Id);

            _logger.LogDebug("Created product {Count}: {Name}", processedCount, product.Name);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating product {Count}", processedCount);
            failedItems.Add($"Item {processedCount}: {ex.Message}");
        }
    }

    _logger.LogInformation(
        "BulkCreate completed: {Created} created, {Failed} failed",
        createdIds.Count, failedItems.Count);

    var response = new BulkCreateResponse
    {
        CreatedCount = createdIds.Count
    };
    response.ProductIds.AddRange(createdIds);
    response.FailedItems.AddRange(failedItems);

    return response;
}

// Client Streaming RPC: อัปโหลด CSV data
public override async Task<ImportProductsResponse> ImportProductsFromCsv(
    IAsyncStreamReader<CsvChunkRequest> requestStream,
    ServerCallContext context)
{
    var csvContent = new System.Text.StringBuilder();
    var totalBytes = 0;

    // รับ chunks ทั้งหมด
    await foreach (var chunk in requestStream.ReadAllAsync(context.CancellationToken))
    {
        csvContent.Append(System.Text.Encoding.UTF8.GetString(chunk.Data.ToByteArray()));
        totalBytes += chunk.Data.Length;

        _logger.LogDebug("Received chunk {ChunkNumber}, total bytes: {Total}",
            chunk.ChunkNumber, totalBytes);
    }

    // Process CSV
    var lines = csvContent.ToString().Split('\n', StringSplitOptions.RemoveEmptyEntries);
    var createdCount = 0;
    var errors = new List<string>();

    foreach (var line in lines.Skip(1)) // Skip header
    {
        var parts = line.Split(',');
        if (parts.Length < 4)
        {
            errors.Add($"Invalid line: {line}");
            continue;
        }

        try
        {
            var product = new ProductEntity
            {
                Id = Guid.NewGuid().ToString(),
                Name = parts[0].Trim(),
                Price = double.Parse(parts[1].Trim()),
                StockQuantity = int.Parse(parts[2].Trim()),
                Category = parts[3].Trim(),
                CreatedAt = DateTime.UtcNow,
                UpdatedAt = DateTime.UtcNow
            };
            await _repository.CreateAsync(product);
            createdCount++;
        }
        catch (Exception ex)
        {
            errors.Add($"Error processing line: {ex.Message}");
        }
    }

    return new ImportProductsResponse
    {
        ImportedCount = createdCount,
        TotalBytes = totalBytes
    };
}
```

### Client Implementation สำหรับ Client Streaming

```csharp
// Client ส่ง client streaming
Console.WriteLine("=== Client Streaming: Bulk Create Products ===");

// สร้าง streaming call
using var bulkCall = client.BulkCreateProducts();

// สร้างสินค้าจำลองและส่งแบบ stream
var productsToCreate = GenerateSampleProducts(1000); // 1000 สินค้า
var sentCount = 0;

foreach (var product in productsToCreate)
{
    await bulkCall.RequestStream.WriteAsync(new CreateProductRequest
    {
        Name = product.Name,
        Description = product.Description,
        Price = product.Price,
        StockQuantity = product.Stock,
        Category = product.Category
    });

    sentCount++;
    if (sentCount % 100 == 0)
    {
        Console.WriteLine($"  Sent {sentCount} products...");
    }
}

// บอก server ว่าส่งเสร็จแล้ว
await bulkCall.RequestStream.CompleteAsync();

// รอรับ response
var result = await bulkCall.ResponseAsync;

Console.WriteLine($"\nBulk create result:");
Console.WriteLine($"  Created: {result.CreatedCount}");
Console.WriteLine($"  Failed: {result.FailedItems.Count}");
if (result.FailedItems.Any())
{
    Console.WriteLine("  Failed items:");
    foreach (var failed in result.FailedItems.Take(5))
    {
        Console.WriteLine($"    - {failed}");
    }
}

// ==========================================
// อัปโหลดไฟล์ CSV แบบ streaming (chunks)
// ==========================================
Console.WriteLine("\n=== Upload CSV File as Stream ===");

const int chunkSize = 64 * 1024; // 64 KB per chunk
var csvFilePath = "products.csv";
var fileBytes = await File.ReadAllBytesAsync(csvFilePath);

using var importCall = client.ImportProductsFromCsv();

for (int i = 0; i < fileBytes.Length; i += chunkSize)
{
    var chunk = fileBytes.Skip(i).Take(chunkSize).ToArray();
    var chunkNumber = (i / chunkSize) + 1;

    await importCall.RequestStream.WriteAsync(new CsvChunkRequest
    {
        ChunkNumber = chunkNumber,
        Data = Google.Protobuf.ByteString.CopyFrom(chunk)
    });

    Console.WriteLine($"Uploaded chunk {chunkNumber} ({chunk.Length} bytes)");
}

await importCall.RequestStream.CompleteAsync();
var importResult = await importCall.ResponseAsync;

Console.WriteLine($"Import complete: {importResult.ImportedCount} products, {importResult.TotalBytes} bytes");

// Helper function
static IEnumerable<(string Name, string Description, double Price, int Stock, string Category)> GenerateSampleProducts(int count)
{
    var categories = new[] { "Electronics", "Clothing", "Food", "Books", "Sports" };
    var rng = new Random();

    for (int i = 1; i <= count; i++)
    {
        yield return (
            Name: $"Product {i:D4}",
            Description: $"Description for product {i}",
            Price: rng.NextDouble() * 10000 + 100,
            Stock: rng.Next(0, 500),
            Category: categories[rng.Next(categories.Length)]
        );
    }
}
```

---

## Step 675: Bidirectional Streaming RPC

### ทำความเข้าใจ Bidirectional Streaming

Bidirectional Streaming คือทั้ง client และ server สามารถส่ง stream ของ messages ได้พร้อมกัน เหมาะสำหรับ real-time communication เช่น chat, live inventory updates

### Proto Definition สำหรับ Bidirectional

```protobuf
// เพิ่มใน product.proto

service ProductService {
  // ... (RPCs อื่น ๆ) ...
  
  // Bidirectional streaming: อัปเดต stock แบบ real-time
  rpc StreamProductUpdates (stream ProductUpdateRequest) returns (stream ProductUpdateResponse);
  
  // Bidirectional streaming: chat/consultation
  rpc ProductConsultation (stream ConsultationMessage) returns (stream ConsultationMessage);
}

message ProductUpdateRequest {
  string product_id = 1;
  int32 quantity_change = 2;
  string update_type = 3; // "sale", "restock", "adjustment"
}

message ProductUpdateResponse {
  string product_id = 1;
  int32 old_quantity = 2;
  int32 new_quantity = 3;
  bool success = 4;
  string error_message = 5;
  google.protobuf.Timestamp updated_at = 6;
}

message ConsultationMessage {
  string session_id = 1;
  string sender = 2;
  string message = 3;
  google.protobuf.Timestamp sent_at = 4;
}
```

### Server Implementation

```csharp
// Bidirectional Streaming: อัปเดต stock สินค้า
public override async Task StreamProductUpdates(
    IAsyncStreamReader<ProductUpdateRequest> requestStream,
    IServerStreamWriter<ProductUpdateResponse> responseStream,
    ServerCallContext context)
{
    _logger.LogInformation("Bidirectional stream started: StreamProductUpdates");

    // อ่าน requests และส่ง responses พร้อมกัน
    await foreach (var request in requestStream.ReadAllAsync(context.CancellationToken))
    {
        _logger.LogDebug("Processing update for product: {ProductId}", request.ProductId);

        ProductUpdateResponse response;

        try
        {
            var product = await _repository.GetByIdAsync(request.ProductId);

            if (product == null)
            {
                response = new ProductUpdateResponse
                {
                    ProductId = request.ProductId,
                    Success = false,
                    ErrorMessage = $"Product '{request.ProductId}' not found"
                };
            }
            else
            {
                var oldQuantity = product.StockQuantity;
                product.StockQuantity = Math.Max(0, product.StockQuantity + request.QuantityChange);
                product.UpdatedAt = DateTime.UtcNow;

                await _repository.UpdateAsync(product);

                response = new ProductUpdateResponse
                {
                    ProductId = product.Id,
                    OldQuantity = oldQuantity,
                    NewQuantity = product.StockQuantity,
                    Success = true,
                    UpdatedAt = Timestamp.FromDateTime(product.UpdatedAt)
                };

                // แจ้งเตือนถ้า stock ต่ำ
                if (product.StockQuantity < 10)
                {
                    _logger.LogWarning(
                        "Low stock alert: {ProductId} = {Quantity}",
                        product.Id, product.StockQuantity);
                }
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error updating product {ProductId}", request.ProductId);
            response = new ProductUpdateResponse
            {
                ProductId = request.ProductId,
                Success = false,
                ErrorMessage = ex.Message
            };
        }

        // ส่ง response กลับ
        await responseStream.WriteAsync(response);
    }

    _logger.LogInformation("StreamProductUpdates completed");
}

// Bidirectional Streaming: Chat consultation
public override async Task ProductConsultation(
    IAsyncStreamReader<ConsultationMessage> requestStream,
    IServerStreamWriter<ConsultationMessage> responseStream,
    ServerCallContext context)
{
    var sessionId = context.GetHttpContext().Request.Headers["session-id"].ToString();
    _logger.LogInformation("Consultation session started: {SessionId}", sessionId);

    await foreach (var message in requestStream.ReadAllAsync(context.CancellationToken))
    {
        _logger.LogInformation("Received: [{Sender}] {Message}", message.Sender, message.Message);

        // สร้าง response อัตโนมัติ (ในระบบจริงอาจเชื่อมกับ AI หรือ human agent)
        var responseMessage = await GenerateConsultationResponse(message);
        await responseStream.WriteAsync(responseMessage);
    }
}

private async Task<ConsultationMessage> GenerateConsultationResponse(ConsultationMessage message)
{
    await Task.Delay(100); // Simulate processing

    var response = "ขอโทษที่ไม่เข้าใจคำถาม กรุณาติดต่อเจ้าหน้าที่";

    if (message.Message.Contains("price", StringComparison.OrdinalIgnoreCase))
        response = "ราคาสินค้าของเราแข่งขันได้และมีโปรโมชั่นประจำเดือน";
    else if (message.Message.Contains("stock", StringComparison.OrdinalIgnoreCase))
        response = "สามารถตรวจสอบสต็อกสินค้าแบบ real-time ได้ผ่านระบบ";
    else if (message.Message.Contains("delivery", StringComparison.OrdinalIgnoreCase))
        response = "เราจัดส่งภายใน 2-3 วันทำการทั่วประเทศ";

    return new ConsultationMessage
    {
        SessionId = message.SessionId,
        Sender = "system",
        Message = response,
        SentAt = Timestamp.FromDateTime(DateTime.UtcNow)
    };
}
```

### Client Implementation

```csharp
// Bidirectional Streaming Client
Console.WriteLine("=== Bidirectional Streaming: Stock Updates ===");

// สร้าง list ของการอัปเดต
var updates = new List<(string Id, int Change, string Type)>
{
    ("prod-001", -5, "sale"),
    ("prod-002", -10, "sale"),
    ("prod-001", 50, "restock"),
    ("prod-003", -3, "sale"),
    ("prod-004", 100, "restock"),
};

using var updateCall = client.StreamProductUpdates();

// ส่ง requests ใน background
var sendTask = Task.Run(async () =>
{
    foreach (var (id, change, type) in updates)
    {
        await updateCall.RequestStream.WriteAsync(new ProductUpdateRequest
        {
            ProductId = id,
            QuantityChange = change,
            UpdateType = type
        });

        Console.WriteLine($"  Sent update: {id} ({(change >= 0 ? "+" : "")}{change}) [{type}]");
        await Task.Delay(200); // ส่งทีละอัน
    }

    // บอก server ว่าส่งเสร็จแล้ว
    await updateCall.RequestStream.CompleteAsync();
    Console.WriteLine("  All updates sent.");
});

// รับ responses
var receiveTask = Task.Run(async () =>
{
    await foreach (var response in updateCall.ResponseStream.ReadAllAsync())
    {
        if (response.Success)
        {
            Console.WriteLine($"  Updated: {response.ProductId} " +
                $"{response.OldQuantity} -> {response.NewQuantity}");
        }
        else
        {
            Console.WriteLine($"  Failed: {response.ProductId} - {response.ErrorMessage}");
        }
    }
});

// รอทั้ง 2 tasks
await Task.WhenAll(sendTask, receiveTask);
Console.WriteLine("Bidirectional streaming completed!");

// ==========================================
// Bidirectional Streaming: Chat
// ==========================================
Console.WriteLine("\n=== Bidirectional Streaming: Product Consultation ===");

var sessionId = Guid.NewGuid().ToString();
using var chatCall = client.ProductConsultation(
    new Metadata { { "session-id", sessionId } });

var chatMessages = new[]
{
    "What is the price of Laptop Pro X1?",
    "Is it in stock?",
    "How long is the delivery time?"
};

// ส่งและรับ messages สลับกัน
foreach (var msg in chatMessages)
{
    await chatCall.RequestStream.WriteAsync(new ConsultationMessage
    {
        SessionId = sessionId,
        Sender = "customer",
        Message = msg,
        SentAt = Timestamp.FromDateTime(DateTime.UtcNow)
    });
    Console.WriteLine($"  Customer: {msg}");

    if (await chatCall.ResponseStream.MoveNext())
    {
        var response = chatCall.ResponseStream.Current;
        Console.WriteLine($"  System: {response.Message}");
    }
}

await chatCall.RequestStream.CompleteAsync();
```

---

## Step 676: gRPC Client Factory กับ HttpClientFactory

### ทำไมต้องใช้ Client Factory

gRPC client factory ช่วยในการจัดการ channel lifecycle, retry policies, และ load balancing อย่างถูกต้อง แทนที่จะสร้าง channel ด้วยตัวเองทุกครั้ง

### ตั้งค่า Client Factory

```csharp
// Program.cs ฝั่ง client application
var builder = WebApplication.CreateBuilder(args);

// เพิ่ม gRPC client factory
builder.Services.AddGrpcClient<ProductService.ProductServiceClient>(options =>
{
    options.Address = new Uri("https://grpc-product-service:7042");
})
.ConfigureChannel(options =>
{
    // ตั้งค่า channel
    options.MaxReceiveMessageSize = 16 * 1024 * 1024;
    options.MaxSendMessageSize = 16 * 1024 * 1024;
    
    // ตั้งค่า credentials
    options.Credentials = ChannelCredentials.SecureSsl;
})
.AddCallCredentials(async (context, metadata) =>
{
    // ใส่ token อัตโนมัติ
    var tokenService = context.ServiceProvider.GetRequiredService<ITokenService>();
    var token = await tokenService.GetTokenAsync();
    metadata.Add("Authorization", $"Bearer {token}");
})
.ConfigurePrimaryHttpMessageHandler(() =>
{
    // Custom HTTP handler
    return new HttpClientHandler
    {
        ServerCertificateCustomValidationCallback =
            HttpClientHandler.DangerousAcceptAnyServerCertificateValidator
    };
})
.AddRetryPolicy(new GrpcRetryPolicy
{
    MaxAttempts = 3,
    RetryableStatusCodes =
    {
        StatusCode.Unavailable,
        StatusCode.DeadlineExceeded
    }
})
.EnableCallContextPropagation(); // propagate deadline/cancellation

// เพิ่ม clients อื่น ๆ
builder.Services.AddGrpcClient<OrderService.OrderServiceClient>(options =>
{
    options.Address = new Uri("https://grpc-order-service:7043");
});

var app = builder.Build();
```

### ใช้ Client ใน Service

```csharp
// Services/ProductApplicationService.cs
public class ProductApplicationService
{
    private readonly ProductService.ProductServiceClient _productClient;
    private readonly ILogger<ProductApplicationService> _logger;

    public ProductApplicationService(
        ProductService.ProductServiceClient productClient,
        ILogger<ProductApplicationService> logger)
    {
        _productClient = productClient;
        _logger = logger;
    }

    public async Task<ProductDto> GetProductAsync(string productId, CancellationToken ct = default)
    {
        try
        {
            var response = await _productClient.GetProductAsync(
                new GetProductRequest { ProductId = productId },
                cancellationToken: ct);

            return new ProductDto
            {
                Id = response.ProductId,
                Name = response.Name,
                Price = response.Price,
                Stock = response.StockQuantity
            };
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.NotFound)
        {
            throw new ProductNotFoundException(productId);
        }
        catch (RpcException ex)
        {
            _logger.LogError(ex, "gRPC error getting product {ProductId}", productId);
            throw new ServiceException($"Failed to get product: {ex.Status.Detail}");
        }
    }

    public async Task<IReadOnlyList<ProductDto>> GetProductsByCategory(
        string category,
        CancellationToken ct = default)
    {
        var products = new List<ProductDto>();

        using var call = _productClient.ListProducts(new ListProductsRequest
        {
            Category = category,
            PageSize = 100
        }, cancellationToken: ct);

        await foreach (var product in call.ResponseStream.ReadAllAsync(ct))
        {
            products.Add(new ProductDto
            {
                Id = product.ProductId,
                Name = product.Name,
                Price = product.Price,
                Stock = product.StockQuantity
            });
        }

        return products;
    }
}
```

### Polly Retry Policy กับ gRPC

```csharp
// Extensions/GrpcRetryExtensions.cs
using Microsoft.Extensions.DependencyInjection;
using Polly;
using Polly.Extensions.Http;
using Grpc.Core;

public static class GrpcRetryExtensions
{
    public static IHttpClientBuilder AddGrpcRetryPolicy(
        this IHttpClientBuilder builder,
        int retryCount = 3)
    {
        return builder.AddPolicyHandler(GetRetryPolicy(retryCount));
    }

    private static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy(int retryCount)
    {
        return HttpPolicyExtensions
            .HandleTransientHttpError()
            .OrResult(msg => msg.StatusCode == System.Net.HttpStatusCode.ServiceUnavailable)
            .WaitAndRetryAsync(
                retryCount,
                retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                onRetry: (outcome, timespan, retryAttempt, context) =>
                {
                    var logger = context.GetLogger();
                    logger?.LogWarning(
                        "gRPC retry {RetryAttempt} after {Delay}s",
                        retryAttempt, timespan.TotalSeconds);
                });
    }
}

// ใช้งาน
builder.Services.AddGrpcClient<ProductService.ProductServiceClient>(options =>
{
    options.Address = new Uri("https://grpc-service:7042");
})
.AddGrpcRetryPolicy(retryCount: 3);
```

---

## Step 677: Authentication และ Interceptors ใน gRPC

### สร้าง Interceptors

Interceptors ใน gRPC ทำงานคล้าย middleware ใน ASP.NET Core ช่วยในการ cross-cutting concerns เช่น logging, authentication, tracing

#### Logging Interceptor

```csharp
// Interceptors/LoggingInterceptor.cs
using Grpc.Core;
using Grpc.Core.Interceptors;
using System.Diagnostics;

public class LoggingInterceptor : Interceptor
{
    private readonly ILogger<LoggingInterceptor> _logger;

    public LoggingInterceptor(ILogger<LoggingInterceptor> logger)
    {
        _logger = logger;
    }

    // Intercept Unary calls
    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        var method = context.Method;
        var peer = context.Peer;
        var sw = Stopwatch.StartNew();

        _logger.LogInformation(
            "gRPC call started | Method: {Method} | Peer: {Peer}",
            method, peer);

        try
        {
            var response = await continuation(request, context);
            sw.Stop();

            _logger.LogInformation(
                "gRPC call completed | Method: {Method} | Duration: {Duration}ms",
                method, sw.ElapsedMilliseconds);

            return response;
        }
        catch (RpcException ex)
        {
            sw.Stop();
            _logger.LogWarning(
                "gRPC call failed | Method: {Method} | Status: {Status} | Duration: {Duration}ms | Detail: {Detail}",
                method, ex.StatusCode, sw.ElapsedMilliseconds, ex.Status.Detail);
            throw;
        }
        catch (Exception ex)
        {
            sw.Stop();
            _logger.LogError(ex,
                "gRPC unhandled error | Method: {Method} | Duration: {Duration}ms",
                method, sw.ElapsedMilliseconds);
            throw new RpcException(new Status(StatusCode.Internal, ex.Message));
        }
    }

    // Intercept Server Streaming calls
    public override async Task ServerStreamingServerHandler<TRequest, TResponse>(
        TRequest request,
        IServerStreamWriter<TResponse> responseStream,
        ServerCallContext context,
        ServerStreamingServerMethod<TRequest, TResponse> continuation)
    {
        var method = context.Method;
        _logger.LogInformation("gRPC server stream started | Method: {Method}", method);

        var wrappedStream = new LoggingServerStreamWriter<TResponse>(responseStream, _logger, method);
        await continuation(request, wrappedStream, context);

        _logger.LogInformation("gRPC server stream completed | Method: {Method} | Sent: {Count}",
            method, wrappedStream.MessageCount);
    }
}

// Helper wrapper สำหรับนับ messages
public class LoggingServerStreamWriter<T> : IServerStreamWriter<T>
{
    private readonly IServerStreamWriter<T> _inner;
    private readonly ILogger _logger;
    private readonly string _method;
    public int MessageCount { get; private set; }

    public LoggingServerStreamWriter(IServerStreamWriter<T> inner, ILogger logger, string method)
    {
        _inner = inner;
        _logger = logger;
        _method = method;
    }

    public WriteOptions? WriteOptions
    {
        get => _inner.WriteOptions;
        set => _inner.WriteOptions = value;
    }

    public async Task WriteAsync(T message)
    {
        await _inner.WriteAsync(message);
        MessageCount++;
        _logger.LogDebug("Stream message {Count} sent | Method: {Method}", MessageCount, _method);
    }
}
```

#### Authentication Interceptor

```csharp
// Interceptors/AuthenticationInterceptor.cs
using Microsoft.IdentityModel.Tokens;
using System.IdentityModel.Tokens.Jwt;

public class AuthenticationInterceptor : Interceptor
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<AuthenticationInterceptor> _logger;

    // Methods ที่ไม่ต้อง auth
    private static readonly HashSet<string> _publicMethods = new(StringComparer.OrdinalIgnoreCase)
    {
        "/product.ProductService/GetProduct",
        "/product.ProductService/ListProducts",
    };

    public AuthenticationInterceptor(
        IConfiguration configuration,
        ILogger<AuthenticationInterceptor> logger)
    {
        _configuration = configuration;
        _logger = logger;
    }

    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        if (!_publicMethods.Contains(context.Method))
        {
            await ValidateTokenAsync(context);
        }

        return await continuation(request, context);
    }

    public override async Task ServerStreamingServerHandler<TRequest, TResponse>(
        TRequest request,
        IServerStreamWriter<TResponse> responseStream,
        ServerCallContext context,
        ServerStreamingServerMethod<TRequest, TResponse> continuation)
    {
        if (!_publicMethods.Contains(context.Method))
        {
            await ValidateTokenAsync(context);
        }

        await continuation(request, responseStream, context);
    }

    private async Task ValidateTokenAsync(ServerCallContext context)
    {
        var authHeader = context.RequestHeaders
            .FirstOrDefault(e => e.Key == "authorization")?.Value;

        if (string.IsNullOrEmpty(authHeader) || !authHeader.StartsWith("Bearer "))
        {
            throw new RpcException(new Status(
                StatusCode.Unauthenticated,
                "Missing or invalid Authorization header"));
        }

        var token = authHeader.Substring("Bearer ".Length);

        try
        {
            var tokenHandler = new JwtSecurityTokenHandler();
            var validationParameters = new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(
                    System.Text.Encoding.UTF8.GetBytes(
                        _configuration["Jwt:SecretKey"]!)),
                ValidateIssuer = true,
                ValidIssuer = _configuration["Jwt:Issuer"],
                ValidateAudience = true,
                ValidAudience = _configuration["Jwt:Audience"],
                ValidateLifetime = true,
                ClockSkew = TimeSpan.Zero
            };

            var principal = tokenHandler.ValidateToken(
                token, validationParameters, out _);

            // เก็บ user info ใน context
            context.UserState["user"] = principal;

            _logger.LogDebug(
                "Token validated for user: {User}",
                principal.Identity?.Name);
        }
        catch (SecurityTokenExpiredException)
        {
            throw new RpcException(new Status(StatusCode.Unauthenticated, "Token has expired"));
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Token validation failed");
            throw new RpcException(new Status(StatusCode.Unauthenticated, "Invalid token"));
        }
    }
}
```

#### Client-side Interceptor

```csharp
// ClientInterceptors/ClientLoggingInterceptor.cs
public class ClientLoggingInterceptor : Interceptor
{
    private readonly ILogger<ClientLoggingInterceptor> _logger;

    public ClientLoggingInterceptor(ILogger<ClientLoggingInterceptor> logger)
    {
        _logger = logger;
    }

    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        _logger.LogInformation("Calling gRPC: {Method}", context.Method);

        var call = continuation(request, context);

        return new AsyncUnaryCall<TResponse>(
            HandleResponse(call.ResponseAsync, context.Method.FullName),
            call.ResponseHeadersAsync,
            call.GetStatus,
            call.GetTrailers,
            call.Dispose);
    }

    private async Task<TResponse> HandleResponse<TResponse>(
        Task<TResponse> task,
        string methodName)
    {
        try
        {
            var response = await task;
            _logger.LogInformation("gRPC call succeeded: {Method}", methodName);
            return response;
        }
        catch (RpcException ex)
        {
            _logger.LogError("gRPC call failed: {Method} - {Status}: {Detail}",
                methodName, ex.StatusCode, ex.Status.Detail);
            throw;
        }
    }
}

// ใช้งาน client interceptor
builder.Services.AddGrpcClient<ProductService.ProductServiceClient>(options =>
{
    options.Address = new Uri("https://grpc-service:7042");
})
.AddInterceptor<ClientLoggingInterceptor>()
.AddInterceptor(() => new HeaderPropagationInterceptor("correlation-id"));
```

---

## Step 678: gRPC-Web สำหรับ Browser Clients

### ปัญหาของ gRPC กับ Browser

Browser ปกติไม่รองรับ HTTP/2 trailers ที่ gRPC ต้องการ gRPC-Web เป็น protocol ที่ช่วยให้ browser สามารถเรียกใช้ gRPC services ได้

### ตั้งค่า gRPC-Web Server

```bash
# เพิ่ม package
dotnet add package Grpc.AspNetCore.Web
```

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddGrpc();
builder.Services.AddGrpcReflection();

// เพิ่ม CORS สำหรับ browser
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader()
              .WithExposedHeaders("Grpc-Status", "Grpc-Message", "Grpc-Encoding", "Grpc-Accept-Encoding");
    });
});

var app = builder.Build();

// ใช้ CORS ก่อน gRPC-Web
app.UseCors("AllowAll");

// เปิดใช้งาน gRPC-Web
app.UseGrpcWeb(new GrpcWebOptions { DefaultEnabled = true });

// Map services ด้วย gRPC-Web
app.MapGrpcService<GrpcProductServiceImpl>().EnableGrpcWeb();
app.MapGrpcService<GrpcOrderService>().EnableGrpcWeb();

if (app.Environment.IsDevelopment())
{
    app.MapGrpcReflectionService();
}

app.Run();
```

### JavaScript/TypeScript Client สำหรับ gRPC-Web

```javascript
// client/src/grpc-client.js
// ต้องติดตั้ง: npm install grpc-web google-protobuf

import { ProductServiceClient } from './generated/product_grpc_web_pb';
import { GetProductRequest, ListProductsRequest } from './generated/product_pb';

// สร้าง client
const client = new ProductServiceClient('http://localhost:5042');

// Unary call
async function getProduct(productId) {
    return new Promise((resolve, reject) => {
        const request = new GetProductRequest();
        request.setProductId(productId);

        // เพิ่ม metadata (headers)
        const metadata = {
            'Authorization': `Bearer ${getToken()}`,
            'x-request-id': crypto.randomUUID()
        };

        client.getProduct(request, metadata, (err, response) => {
            if (err) {
                console.error('gRPC error:', err.code, err.message);
                reject(err);
            } else {
                resolve({
                    id: response.getProductId(),
                    name: response.getName(),
                    price: response.getPrice(),
                    stock: response.getStockQuantity()
                });
            }
        });
    });
}

// Server streaming
function listProducts(category, onProduct, onDone, onError) {
    const request = new ListProductsRequest();
    request.setCategory(category);
    request.setPageSize(50);

    const stream = client.listProducts(request, {
        'Authorization': `Bearer ${getToken()}`
    });

    stream.on('data', (product) => {
        onProduct({
            id: product.getProductId(),
            name: product.getName(),
            price: product.getPrice()
        });
    });

    stream.on('end', onDone);
    stream.on('error', onError);
    stream.on('status', (status) => {
        console.log('Stream status:', status.code, status.details);
    });

    // ยกเลิก stream
    return () => stream.cancel();
}

// ตัวอย่างการใช้งาน
async function main() {
    try {
        // Get single product
        const product = await getProduct('prod-001');
        console.log('Product:', product.name, '@', product.price);

        // List products with streaming
        const cancelStream = listProducts(
            'Electronics',
            (p) => console.log('Received:', p.name),
            () => console.log('Stream ended'),
            (err) => console.error('Stream error:', err)
        );

        // Cancel after 10 seconds
        setTimeout(cancelStream, 10000);

    } catch (err) {
        if (err.code === 5) { // NOT_FOUND
            console.error('Product not found');
        } else if (err.code === 7) { // PERMISSION_DENIED
            console.error('Access denied');
        } else {
            console.error('Error:', err.message);
        }
    }
}

function getToken() {
    return localStorage.getItem('auth_token') || '';
}

main();
```

### Blazor WebAssembly กับ gRPC-Web

```csharp
// BlazorApp/Program.cs
using Grpc.Net.Client;
using Grpc.Net.Client.Web;

var builder = WebAssemblyHostBuilder.CreateDefault(args);
builder.RootComponents.Add<App>("#app");

// ตั้งค่า gRPC-Web client สำหรับ Blazor WASM
builder.Services.AddSingleton(sp =>
{
    var httpClient = new HttpClient(new GrpcWebHandler(GrpcWebMode.GrpcWeb, new HttpClientHandler()));
    var channel = GrpcChannel.ForAddress(
        "https://api.example.com",
        new GrpcChannelOptions { HttpClient = httpClient });
    return new ProductService.ProductServiceClient(channel);
});

await builder.Build().RunAsync();

// BlazorApp/Pages/Products.razor
@page "/products"
@inject ProductService.ProductServiceClient GrpcClient

<h1>Products</h1>

@if (_loading)
{
    <p>Loading...</p>
}
else
{
    <table class="table">
        <thead>
            <tr><th>Name</th><th>Price</th><th>Stock</th></tr>
        </thead>
        <tbody>
            @foreach (var product in _products)
            {
                <tr>
                    <td>@product.Name</td>
                    <td>@product.Price.ToString("C")</td>
                    <td>@product.StockQuantity</td>
                </tr>
            }
        </tbody>
    </table>
}

@code {
    private List<ProductResponse> _products = new();
    private bool _loading = true;

    protected override async Task OnInitializedAsync()
    {
        try
        {
            using var call = GrpcClient.ListProducts(new ListProductsRequest
            {
                Category = "Electronics",
                PageSize = 50
            });

            await foreach (var product in call.ResponseStream.ReadAllAsync())
            {
                _products.Add(product);
                StateHasChanged(); // อัปเดต UI ทีละรายการ
            }
        }
        finally
        {
            _loading = false;
            StateHasChanged();
        }
    }
}
```

---

## Step 679: gRPC Health Checks และ Reflection

### gRPC Health Check Protocol

gRPC มี standard health check protocol ที่ช่วย load balancers และ orchestrators (Kubernetes) ตรวจสอบ service status

```bash
# เพิ่ม package
dotnet add package Grpc.HealthCheck
dotnet add package AspNetCore.HealthChecks.SqlServer
dotnet add package AspNetCore.HealthChecks.Redis
```

```csharp
// Program.cs - Health Check Setup
using Grpc.HealthCheck;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddGrpc();

// เพิ่ม health check services
builder.Services.AddGrpcHealthChecks()
    .AddCheck<ProductServiceHealthCheck>("ProductService")
    .AddCheck<DatabaseHealthCheck>("Database")
    .AddSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "sql-server")
    .AddRedis(
        builder.Configuration["Redis:ConnectionString"]!,
        name: "redis");

var app = builder.Build();

app.MapGrpcService<GrpcProductServiceImpl>();
app.MapGrpcHealthChecksService();

app.Run();
```

#### Custom Health Check

```csharp
// HealthChecks/ProductServiceHealthCheck.cs
using Microsoft.Extensions.Diagnostics.HealthChecks;

public class ProductServiceHealthCheck : IHealthCheck
{
    private readonly IProductRepository _repository;
    private readonly ILogger<ProductServiceHealthCheck> _logger;

    public ProductServiceHealthCheck(
        IProductRepository repository,
        ILogger<ProductServiceHealthCheck> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // ตรวจสอบว่า repository ทำงานได้
            var count = await _repository.GetCountAsync(cancellationToken);

            var data = new Dictionary<string, object>
            {
                { "product_count", count },
                { "checked_at", DateTime.UtcNow }
            };

            if (count < 0)
            {
                return HealthCheckResult.Unhealthy(
                    "Repository returned invalid count",
                    data: data);
            }

            return HealthCheckResult.Healthy(
                $"Service is healthy. Products: {count}",
                data: data);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Health check failed");
            return HealthCheckResult.Unhealthy(
                "Repository is unavailable",
                exception: ex);
        }
    }
}
```

#### Health Check Client

```csharp
// ตรวจสอบ health ของ gRPC service
using Grpc.Core;
using Grpc.Health.V1;
using Grpc.Net.Client;

async Task CheckServiceHealth(string serviceAddress, string serviceName)
{
    using var channel = GrpcChannel.ForAddress(serviceAddress);
    var client = new Health.HealthClient(channel);

    try
    {
        var response = await client.CheckAsync(new HealthCheckRequest
        {
            Service = serviceName // "" = overall health, หรือ "ProductService"
        });

        Console.WriteLine($"Service '{serviceName}': {response.Status}");

        switch (response.Status)
        {
            case HealthCheckResponse.Types.ServingStatus.Serving:
                Console.WriteLine("  Service is healthy and serving");
                break;
            case HealthCheckResponse.Types.ServingStatus.NotServing:
                Console.WriteLine("  Service is NOT serving");
                break;
            case HealthCheckResponse.Types.ServingStatus.Unknown:
                Console.WriteLine("  Service status is unknown");
                break;
        }
    }
    catch (RpcException ex) when (ex.StatusCode == StatusCode.NotFound)
    {
        Console.WriteLine($"  Health check not found for service: {serviceName}");
    }
}

// Streaming health watch
async Task WatchServiceHealth(string serviceAddress, string serviceName, CancellationToken ct)
{
    using var channel = GrpcChannel.ForAddress(serviceAddress);
    var client = new Health.HealthClient(channel);

    using var call = client.Watch(new HealthCheckRequest { Service = serviceName },
        cancellationToken: ct);

    Console.WriteLine($"Watching health of '{serviceName}'...");

    await foreach (var response in call.ResponseStream.ReadAllAsync(ct))
    {
        var status = response.Status == HealthCheckResponse.Types.ServingStatus.Serving
            ? "HEALTHY" : "UNHEALTHY";
        Console.WriteLine($"[{DateTime.Now:HH:mm:ss}] {serviceName}: {status}");
    }
}
```

### gRPC Reflection

gRPC Reflection ช่วยให้ tools เช่น grpcurl, Postman รู้จัก services และ methods ที่มีอยู่โดยไม่ต้องมี .proto file

```csharp
// Program.cs - เปิด Reflection
builder.Services.AddGrpcReflection();

if (app.Environment.IsDevelopment())
{
    app.MapGrpcReflectionService();
}
```

```bash
# ใช้ grpcurl ทดสอบ service
# ติดตั้ง: https://github.com/fullstorydev/grpcurl

# List services
grpcurl -plaintext localhost:5042 list

# Describe service
grpcurl -plaintext localhost:5042 describe product.ProductService

# Describe message
grpcurl -plaintext localhost:5042 describe product.CreateProductRequest

# เรียกใช้ method
grpcurl -plaintext \
  -d '{"name": "Test Product", "price": 100.0, "stock_quantity": 50, "category": "Test"}' \
  localhost:5042 \
  product.ProductService/CreateProduct

# List products (server streaming)
grpcurl -plaintext \
  -d '{"category": "Electronics", "page_size": 10}' \
  localhost:5042 \
  product.ProductService/ListProducts
```

### Kubernetes Health Check Configuration

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grpc-product-service
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: grpc-product-service
        image: grpc-product-service:latest
        ports:
        - containerPort: 5042
          name: grpc
        livenessProbe:
          grpc:
            port: 5042
            service: "" # ตรวจสอบ overall health
          initialDelaySeconds: 10
          periodSeconds: 30
          failureThreshold: 3
        readinessProbe:
          grpc:
            port: 5042
            service: "ProductService"
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
```

---

## Step 680: Full gRPC Product Catalog Service Example

### ภาพรวมระบบ

ส่วนนี้เป็นตัวอย่างที่สมบูรณ์ของ Product Catalog Service ที่ใช้ gRPC โดยครอบคลุมทุก pattern ที่เรียนมา

### Proto Definition ที่สมบูรณ์

```protobuf
// Protos/catalog.proto
syntax = "proto3";

option csharp_namespace = "ProductCatalog.Grpc";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/wrappers.proto";

package catalog;

// =====================
// Product Catalog Service
// =====================
service CatalogService {
  // Product CRUD
  rpc GetProduct       (GetProductRequest)       returns (Product);
  rpc CreateProduct    (CreateProductRequest)    returns (Product);
  rpc UpdateProduct    (UpdateProductRequest)    returns (Product);
  rpc DeleteProduct    (DeleteProductRequest)    returns (google.protobuf.Empty);

  // Search & Listing
  rpc SearchProducts   (SearchProductsRequest)   returns (stream Product);
  rpc GetProductsByIds (GetProductsByIdsRequest) returns (stream Product);

  // Inventory Management
  rpc UpdateStock      (stream StockUpdateRequest)  returns (StockUpdateSummary);
  rpc WatchInventory   (WatchInventoryRequest)      returns (stream InventoryEvent);

  // Price Management
  rpc BulkUpdatePrices (stream PriceUpdateRequest)  returns (BulkUpdateResult);
  rpc StreamPriceAlerts(PriceAlertSubscription)     returns (stream PriceAlert);
}

// =====================
// Order Service
// =====================
service OrderService {
  rpc CreateOrder      (CreateOrderRequest)   returns (Order);
  rpc GetOrder         (GetOrderRequest)      returns (Order);
  rpc CancelOrder      (CancelOrderRequest)   returns (Order);
  rpc StreamOrderStatus(StreamOrderRequest)   returns (stream OrderStatusUpdate);
  rpc ProcessOrders    (stream OrderItem)     returns (ProcessOrdersResult);
}

// =====================
// Messages
// =====================
message Product {
  string   id           = 1;
  string   name         = 2;
  string   description  = 3;
  double   price        = 4;
  int32    stock        = 5;
  string   category     = 6;
  string   sku          = 7;
  repeated string tags  = 8;
  ProductStatus status  = 9;
  google.protobuf.Timestamp created_at = 10;
  google.protobuf.Timestamp updated_at = 11;
}

enum ProductStatus {
  PRODUCT_STATUS_UNSPECIFIED = 0;
  PRODUCT_STATUS_ACTIVE      = 1;
  PRODUCT_STATUS_INACTIVE    = 2;
  PRODUCT_STATUS_OUT_OF_STOCK= 3;
  PRODUCT_STATUS_DISCONTINUED= 4;
}

message GetProductRequest    { string id = 1; }
message DeleteProductRequest { string id = 1; }

message CreateProductRequest {
  string   name        = 1;
  string   description = 2;
  double   price       = 3;
  int32    stock       = 4;
  string   category    = 5;
  string   sku         = 6;
  repeated string tags = 7;
}

message UpdateProductRequest {
  string id                               = 1;
  google.protobuf.StringValue name        = 2;
  google.protobuf.StringValue description = 3;
  google.protobuf.DoubleValue price       = 4;
  google.protobuf.Int32Value  stock       = 5;
  repeated string tags                    = 6;
}

message SearchProductsRequest {
  string   query        = 1;
  string   category     = 2;
  double   min_price    = 3;
  double   max_price    = 4;
  repeated string tags  = 5;
  string   sort_by      = 6;
  bool     sort_desc    = 7;
  int32    page_size    = 8;
}

message GetProductsByIdsRequest {
  repeated string ids = 1;
}

message StockUpdateRequest {
  string product_id  = 1;
  int32  delta       = 2;
  string reason      = 3;
  string reference   = 4;
}

message StockUpdateSummary {
  int32    total_processed = 1;
  int32    successful      = 2;
  int32    failed          = 3;
  repeated string errors   = 4;
}

message WatchInventoryRequest {
  repeated string product_ids  = 1;
  int32           low_stock_threshold = 2;
}

message InventoryEvent {
  string product_id   = 1;
  int32  old_stock    = 2;
  int32  new_stock    = 3;
  string event_type   = 4;
  bool   is_low_stock = 5;
  google.protobuf.Timestamp occurred_at = 6;
}

message PriceUpdateRequest {
  string product_id = 1;
  double new_price  = 2;
  string reason     = 3;
}

message BulkUpdateResult {
  int32    updated = 1;
  int32    failed  = 2;
  repeated string errors = 3;
}

message PriceAlertSubscription {
  repeated string product_ids       = 1;
  double          min_change_percent = 2;
}

message PriceAlert {
  string product_id      = 1;
  double old_price       = 2;
  double new_price       = 3;
  double change_percent  = 4;
  google.protobuf.Timestamp changed_at = 5;
}

message Order {
  string id              = 1;
  string customer_id     = 2;
  repeated OrderItem items= 3;
  double total_amount    = 4;
  OrderStatus status     = 5;
  google.protobuf.Timestamp created_at = 6;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING     = 1;
  ORDER_STATUS_CONFIRMED   = 2;
  ORDER_STATUS_PROCESSING  = 3;
  ORDER_STATUS_SHIPPED     = 4;
  ORDER_STATUS_DELIVERED   = 5;
  ORDER_STATUS_CANCELLED   = 6;
}

message OrderItem {
  string product_id = 1;
  int32  quantity   = 2;
  double price      = 3;
}

message CreateOrderRequest {
  string             customer_id = 1;
  repeated OrderItem items       = 2;
}

message GetOrderRequest    { string id = 1; }
message CancelOrderRequest { string id = 1; string reason = 2; }
message StreamOrderRequest { string order_id = 1; }

message OrderStatusUpdate {
  string      order_id    = 1;
  OrderStatus old_status  = 2;
  OrderStatus new_status  = 3;
  string      note        = 4;
  google.protobuf.Timestamp updated_at = 5;
}

message ProcessOrdersResult {
  int32    processed = 1;
  int32    failed    = 2;
  double   total     = 3;
}
```

### Full Service Implementation

```csharp
// Services/CatalogServiceImpl.cs
using Grpc.Core;
using Google.Protobuf.WellKnownTypes;
using ProductCatalog.Grpc;
using ProductCatalog.Domain;

namespace ProductCatalog.Services;

public class CatalogServiceImpl : CatalogService.CatalogServiceBase
{
    private readonly IProductRepository _products;
    private readonly IInventoryEventBus _eventBus;
    private readonly IPriceAlertService _priceAlerts;
    private readonly ILogger<CatalogServiceImpl> _logger;

    public CatalogServiceImpl(
        IProductRepository products,
        IInventoryEventBus eventBus,
        IPriceAlertService priceAlerts,
        ILogger<CatalogServiceImpl> logger)
    {
        _products = products;
        _eventBus = eventBus;
        _priceAlerts = priceAlerts;
        _logger = logger;
    }

    // ==============================
    // CRUD Operations
    // ==============================

    public override async Task<Product> GetProduct(
        GetProductRequest request, ServerCallContext context)
    {
        ValidateId(request.Id, "product");

        var product = await _products.FindAsync(request.Id)
            ?? throw new RpcException(new Status(
                StatusCode.NotFound, $"Product '{request.Id}' not found"));

        return ToProto(product);
    }

    public override async Task<Product> CreateProduct(
        CreateProductRequest request, ServerCallContext context)
    {
        ValidateCreateRequest(request);

        // ตรวจสอบ SKU ซ้ำ
        if (!string.IsNullOrEmpty(request.Sku))
        {
            var existing = await _products.FindBySkuAsync(request.Sku);
            if (existing != null)
            {
                throw new RpcException(new Status(
                    StatusCode.AlreadyExists,
                    $"Product with SKU '{request.Sku}' already exists"));
            }
        }

        var entity = new ProductEntity
        {
            Id = Guid.NewGuid().ToString(),
            Name = request.Name,
            Description = request.Description,
            Price = request.Price,
            Stock = request.Stock,
            Category = request.Category,
            Sku = request.Sku,
            Tags = request.Tags.ToList(),
            Status = ProductStatusEnum.Active,
            CreatedAt = DateTime.UtcNow,
            UpdatedAt = DateTime.UtcNow
        };

        await _products.AddAsync(entity);

        _logger.LogInformation("Product created: {Id} - {Name}", entity.Id, entity.Name);
        return ToProto(entity);
    }

    public override async Task<Product> UpdateProduct(
        UpdateProductRequest request, ServerCallContext context)
    {
        ValidateId(request.Id, "product");

        var entity = await _products.FindAsync(request.Id)
            ?? throw new RpcException(new Status(
                StatusCode.NotFound, $"Product '{request.Id}' not found"));

        var oldPrice = entity.Price;

        // อัปเดตเฉพาะ fields ที่ระบุมา
        if (request.Name != null)        entity.Name = request.Name.Value;
        if (request.Description != null) entity.Description = request.Description.Value;
        if (request.Price != null)       entity.Price = request.Price.Value;
        if (request.Stock != null)       entity.Stock = request.Stock.Value;
        if (request.Tags.Count > 0)      entity.Tags = request.Tags.ToList();
        entity.UpdatedAt = DateTime.UtcNow;

        // อัปเดต status ตาม stock
        if (entity.Stock == 0)
            entity.Status = ProductStatusEnum.OutOfStock;
        else if (entity.Status == ProductStatusEnum.OutOfStock)
            entity.Status = ProductStatusEnum.Active;

        await _products.UpdateAsync(entity);

        // แจ้งเตือนถ้าราคาเปลี่ยน
        if (request.Price != null && Math.Abs(oldPrice - entity.Price) > 0.001)
        {
            await _priceAlerts.PublishPriceChangeAsync(entity.Id, oldPrice, entity.Price);
        }

        return ToProto(entity);
    }

    public override async Task<Empty> DeleteProduct(
        DeleteProductRequest request, ServerCallContext context)
    {
        ValidateId(request.Id, "product");

        var deleted = await _products.DeleteAsync(request.Id);
        if (!deleted)
        {
            throw new RpcException(new Status(
                StatusCode.NotFound, $"Product '{request.Id}' not found"));
        }

        _logger.LogInformation("Product deleted: {Id}", request.Id);
        return new Empty();
    }

    // ==============================
    // Search & Listing
    // ==============================

    public override async Task SearchProducts(
        SearchProductsRequest request,
        IServerStreamWriter<Product> responseStream,
        ServerCallContext context)
    {
        var filter = new ProductFilter
        {
            Query = request.Query,
            Category = request.Category,
            MinPrice = request.MinPrice > 0 ? request.MinPrice : null,
            MaxPrice = request.MaxPrice > 0 ? request.MaxPrice : null,
            Tags = request.Tags.ToList(),
            SortBy = request.SortBy,
            SortDescending = request.SortDesc,
            PageSize = request.PageSize > 0 ? request.PageSize : 50
        };

        var products = _products.SearchAsync(filter, context.CancellationToken);

        await foreach (var product in products.WithCancellation(context.CancellationToken))
        {
            await responseStream.WriteAsync(ToProto(product));
        }
    }

    public override async Task GetProductsByIds(
        GetProductsByIdsRequest request,
        IServerStreamWriter<Product> responseStream,
        ServerCallContext context)
    {
        if (request.Ids.Count == 0)
        {
            throw new RpcException(new Status(
                StatusCode.InvalidArgument, "At least one ID is required"));
        }

        if (request.Ids.Count > 1000)
        {
            throw new RpcException(new Status(
                StatusCode.InvalidArgument, "Maximum 1000 IDs per request"));
        }

        // ดึงข้อมูลเป็น batches
        const int batchSize = 50;
        for (int i = 0; i < request.Ids.Count; i += batchSize)
        {
            if (context.CancellationToken.IsCancellationRequested) break;

            var batch = request.Ids.Skip(i).Take(batchSize);
            var products = await _products.GetByIdsAsync(batch, context.CancellationToken);

            foreach (var product in products)
            {
                await responseStream.WriteAsync(ToProto(product));
            }
        }
    }

    // ==============================
    // Inventory Management
    // ==============================

    public override async Task<StockUpdateSummary> UpdateStock(
        IAsyncStreamReader<StockUpdateRequest> requestStream,
        ServerCallContext context)
    {
        var summary = new StockUpdateSummary();
        var errors = new List<string>();

        await foreach (var request in requestStream.ReadAllAsync(context.CancellationToken))
        {
            summary.TotalProcessed++;

            try
            {
                var product = await _products.FindAsync(request.ProductId);
                if (product == null)
                {
                    errors.Add($"Product '{request.ProductId}' not found");
                    summary.Failed++;
                    continue;
                }

                var oldStock = product.Stock;
                product.Stock = Math.Max(0, product.Stock + request.Delta);
                product.UpdatedAt = DateTime.UtcNow;

                if (product.Stock == 0)
                    product.Status = ProductStatusEnum.OutOfStock;
                else if (product.Status == ProductStatusEnum.OutOfStock && product.Stock > 0)
                    product.Status = ProductStatusEnum.Active;

                await _products.UpdateAsync(product);

                // Publish inventory event
                await _eventBus.PublishAsync(new InventoryChangedEvent
                {
                    ProductId = product.Id,
                    OldStock = oldStock,
                    NewStock = product.Stock,
                    Reason = request.Reason,
                    Reference = request.Reference,
                    OccurredAt = DateTime.UtcNow
                });

                summary.Successful++;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error updating stock for {ProductId}", request.ProductId);
                errors.Add($"Error for '{request.ProductId}': {ex.Message}");
                summary.Failed++;
            }
        }

        summary.Errors.AddRange(errors);
        return summary;
    }

    public override async Task WatchInventory(
        WatchInventoryRequest request,
        IServerStreamWriter<InventoryEvent> responseStream,
        ServerCallContext context)
    {
        var productIds = request.ProductIds.ToHashSet();
        var threshold = request.LowStockThreshold > 0 ? request.LowStockThreshold : 10;

        _logger.LogInformation(
            "Starting inventory watch for {Count} products",
            productIds.Count);

        var subscription = _eventBus.Subscribe<InventoryChangedEvent>();

        try
        {
            await foreach (var @event in subscription.WithCancellation(context.CancellationToken))
            {
                // กรองเฉพาะ products ที่สนใจ
                if (productIds.Count > 0 && !productIds.Contains(@event.ProductId))
                    continue;

                await responseStream.WriteAsync(new InventoryEvent
                {
                    ProductId = @event.ProductId,
                    OldStock = @event.OldStock,
                    NewStock = @event.NewStock,
                    EventType = @event.Reason ?? "update",
                    IsLowStock = @event.NewStock <= threshold,
                    OccurredAt = Timestamp.FromDateTime(@event.OccurredAt)
                });
            }
        }
        finally
        {
            _eventBus.Unsubscribe(subscription);
        }
    }

    // ==============================
    // Price Management
    // ==============================

    public override async Task<BulkUpdateResult> BulkUpdatePrices(
        IAsyncStreamReader<PriceUpdateRequest> requestStream,
        ServerCallContext context)
    {
        var result = new BulkUpdateResult();
        var errors = new List<string>();

        await foreach (var request in requestStream.ReadAllAsync(context.CancellationToken))
        {
            try
            {
                if (request.NewPrice <= 0)
                {
                    errors.Add($"Invalid price for '{request.ProductId}': {request.NewPrice}");
                    result.Failed++;
                    continue;
                }

                var product = await _products.FindAsync(request.ProductId);
                if (product == null)
                {
                    errors.Add($"Product '{request.ProductId}' not found");
                    result.Failed++;
                    continue;
                }

                var oldPrice = product.Price;
                product.Price = request.NewPrice;
                product.UpdatedAt = DateTime.UtcNow;

                await _products.UpdateAsync(product);
                await _priceAlerts.PublishPriceChangeAsync(product.Id, oldPrice, product.Price);

                result.Updated++;
            }
            catch (Exception ex)
            {
                errors.Add($"Error: {ex.Message}");
                result.Failed++;
            }
        }

        result.Errors.AddRange(errors);
        return result;
    }

    public override async Task StreamPriceAlerts(
        PriceAlertSubscription request,
        IServerStreamWriter<PriceAlert> responseStream,
        ServerCallContext context)
    {
        var productIds = request.ProductIds.ToHashSet();
        var minChangePercent = request.MinChangePercent > 0 ? request.MinChangePercent : 1.0;

        var subscription = _priceAlerts.Subscribe(productIds);

        try
        {
            await foreach (var alert in subscription.WithCancellation(context.CancellationToken))
            {
                var changePercent = Math.Abs((alert.NewPrice - alert.OldPrice) / alert.OldPrice * 100);

                if (changePercent >= minChangePercent)
                {
                    await responseStream.WriteAsync(new PriceAlert
                    {
                        ProductId = alert.ProductId,
                        OldPrice = alert.OldPrice,
                        NewPrice = alert.NewPrice,
                        ChangePercent = changePercent,
                        ChangedAt = Timestamp.FromDateTime(alert.ChangedAt)
                    });
                }
            }
        }
        finally
        {
            _priceAlerts.Unsubscribe(subscription);
        }
    }

    // ==============================
    // Helper Methods
    // ==============================

    private static Product ToProto(ProductEntity entity) => new()
    {
        Id = entity.Id,
        Name = entity.Name,
        Description = entity.Description,
        Price = entity.Price,
        Stock = entity.Stock,
        Category = entity.Category,
        Sku = entity.Sku ?? "",
        Status = (ProductStatus)entity.Status,
        CreatedAt = Timestamp.FromDateTime(entity.CreatedAt),
        UpdatedAt = Timestamp.FromDateTime(entity.UpdatedAt),
        Tags = { entity.Tags }
    };

    private static void ValidateId(string id, string entityName)
    {
        if (string.IsNullOrWhiteSpace(id))
        {
            throw new RpcException(new Status(
                StatusCode.InvalidArgument, $"The {entityName} ID cannot be empty"));
        }
    }

    private static void ValidateCreateRequest(CreateProductRequest request)
    {
        var errors = new List<string>();

        if (string.IsNullOrWhiteSpace(request.Name))
            errors.Add("Name is required");

        if (request.Price <= 0)
            errors.Add("Price must be greater than 0");

        if (request.Stock < 0)
            errors.Add("Stock cannot be negative");

        if (errors.Count > 0)
        {
            throw new RpcException(new Status(
                StatusCode.InvalidArgument,
                string.Join("; ", errors)));
        }
    }
}
```

### Integration Test ตัวอย่าง

```csharp
// Tests/CatalogServiceIntegrationTests.cs
using Grpc.Core;
using Grpc.Net.Client;
using Microsoft.AspNetCore.Mvc.Testing;
using Xunit;

public class CatalogServiceIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public CatalogServiceIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }

    private CatalogService.CatalogServiceClient CreateClient()
    {
        var httpClient = _factory.CreateDefaultClient();
        var channel = GrpcChannel.ForAddress(
            httpClient.BaseAddress!,
            new GrpcChannelOptions { HttpClient = httpClient });
        return new CatalogService.CatalogServiceClient(channel);
    }

    [Fact]
    public async Task CreateProduct_ValidRequest_ReturnsCreatedProduct()
    {
        // Arrange
        var client = CreateClient();
        var request = new CreateProductRequest
        {
            Name = "Test Laptop",
            Description = "Test description",
            Price = 35000.00,
            Stock = 50,
            Category = "Electronics",
            Sku = "LAPTOP-TEST-001"
        };

        // Act
        var product = await client.CreateProductAsync(request);

        // Assert
        Assert.NotEmpty(product.Id);
        Assert.Equal("Test Laptop", product.Name);
        Assert.Equal(35000.00, product.Price);
        Assert.Equal(50, product.Stock);
        Assert.Equal(ProductStatus.Active, product.Status);
    }

    [Fact]
    public async Task GetProduct_NotFound_ThrowsRpcException()
    {
        // Arrange
        var client = CreateClient();

        // Act & Assert
        var exception = await Assert.ThrowsAsync<RpcException>(
            () => client.GetProductAsync(
                new GetProductRequest { Id = "non-existent-id" }).ResponseAsync);

        Assert.Equal(StatusCode.NotFound, exception.StatusCode);
        Assert.Contains("non-existent-id", exception.Status.Detail);
    }

    [Fact]
    public async Task SearchProducts_ReturnsMachingProducts()
    {
        // Arrange
        var client = CreateClient();

        // Act
        var request = new SearchProductsRequest
        {
            Query = "laptop",
            Category = "Electronics",
            MaxPrice = 50000,
            PageSize = 20
        };

        var products = new List<Product>();
        using var call = client.SearchProducts(request);
        await foreach (var product in call.ResponseStream.ReadAllAsync())
        {
            products.Add(product);
        }

        // Assert
        Assert.All(products, p => Assert.Contains(
            "laptop", p.Name.ToLower() + p.Description.ToLower()));
    }

    [Fact]
    public async Task UpdateStock_MultipleProducts_ReturnsCorrectSummary()
    {
        // Arrange
        var client = CreateClient();

        // สร้างสินค้าทดสอบก่อน
        var product1 = await client.CreateProductAsync(new CreateProductRequest
        {
            Name = "Product A", Price = 1000, Stock = 100, Category = "Test"
        });
        var product2 = await client.CreateProductAsync(new CreateProductRequest
        {
            Name = "Product B", Price = 2000, Stock = 200, Category = "Test"
        });

        // Act
        using var call = client.UpdateStock();

        await call.RequestStream.WriteAsync(new StockUpdateRequest
        {
            ProductId = product1.Id,
            Delta = -10,
            Reason = "sale"
        });

        await call.RequestStream.WriteAsync(new StockUpdateRequest
        {
            ProductId = product2.Id,
            Delta = 50,
            Reason = "restock"
        });

        await call.RequestStream.WriteAsync(new StockUpdateRequest
        {
            ProductId = "non-existent",
            Delta = -5,
            Reason = "sale"
        });

        await call.RequestStream.CompleteAsync();
        var result = await call.ResponseAsync;

        // Assert
        Assert.Equal(3, result.TotalProcessed);
        Assert.Equal(2, result.Successful);
        Assert.Equal(1, result.Failed);
        Assert.Single(result.Errors);
    }

    [Fact]
    public async Task GetProduct_WithDeadline_CancelsOnTimeout()
    {
        // Arrange
        var client = CreateClient();

        // Act & Assert
        var exception = await Assert.ThrowsAsync<RpcException>(async () =>
        {
            // ตั้ง deadline ที่ผ่านมาแล้ว
            var deadline = DateTime.UtcNow.AddMilliseconds(-1);
            await client.GetProductAsync(
                new GetProductRequest { Id = "any-id" },
                deadline: deadline);
        });

        Assert.Equal(StatusCode.DeadlineExceeded, exception.StatusCode);
    }
}
```

### การ Deploy และ Configuration สำหรับ Production

```csharp
// appsettings.Production.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "Kestrel": {
    "Endpoints": {
      "Grpc": {
        "Url": "https://0.0.0.0:443",
        "Protocols": "Http2"
      }
    },
    "Certificates": {
      "Default": {
        "Path": "/etc/ssl/certs/service.pfx",
        "Password": "#{CERT_PASSWORD}#"
      }
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "#{DB_CONNECTION}#"
  },
  "Redis": {
    "ConnectionString": "#{REDIS_CONNECTION}#"
  },
  "Jwt": {
    "SecretKey": "#{JWT_SECRET_KEY}#",
    "Issuer": "product-catalog-service",
    "Audience": "product-catalog-clients"
  }
}

// Dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["ProductCatalog.csproj", "."]
RUN dotnet restore
COPY . .
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "ProductCatalog.dll"]
```

### สรุป Best Practices ของ gRPC

```csharp
// ProductCatalog.Grpc/BestPractices.cs

/*
 * Best Practices สำหรับ gRPC ใน C#/.NET
 *
 * 1. ERROR HANDLING
 *    - ใช้ RpcException พร้อม StatusCode ที่เหมาะสม
 *    - StatusCode.NotFound สำหรับ resource ที่ไม่พบ
 *    - StatusCode.InvalidArgument สำหรับ input ที่ไม่ถูกต้อง
 *    - StatusCode.AlreadyExists สำหรับ duplicate
 *    - StatusCode.PermissionDenied สำหรับ authorization error
 *    - StatusCode.Unauthenticated สำหรับ authentication error
 *    - StatusCode.Internal สำหรับ server error
 *
 * 2. DEADLINES & TIMEOUTS
 *    - ตั้ง deadline ทุกครั้งที่ call ฝั่ง client
 *    - ตรวจสอบ CancellationToken ใน server ที่มี loop/stream
 *    - Propagate cancellation token ผ่าน service layers
 *
 * 3. STREAMING
 *    - ตรวจสอบ context.CancellationToken.IsCancellationRequested ใน loop
 *    - ใช้ IAsyncEnumerable สำหรับ efficient streaming
 *    - Handle stream errors gracefully
 *
 * 4. PERFORMANCE
 *    - Reuse GrpcChannel (expensive to create)
 *    - ใช้ IHttpClientFactory สำหรับจัดการ channel lifecycle
 *    - Enable compression สำหรับ large payloads
 *    - ใช้ bytes แทน string สำหรับ binary data
 *
 * 5. SECURITY
 *    - ใช้ TLS ในทุก environment ยกเว้น development
 *    - Validate token ใน interceptor ไม่ใช่ใน method
 *    - ไม่ expose internal errors ให้ client ใน production
 *
 * 6. TESTING
 *    - ใช้ WebApplicationFactory สำหรับ integration tests
 *    - Mock grpc client ด้วย Moq
 *    - ทดสอบ error cases เช่น NotFound, InvalidArgument
 */

// ตัวอย่างการใช้ gRPC อย่างถูกต้อง
public class ProductServiceClient
{
    private readonly CatalogService.CatalogServiceClient _client;

    public async Task<Product?> TryGetProductAsync(string id, CancellationToken ct = default)
    {
        try
        {
            // ตั้ง deadline 10 วินาที
            var deadline = DateTime.UtcNow.AddSeconds(10);
            return await _client.GetProductAsync(
                new GetProductRequest { Id = id },
                deadline: deadline,
                cancellationToken: ct);
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.NotFound)
        {
            return null; // Convert to null instead of throwing
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.DeadlineExceeded)
        {
            throw new TimeoutException($"GetProduct timed out for ID: {id}");
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.Cancelled)
        {
            ct.ThrowIfCancellationRequested();
            throw;
        }
    }

    public async IAsyncEnumerable<Product> StreamProductsAsync(
        SearchProductsRequest request,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
    {
        using var call = _client.SearchProducts(request, cancellationToken: ct);

        await foreach (var product in call.ResponseStream.ReadAllAsync(ct))
        {
            yield return product;
        }
    }
}
```

---

## สรุป Part 68

ใน Part นี้เราได้เรียนรู้ gRPC อย่างครบถ้วน:

| Step | หัวข้อ | Key Takeaways |
|------|--------|---------------|
| 671 | gRPC Overview & Setup | Protocol Buffers, server config, .proto syntax |
| 672 | Unary RPC | Request/response, error handling, deadline |
| 673 | Server Streaming | Stream ข้อมูลจำนวนมาก, real-time updates |
| 674 | Client Streaming | Batch operations, file upload |
| 675 | Bidirectional Streaming | Chat, live inventory management |
| 676 | Client Factory | HttpClientFactory, retry policies |
| 677 | Auth & Interceptors | JWT, logging interceptors |
| 678 | gRPC-Web | Browser support, Blazor WASM |
| 679 | Health Checks & Reflection | Kubernetes integration, grpcurl |
| 680 | Full Example | Production-ready catalog service |

### เปรียบเทียบ gRPC vs REST

| คุณสมบัติ | gRPC | REST |
|-----------|------|------|
| Protocol | HTTP/2 | HTTP/1.1 |
| Format | Binary (Protobuf) | JSON/XML |
| Streaming | Built-in | Limited (SSE, WebSocket) |
| Type Safety | Strongly typed | Weakly typed |
| Browser Support | gRPC-Web | Native |
| Learning Curve | Moderate | Low |
| Performance | Very High | Moderate |

---

**ก่อนหน้า → [Part 67: IoT & Hardware](part67-iot-csharp.md)**
**ต่อไป → [Part 69: GraphQL](part69-graphql.md)**
