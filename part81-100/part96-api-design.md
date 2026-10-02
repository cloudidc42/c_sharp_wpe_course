# Part 96: API Design & OpenAPI
## ขั้นตอนที่ 951-960: API Design

---

## ขั้นตอนที่ 951: RESTful API Design Principles (Richardson Maturity Model, HATEOAS)

### หลักการออกแบบ RESTful API ที่ดี

Richardson Maturity Model (RMM) คือกรอบในการวัดระดับความ "RESTful" ของ API โดยแบ่งออกเป็น 4 ระดับ ตั้งแต่ Level 0 ถึง Level 3

**Level 0 - The Swamp of POX**: ใช้ HTTP เป็นเพียง transport layer ไม่ได้ใช้ประโยชน์จาก HTTP
**Level 1 - Resources**: แบ่ง API เป็น resource ต่างๆ แทนที่จะมี endpoint เดียว
**Level 2 - HTTP Verbs**: ใช้ HTTP methods อย่างถูกต้อง (GET, POST, PUT, DELETE)
**Level 3 - Hypermedia (HATEOAS)**: Response บอก client ว่าสามารถทำอะไรต่อไปได้บ้าง

```csharp
// ตัวอย่าง Level 2 - HTTP Verbs ที่ถูกต้อง
// Program.cs - ASP.NET Core Web API

using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

var app = builder.Build();
app.UseRouting();
app.MapControllers();
app.Run();

// ProductsController.cs
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private static readonly List<Product> _products = new()
    {
        new Product { Id = 1, Name = "Laptop", Price = 25000, Stock = 50 },
        new Product { Id = 2, Name = "Mouse", Price = 500, Stock = 200 },
        new Product { Id = 3, Name = "Keyboard", Price = 1500, Stock = 150 }
    };

    // Level 2: GET /api/products - ดึงรายการสินค้าทั้งหมด
    [HttpGet]
    [ProducesResponseType(typeof(IEnumerable<ProductDto>), StatusCodes.Status200OK)]
    public IActionResult GetAll()
    {
        var dtos = _products.Select(p => ToDto(p));
        return Ok(dtos);
    }

    // Level 2: GET /api/products/{id} - ดึงสินค้าตาม ID
    [HttpGet("{id:int}")]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public IActionResult GetById(int id)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product is null)
            return NotFound(new { Message = $"Product with id {id} not found" });

        return Ok(ToDto(product));
    }

    // Level 2: POST /api/products - สร้างสินค้าใหม่
    [HttpPost]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public IActionResult Create([FromBody] CreateProductRequest request)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        var product = new Product
        {
            Id = _products.Max(p => p.Id) + 1,
            Name = request.Name,
            Price = request.Price,
            Stock = request.InitialStock
        };

        _products.Add(product);

        return CreatedAtAction(nameof(GetById), new { id = product.Id }, ToDto(product));
    }

    // Level 2: PUT /api/products/{id} - อัปเดตสินค้า
    [HttpPut("{id:int}")]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public IActionResult Update(int id, [FromBody] UpdateProductRequest request)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product is null)
            return NotFound();

        product.Name = request.Name;
        product.Price = request.Price;
        product.Stock = request.Stock;

        return Ok(ToDto(product));
    }

    // Level 2: DELETE /api/products/{id} - ลบสินค้า
    [HttpDelete("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public IActionResult Delete(int id)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product is null)
            return NotFound();

        _products.Remove(product);
        return NoContent();
    }

    // Level 3: HATEOAS - Response ที่มี links บอก client ว่าทำอะไรได้บ้าง
    private ProductDto ToDto(Product product) => new ProductDto
    {
        Id = product.Id,
        Name = product.Name,
        Price = product.Price,
        Stock = product.Stock,
        Links = new List<HateoasLink>
        {
            new HateoasLink
            {
                Rel = "self",
                Href = Url.Action(nameof(GetById), new { id = product.Id }),
                Method = "GET"
            },
            new HateoasLink
            {
                Rel = "update",
                Href = Url.Action(nameof(Update), new { id = product.Id }),
                Method = "PUT"
            },
            new HateoasLink
            {
                Rel = "delete",
                Href = Url.Action(nameof(Delete), new { id = product.Id }),
                Method = "DELETE"
            }
        }
    };
}

// Models
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Stock { get; set; }
}

public class ProductDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public List<HateoasLink> Links { get; set; } = new();
}

public class HateoasLink
{
    public string Rel { get; set; } = string.Empty;
    public string? Href { get; set; }
    public string Method { get; set; } = string.Empty;
}

public record CreateProductRequest(
    [Required][MaxLength(200)] string Name,
    [Range(0.01, double.MaxValue)] decimal Price,
    [Range(0, int.MaxValue)] int InitialStock
);

public record UpdateProductRequest(
    [Required][MaxLength(200)] string Name,
    [Range(0.01, double.MaxValue)] decimal Price,
    [Range(0, int.MaxValue)] int Stock
);
```

```json
// ตัวอย่าง HATEOAS Response ที่ได้จาก GET /api/products/1
{
  "id": 1,
  "name": "Laptop",
  "price": 25000,
  "stock": 50,
  "links": [
    {
      "rel": "self",
      "href": "/api/products/1",
      "method": "GET"
    },
    {
      "rel": "update",
      "href": "/api/products/1",
      "method": "PUT"
    },
    {
      "rel": "delete",
      "href": "/api/products/1",
      "method": "DELETE"
    }
  ]
}
```

### HTTP Status Codes ที่ควรใช้

| Status Code | ความหมาย | เมื่อไหร่ควรใช้ |
|-------------|----------|----------------|
| 200 OK | สำเร็จ | GET, PUT ที่สำเร็จ |
| 201 Created | สร้างสำเร็จ | POST ที่สร้าง resource ใหม่ |
| 204 No Content | สำเร็จไม่มีข้อมูลตอบ | DELETE ที่สำเร็จ |
| 400 Bad Request | ข้อมูลผิด | Validation error |
| 401 Unauthorized | ไม่ได้ authenticate | ต้อง login ก่อน |
| 403 Forbidden | ไม่มีสิทธิ์ | ไม่มีสิทธิ์เข้าถึง |
| 404 Not Found | ไม่พบ | Resource ไม่มี |
| 409 Conflict | ขัดแย้ง | Duplicate data |
| 422 Unprocessable | ข้อมูลผิดรูปแบบ | Business rule violation |
| 500 Server Error | ข้อผิดพลาดเซิร์ฟเวอร์ | Unexpected error |

---

## ขั้นตอนที่ 952: OpenAPI กับ Swashbuckle - การตั้งค่าแบบสมบูรณ์

### การติดตั้งและตั้งค่า Swashbuckle

Swashbuckle เป็น library ที่ช่วยสร้าง OpenAPI specification และ Swagger UI สำหรับ ASP.NET Core

```bash
# ติดตั้ง packages
dotnet add package Swashbuckle.AspNetCore
dotnet add package Swashbuckle.AspNetCore.Annotations
dotnet add package Swashbuckle.AspNetCore.Filters
```

```csharp
// Program.cs - การตั้งค่า Swashbuckle แบบสมบูรณ์
using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.Filters;
using System.Reflection;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

// ตั้งค่า Swagger/OpenAPI แบบละเอียด
builder.Services.AddSwaggerGen(options =>
{
    // ข้อมูลพื้นฐาน
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Product Management API",
        Version = "v1",
        Description = "API สำหรับจัดการสินค้าในระบบ E-commerce",
        Contact = new OpenApiContact
        {
            Name = "Development Team",
            Email = "dev@company.com",
            Url = new Uri("https://company.com")
        },
        License = new OpenApiLicense
        {
            Name = "MIT License",
            Url = new Uri("https://opensource.org/licenses/MIT")
        },
        TermsOfService = new Uri("https://company.com/terms")
    });

    // รวม XML documentation
    var xmlFilename = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFilename);
    if (File.Exists(xmlPath))
        options.IncludeXmlComments(xmlPath);

    // เพิ่ม Security Definition สำหรับ JWT
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT",
        In = ParameterLocation.Header,
        Description = "ใส่ JWT token ในรูปแบบ: Bearer {token}"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });

    // ใช้ Operation Filters
    options.OperationFilter<AddResponseHeadersFilter>();
    options.OperationFilter<AppendAuthorizeToSummaryOperationFilter>();
    options.OperationFilter<SecurityRequirementsOperationFilter>();

    // ตั้งค่า Custom Filters
    options.OperationFilter<DeprecatedOperationFilter>();

    // เรียงลำดับ endpoints ตามชื่อ
    options.OrderActionsBy(a => $"{a.ActionDescriptor.RouteValues["controller"]}_{a.HttpMethod}");
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger(options =>
    {
        options.SerializeAsV2 = false; // ใช้ OpenAPI 3.0
    });

    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "Product API v1");
        options.RoutePrefix = "docs"; // เข้าถึงที่ /docs
        options.DocumentTitle = "Product API Documentation";
        options.DefaultModelsExpandDepth(2);
        options.DefaultModelRendering(Swashbuckle.AspNetCore.SwaggerUI.ModelRendering.Model);
        options.DisplayRequestDuration();
        options.EnableFilter();
        options.EnableDeepLinking();
    });
}

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();

// Custom Operation Filter สำหรับ endpoints ที่ถูก deprecated
public class DeprecatedOperationFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        var deprecatedAttr = context.MethodInfo
            .GetCustomAttributes(true)
            .OfType<ObsoleteAttribute>()
            .FirstOrDefault();

        if (deprecatedAttr != null)
        {
            operation.Deprecated = true;
            operation.Description = $"⚠️ Deprecated: {deprecatedAttr.Message}\n\n{operation.Description}";
        }
    }
}
```

```csharp
// Controllers/ProductsController.cs - การใช้ XML Documentation และ Annotations
using Microsoft.AspNetCore.Mvc;
using Swashbuckle.AspNetCore.Annotations;

/// <summary>
/// Controller สำหรับจัดการสินค้า
/// </summary>
[ApiController]
[Route("api/v1/[controller]")]
[Produces("application/json")]
[SwaggerTag("Products", "การจัดการสินค้าในระบบ")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;

    public ProductsController(IProductService productService)
    {
        _productService = productService;
    }

    /// <summary>
    /// ดึงรายการสินค้าทั้งหมด
    /// </summary>
    /// <param name="page">หน้าที่ต้องการ (เริ่มจาก 1)</param>
    /// <param name="pageSize">จำนวนรายการต่อหน้า (สูงสุด 100)</param>
    /// <param name="search">คำค้นหาชื่อสินค้า</param>
    /// <returns>รายการสินค้าพร้อมข้อมูล pagination</returns>
    /// <response code="200">ดึงข้อมูลสำเร็จ</response>
    /// <response code="400">ข้อมูลที่ส่งมาไม่ถูกต้อง</response>
    [HttpGet]
    [SwaggerOperation(
        Summary = "ดึงรายการสินค้าทั้งหมด",
        Description = "ดึงรายการสินค้าพร้อมระบบ pagination และการค้นหา",
        OperationId = "GetProducts",
        Tags = new[] { "Products" }
    )]
    [SwaggerResponse(200, "สำเร็จ", typeof(PagedResponse<ProductDto>))]
    [SwaggerResponse(400, "ข้อมูลผิด", typeof(ValidationProblemDetails))]
    [ProducesResponseType(typeof(PagedResponse<ProductDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> GetAll(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20,
        [FromQuery] string? search = null)
    {
        if (page < 1 || pageSize < 1 || pageSize > 100)
            return BadRequest(new { Message = "Invalid pagination parameters" });

        var result = await _productService.GetPagedAsync(page, pageSize, search);
        return Ok(result);
    }

    /// <summary>
    /// ดึงข้อมูลสินค้าตาม ID
    /// </summary>
    /// <param name="id">ID ของสินค้า</param>
    /// <returns>ข้อมูลสินค้า</returns>
    [HttpGet("{id:int}", Name = "GetProductById")]
    [SwaggerOperation(OperationId = "GetProductById")]
    public async Task<IActionResult> GetById(
        [SwaggerParameter("ID ของสินค้า", Required = true)] int id)
    {
        var product = await _productService.GetByIdAsync(id);
        if (product is null)
            return NotFound(new ProblemDetails
            {
                Title = "Product Not Found",
                Detail = $"ไม่พบสินค้า ID: {id}",
                Status = StatusCodes.Status404NotFound
            });

        return Ok(product);
    }
}

// Response Models
public class PagedResponse<T>
{
    /// <summary>รายการข้อมูล</summary>
    public IEnumerable<T> Items { get; set; } = Enumerable.Empty<T>();

    /// <summary>จำนวนรายการทั้งหมด</summary>
    public int TotalCount { get; set; }

    /// <summary>หน้าปัจจุบัน</summary>
    public int Page { get; set; }

    /// <summary>จำนวนรายการต่อหน้า</summary>
    public int PageSize { get; set; }

    /// <summary>จำนวนหน้าทั้งหมด</summary>
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
}
```

```xml
<!-- ProjectName.csproj - เปิด XML documentation -->
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);1591</NoWarn>
</PropertyGroup>
```

---

## ขั้นตอนที่ 953: API Versioning Strategies

### การทำ Versioning ด้วย Asp.Versioning

API Versioning มี 3 วิธีหลัก: URL, Header, และ Query String

```bash
dotnet add package Asp.Versioning.Mvc
dotnet add package Asp.Versioning.Mvc.ApiExplorer
```

```csharp
// Program.cs - ตั้งค่า API Versioning
using Asp.Versioning;
using Asp.Versioning.ApiExplorer;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// ตั้งค่า API Versioning
builder.Services.AddApiVersioning(options =>
{
    // Default version เมื่อไม่ระบุ
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;

    // รายงาน API versions ใน response headers
    options.ReportApiVersions = true;

    // รองรับทุกรูปแบบ versioning
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),           // /api/v1/products
        new HeaderApiVersionReader("X-Api-Version"), // Header: X-Api-Version: 1.0
        new QueryStringApiVersionReader("api-version") // ?api-version=1.0
    );
})
.AddApiExplorer(options =>
{
    // Format: 'v'major[.minor] เช่น v1, v1.1, v2
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

builder.Services.AddSwaggerGen();

// ต้องสร้าง ConfigureSwaggerOptions
builder.Services.ConfigureOptions<ConfigureSwaggerOptions>();

var app = builder.Build();

var apiVersionProvider = app.Services.GetRequiredService<IApiVersionDescriptionProvider>();

app.UseSwagger();
app.UseSwaggerUI(options =>
{
    foreach (var description in apiVersionProvider.ApiVersionDescriptions)
    {
        options.SwaggerEndpoint(
            $"/swagger/{description.GroupName}/swagger.json",
            $"Products API {description.ApiVersion}");
    }
});

app.UseRouting();
app.MapControllers();
app.Run();

// ConfigureSwaggerOptions.cs
using Asp.Versioning.ApiExplorer;
using Microsoft.Extensions.Options;
using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;

public class ConfigureSwaggerOptions : IConfigureOptions<SwaggerGenOptions>
{
    private readonly IApiVersionDescriptionProvider _provider;

    public ConfigureSwaggerOptions(IApiVersionDescriptionProvider provider)
    {
        _provider = provider;
    }

    public void Configure(SwaggerGenOptions options)
    {
        foreach (var description in _provider.ApiVersionDescriptions)
        {
            options.SwaggerDoc(description.GroupName, CreateInfoForApiVersion(description));
        }
    }

    private static OpenApiInfo CreateInfoForApiVersion(ApiVersionDescription description)
    {
        var info = new OpenApiInfo
        {
            Title = "Product API",
            Version = description.ApiVersion.ToString(),
            Description = "API สำหรับจัดการสินค้า"
        };

        if (description.IsDeprecated)
            info.Description += " ⚠️ **Version นี้ถูก deprecated แล้ว**";

        return info;
    }
}
```

```csharp
// Controllers/V1/ProductsController.cs
using Asp.Versioning;

namespace MyApi.Controllers.V1;

[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public IActionResult GetAll()
    {
        return Ok(new
        {
            Version = "1.0",
            Products = new[]
            {
                new { Id = 1, Name = "Product A", Price = 100 }
            }
        });
    }
}

// Controllers/V2/ProductsController.cs
namespace MyApi.Controllers.V2;

[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("2.0")]
    public IActionResult GetAll()
    {
        // V2 มี features เพิ่มเติม
        return Ok(new
        {
            Version = "2.0",
            Products = new[]
            {
                new
                {
                    Id = 1,
                    Name = "Product A",
                    Price = 100,
                    Category = "Electronics", // ฟีเจอร์ใหม่ใน V2
                    Tags = new[] { "new", "sale" } // ฟีเจอร์ใหม่ใน V2
                }
            },
            Meta = new
            {
                TotalCount = 1,
                Page = 1
            }
        });
    }

    [HttpGet("deprecated-endpoint")]
    [MapToApiVersion("2.0")]
    [Obsolete("ใช้ GetAll แทน")]
    public IActionResult OldEndpoint()
    {
        return Ok("This endpoint will be removed in v3");
    }
}

// ตัวอย่างการเรียกใช้ 3 รูปแบบ:
// 1. URL: GET /api/v1/products
// 2. Header: GET /api/products + Header: X-Api-Version: 1.0
// 3. Query: GET /api/products?api-version=1.0
```

---

## ขั้นตอนที่ 954: Minimal APIs vs Controller-based APIs

### เปรียบเทียบและ Trade-offs

Minimal APIs (เพิ่มมาใน .NET 6) เหมาะกับ API ที่เรียบง่าย ส่วน Controller-based เหมาะกับ API ที่ซับซ้อน

```csharp
// Minimal API - เหมาะกับ microservices ขนาดเล็ก
// Program.cs
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddSingleton<IProductRepository, InMemoryProductRepository>();

var app = builder.Build();
app.UseSwagger();
app.UseSwaggerUI();

// จัดกลุ่ม endpoints
var productsGroup = app.MapGroup("/api/products")
    .WithTags("Products")
    .WithOpenApi();

// GET /api/products
productsGroup.MapGet("/", async (
    IProductRepository repo,
    int page = 1,
    int pageSize = 20) =>
{
    var products = await repo.GetAllAsync(page, pageSize);
    return Results.Ok(products);
})
.WithName("GetProducts")
.WithSummary("ดึงรายการสินค้า")
.WithDescription("ดึงรายการสินค้าทั้งหมดพร้อม pagination")
.Produces<IEnumerable<Product>>(200)
.Produces<ProblemDetails>(400);

// GET /api/products/{id}
productsGroup.MapGet("/{id:int}", async (
    int id,
    IProductRepository repo) =>
{
    var product = await repo.GetByIdAsync(id);
    return product is not null
        ? Results.Ok(product)
        : Results.NotFound(new { Message = $"Product {id} not found" });
})
.WithName("GetProductById")
.Produces<Product>(200)
.Produces(404);

// POST /api/products
productsGroup.MapPost("/", async (
    CreateProductRequest request,
    IProductRepository repo) =>
{
    if (string.IsNullOrWhiteSpace(request.Name))
        return Results.BadRequest(new { Message = "Name is required" });

    var product = await repo.CreateAsync(request);
    return Results.CreatedAtRoute("GetProductById", new { id = product.Id }, product);
})
.WithName("CreateProduct")
.Produces<Product>(201)
.Produces(400);

// PUT /api/products/{id}
productsGroup.MapPut("/{id:int}", async (
    int id,
    UpdateProductRequest request,
    IProductRepository repo) =>
{
    var updated = await repo.UpdateAsync(id, request);
    return updated is not null
        ? Results.Ok(updated)
        : Results.NotFound();
})
.WithName("UpdateProduct");

// DELETE /api/products/{id}
productsGroup.MapDelete("/{id:int}", async (
    int id,
    IProductRepository repo) =>
{
    var deleted = await repo.DeleteAsync(id);
    return deleted ? Results.NoContent() : Results.NotFound();
})
.WithName("DeleteProduct");

app.Run();

// Repository Interface
public interface IProductRepository
{
    Task<IEnumerable<Product>> GetAllAsync(int page, int pageSize);
    Task<Product?> GetByIdAsync(int id);
    Task<Product> CreateAsync(CreateProductRequest request);
    Task<Product?> UpdateAsync(int id, UpdateProductRequest request);
    Task<bool> DeleteAsync(int id);
}

// InMemory Implementation
public class InMemoryProductRepository : IProductRepository
{
    private readonly List<Product> _products = new()
    {
        new Product { Id = 1, Name = "Laptop", Price = 25000 },
        new Product { Id = 2, Name = "Mouse", Price = 500 }
    };

    public Task<IEnumerable<Product>> GetAllAsync(int page, int pageSize)
    {
        var result = _products
            .Skip((page - 1) * pageSize)
            .Take(pageSize);
        return Task.FromResult(result);
    }

    public Task<Product?> GetByIdAsync(int id)
        => Task.FromResult(_products.FirstOrDefault(p => p.Id == id));

    public Task<Product> CreateAsync(CreateProductRequest request)
    {
        var product = new Product
        {
            Id = _products.Any() ? _products.Max(p => p.Id) + 1 : 1,
            Name = request.Name,
            Price = request.Price
        };
        _products.Add(product);
        return Task.FromResult(product);
    }

    public Task<Product?> UpdateAsync(int id, UpdateProductRequest request)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product is null) return Task.FromResult<Product?>(null);

        product.Name = request.Name;
        product.Price = request.Price;
        return Task.FromResult<Product?>(product);
    }

    public Task<bool> DeleteAsync(int id)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product is null) return Task.FromResult(false);

        _products.Remove(product);
        return Task.FromResult(true);
    }
}
```

### เปรียบเทียบ Minimal APIs vs Controllers

| คุณสมบัติ | Minimal APIs | Controller-based |
|-----------|-------------|-----------------|
| Code ที่ต้องเขียน | น้อยกว่า | มากกว่า |
| การจัดระเบียบ | ยากเมื่อโต | มีโครงสร้างชัดเจน |
| Middleware | ต้องกำหนดเอง | มี built-in |
| Model Binding | รองรับ | รองรับครบถ้วน |
| Filter | จำกัด | ครบถ้วน |
| Unit Testing | ยากกว่า | ง่ายกว่า |
| เหมาะกับ | Microservices เล็กๆ | Enterprise APIs |

---

## ขั้นตอนที่ 955: API Gateway Patterns

### Backend for Frontend (BFF), Aggregation, และ Transformation

```csharp
// BFF Pattern - สร้าง gateway เฉพาะสำหรับ frontend
// ApiGateway/Program.cs
using System.Net.Http.Json;

var builder = WebApplication.CreateBuilder(args);

// ลงทะเบียน HTTP clients สำหรับ microservices ต่างๆ
builder.Services.AddHttpClient("ProductService", client =>
{
    client.BaseAddress = new Uri("http://product-service:8080/");
    client.Timeout = TimeSpan.FromSeconds(30);
});

builder.Services.AddHttpClient("UserService", client =>
{
    client.BaseAddress = new Uri("http://user-service:8080/");
    client.Timeout = TimeSpan.FromSeconds(30);
});

builder.Services.AddHttpClient("OrderService", client =>
{
    client.BaseAddress = new Uri("http://order-service:8080/");
    client.Timeout = TimeSpan.FromSeconds(30);
});

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();
app.UseSwagger();
app.UseSwaggerUI();

// BFF Endpoint: ดึงข้อมูล Dashboard สำหรับ Mobile App
// Aggregation: รวมข้อมูลจากหลาย microservices ในครั้งเดียว
app.MapGet("/mobile/dashboard/{userId}", async (
    int userId,
    IHttpClientFactory clientFactory) =>
{
    var productClient = clientFactory.CreateClient("ProductService");
    var userClient = clientFactory.CreateClient("UserService");
    var orderClient = clientFactory.CreateClient("OrderService");

    // ดึงข้อมูลพร้อมกัน (Parallel)
    var userTask = userClient.GetFromJsonAsync<UserInfo>($"users/{userId}");
    var ordersTask = orderClient.GetFromJsonAsync<List<OrderSummary>>($"orders/user/{userId}/recent");
    var recommendedTask = productClient.GetFromJsonAsync<List<ProductSummary>>($"products/recommended/{userId}");

    await Task.WhenAll(userTask, ordersTask, recommendedTask);

    // Transformation: แปลงข้อมูลให้เหมาะกับ mobile
    return Results.Ok(new MobileDashboardResponse
    {
        User = userTask.Result,
        RecentOrders = ordersTask.Result?.Take(5).Select(o => new
        {
            o.Id,
            o.Status,
            // แปลงวันที่เป็น relative time
            PlacedAgo = GetRelativeTime(o.PlacedAt),
            o.TotalAmount
        }),
        RecommendedProducts = recommendedTask.Result?.Take(4).Select(p => new
        {
            p.Id,
            p.Name,
            // Transformation: ราคาเป็น string พร้อมสกุลเงิน
            PriceDisplay = $"฿{p.Price:N0}",
            p.ImageUrl
        })
    });
})
.WithTags("BFF-Mobile")
.WithSummary("ดึงข้อมูล Dashboard สำหรับ Mobile");

// BFF Endpoint สำหรับ Web App - ข้อมูลละเอียดกว่า
app.MapGet("/web/dashboard/{userId}", async (
    int userId,
    IHttpClientFactory clientFactory) =>
{
    var productClient = clientFactory.CreateClient("ProductService");
    var userClient = clientFactory.CreateClient("UserService");
    var orderClient = clientFactory.CreateClient("OrderService");

    var userTask = userClient.GetFromJsonAsync<UserInfo>($"users/{userId}");
    var ordersTask = orderClient.GetFromJsonAsync<List<OrderDetail>>($"orders/user/{userId}");
    var statsTask = orderClient.GetFromJsonAsync<UserOrderStats>($"orders/user/{userId}/stats");

    await Task.WhenAll(userTask, ordersTask, statsTask);

    return Results.Ok(new WebDashboardResponse
    {
        User = userTask.Result,
        Orders = ordersTask.Result,
        Statistics = statsTask.Result
    });
})
.WithTags("BFF-Web")
.WithSummary("ดึงข้อมูล Dashboard สำหรับ Web");

app.Run();

// Helper method
static string GetRelativeTime(DateTime dateTime)
{
    var diff = DateTime.UtcNow - dateTime;
    return diff.TotalDays >= 1
        ? $"{(int)diff.TotalDays} วันที่แล้ว"
        : diff.TotalHours >= 1
            ? $"{(int)diff.TotalHours} ชั่วโมงที่แล้ว"
            : $"{(int)diff.TotalMinutes} นาทีที่แล้ว";
}

// DTOs
public record UserInfo(int Id, string Name, string Email, string AvatarUrl);
public record OrderSummary(int Id, string Status, DateTime PlacedAt, decimal TotalAmount);
public record OrderDetail(int Id, string Status, DateTime PlacedAt, decimal TotalAmount, List<OrderItem> Items);
public record OrderItem(int ProductId, string ProductName, int Quantity, decimal Price);
public record ProductSummary(int Id, string Name, decimal Price, string ImageUrl);
public record UserOrderStats(int TotalOrders, decimal TotalSpent, int PendingOrders);

public class MobileDashboardResponse
{
    public UserInfo? User { get; set; }
    public IEnumerable<object>? RecentOrders { get; set; }
    public IEnumerable<object>? RecommendedProducts { get; set; }
}

public class WebDashboardResponse
{
    public UserInfo? User { get; set; }
    public List<OrderDetail>? Orders { get; set; }
    public UserOrderStats? Statistics { get; set; }
}
```

```csharp
// YARP (Yet Another Reverse Proxy) - สำหรับ API Gateway ที่ซับซ้อน
// Program.cs สำหรับ YARP Gateway

// dotnet add package Yarp.ReverseProxy

using Yarp.ReverseProxy.Transforms;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddTransforms(builderContext =>
    {
        // เพิ่ม Transform สำหรับ request
        builderContext.AddRequestTransform(async context =>
        {
            // เพิ่ม Correlation ID
            context.ProxyRequest.Headers.Add(
                "X-Correlation-Id",
                Guid.NewGuid().ToString());

            // Forward user ID จาก JWT
            if (context.HttpContext.User.Identity?.IsAuthenticated == true)
            {
                var userId = context.HttpContext.User.FindFirst("sub")?.Value;
                if (userId != null)
                    context.ProxyRequest.Headers.Add("X-User-Id", userId);
            }
        });

        // เพิ่ม Transform สำหรับ response
        builderContext.AddResponseTransform(async context =>
        {
            // เพิ่ม timing header
            context.HttpContext.Response.Headers.Add(
                "X-Response-Time",
                DateTime.UtcNow.ToString("O"));
        });
    });

var app = builder.Build();
app.MapReverseProxy();
app.Run();
```

```json
// appsettings.json - ตั้งค่า YARP
{
  "ReverseProxy": {
    "Routes": {
      "product-route": {
        "ClusterId": "product-cluster",
        "Match": {
          "Path": "/api/products/{**catch-all}"
        }
      },
      "user-route": {
        "ClusterId": "user-cluster",
        "Match": {
          "Path": "/api/users/{**catch-all}"
        }
      }
    },
    "Clusters": {
      "product-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "http://product-service:8080/"
          }
        }
      },
      "user-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "http://user-service:8080/"
          }
        }
      }
    }
  }
}
```

---

## ขั้นตอนที่ 956: GraphQL vs REST vs gRPC - คู่มือการตัดสินใจ

### เปรียบเทียบ 3 เทคโนโลยี

```csharp
// REST API - เหมาะกับ public APIs ที่ต้องการความเรียบง่าย
[ApiController]
[Route("api/[controller]")]
public class OrdersRestController : ControllerBase
{
    // REST: Over-fetching - ดึงข้อมูลมากกว่าที่ต้องการ
    // Client อาจต้องการแค่ orderId และ status แต่ได้ข้อมูลทั้งหมด
    [HttpGet("{id}")]
    public IActionResult GetOrder(int id)
    {
        return Ok(new
        {
            Id = id,
            Status = "Delivered",
            CreatedAt = DateTime.UtcNow.AddDays(-3),
            UpdatedAt = DateTime.UtcNow.AddDays(-1),
            Customer = new { Id = 1, Name = "สมชาย ใจดี", Email = "somchai@example.com" },
            Items = new[]
            {
                new { ProductId = 1, Name = "Laptop", Quantity = 1, Price = 25000 },
                new { ProductId = 2, Name = "Mouse", Quantity = 2, Price = 500 }
            },
            Shipping = new { Address = "123 ถนนสุขุมวิท", City = "กรุงเทพ" },
            Payment = new { Method = "Credit Card", Amount = 26000 }
        });
    }
}
```

```csharp
// GraphQL - เหมาะกับ complex queries ที่ต้องการ flexibility
// dotnet add package HotChocolate.AspNetCore

using HotChocolate.Types;

// GraphQL Schema
public class Query
{
    // GraphQL: Exact-fetching - client เลือกเองว่าอยากได้อะไร
    public async Task<Order?> GetOrderAsync(int id, [Service] IOrderRepository repo)
        => await repo.GetByIdAsync(id);

    public async Task<IEnumerable<Order>> GetOrdersAsync(
        [Service] IOrderRepository repo,
        int? customerId = null,
        string? status = null)
        => await repo.GetAllAsync(customerId, status);
}

public class Mutation
{
    public async Task<Order> CreateOrderAsync(
        CreateOrderInput input,
        [Service] IOrderRepository repo)
        => await repo.CreateAsync(input);
}

public class Subscription
{
    [Subscribe]
    [Topic("order-updated")]
    public Order OnOrderUpdated([EventMessage] Order order)
        => order;
}

[ObjectType]
public class OrderType : ObjectType<Order>
{
    protected override void Configure(IObjectTypeDescriptor<Order> descriptor)
    {
        descriptor.Field(o => o.Id).Type<NonNullType<IntType>>();
        descriptor.Field(o => o.Status).Type<NonNullType<StringType>>();
        descriptor.Field(o => o.Customer)
            .ResolveWith<CustomerResolver>(r => r.GetCustomerAsync(default!, default!));
        descriptor.Field(o => o.Items)
            .ResolveWith<OrderItemsResolver>(r => r.GetItemsAsync(default!, default!));
    }
}

// GraphQL Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddSubscriptionType<Subscription>()
    .AddType<OrderType>()
    .AddFiltering()
    .AddSorting()
    .AddProjections()
    .AddInMemorySubscriptions();

var app = builder.Build();
app.UseWebSockets();
app.MapGraphQL("/graphql");
app.Run();
```

```csharp
// gRPC - เหมาะกับ internal microservices ที่ต้องการ performance สูง
// dotnet add package Grpc.AspNetCore

// Protos/order.proto
/*
syntax = "proto3";
option csharp_namespace = "GrpcOrderService";

package order;

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (OrderResponse);
  rpc CreateOrder(CreateOrderRequest) returns (OrderResponse);
  rpc UpdateOrderStatus(UpdateStatusRequest) returns (OrderResponse);
  rpc StreamOrders(StreamOrdersRequest) returns (stream OrderResponse);
}

message GetOrderRequest {
  int32 id = 1;
}

message CreateOrderRequest {
  int32 customer_id = 1;
  repeated OrderItem items = 2;
}

message UpdateStatusRequest {
  int32 order_id = 1;
  string status = 2;
}

message StreamOrdersRequest {
  int32 customer_id = 1;
}

message OrderResponse {
  int32 id = 1;
  string status = 2;
  int32 customer_id = 3;
  double total_amount = 4;
  string created_at = 5;
}

message OrderItem {
  int32 product_id = 1;
  int32 quantity = 2;
  double price = 3;
}
*/

// gRPC Service Implementation
using Grpc.Core;
using GrpcOrderService;

public class OrderGrpcService : OrderService.OrderServiceBase
{
    private readonly IOrderRepository _repository;

    public OrderGrpcService(IOrderRepository repository)
    {
        _repository = repository;
    }

    public override async Task<OrderResponse> GetOrder(
        GetOrderRequest request,
        ServerCallContext context)
    {
        var order = await _repository.GetByIdAsync(request.Id);
        if (order is null)
            throw new RpcException(new Status(StatusCode.NotFound, $"Order {request.Id} not found"));

        return MapToResponse(order);
    }

    // Server Streaming - ส่ง updates แบบ real-time
    public override async Task StreamOrders(
        StreamOrdersRequest request,
        IServerStreamWriter<OrderResponse> responseStream,
        ServerCallContext context)
    {
        while (!context.CancellationToken.IsCancellationRequested)
        {
            var orders = await _repository.GetByCustomerIdAsync(request.CustomerId);
            foreach (var order in orders)
            {
                await responseStream.WriteAsync(MapToResponse(order));
            }

            await Task.Delay(TimeSpan.FromSeconds(5), context.CancellationToken);
        }
    }

    private static OrderResponse MapToResponse(Order order) => new()
    {
        Id = order.Id,
        Status = order.Status,
        CustomerId = order.CustomerId,
        TotalAmount = (double)order.TotalAmount,
        CreatedAt = order.CreatedAt.ToString("O")
    };
}

// gRPC Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddGrpc();
builder.Services.AddGrpcReflection(); // สำหรับ development

var app = builder.Build();
app.MapGrpcService<OrderGrpcService>();
app.MapGrpcReflectionService();
app.Run();
```

### คู่มือเลือกเทคโนโลยี

| เกณฑ์ | REST | GraphQL | gRPC |
|-------|------|---------|------|
| Public API | ✅ ดีที่สุด | ✅ ดี | ❌ ไม่แนะนำ |
| Internal Services | ✅ ดี | ✅ ดี | ✅ ดีที่สุด |
| Performance | ✅ ดี | ⚠️ ปานกลาง | ✅ ดีที่สุด |
| Flexibility | ⚠️ จำกัด | ✅ สูง | ❌ ต่ำ |
| Browser Support | ✅ ดีที่สุด | ✅ ดี | ⚠️ ต้องใช้ grpc-web |
| Learning Curve | ✅ ง่าย | ⚠️ ปานกลาง | ⚠️ สูง |
| Type Safety | ⚠️ ต้องระวัง | ✅ ดี | ✅ ดีที่สุด |
| Streaming | ❌ จำกัด | ✅ Subscriptions | ✅ Built-in |

---

## ขั้นตอนที่ 957: OpenAPI Code Generation สำหรับ Client

### NSwag และ Kiota

```bash
# ติดตั้ง NSwag CLI
dotnet tool install -g NSwag.ConsoleX

# ติดตั้ง Microsoft Kiota
dotnet tool install --global Microsoft.OpenApi.Kiota
```

```csharp
// nswag.json - NSwag configuration
/*
{
  "runtime": "Net80",
  "defaultVariables": null,
  "documentGenerator": {
    "aspNetCoreToOpenApi": {
      "project": "../MyApi/MyApi.csproj",
      "documentName": "v1",
      "output": "swagger.json",
      "outputType": "OpenApi3"
    }
  },
  "codeGenerators": {
    "openApiToCSharpClient": {
      "clientBaseClass": "BaseApiClient",
      "generateClientClasses": true,
      "generateDtoTypes": true,
      "injectHttpClient": true,
      "namespace": "MyApp.ApiClient",
      "className": "{controller}Client",
      "generateExceptionClasses": true,
      "output": "../MyApp.Client/ApiClient.cs"
    }
  }
}
*/

// สั่ง generate client
// nswag run nswag.json

// ตัวอย่าง Generated Client ที่ได้จาก NSwag
public partial class ProductsClient : BaseApiClient
{
    private readonly HttpClient _httpClient;

    public ProductsClient(string baseUrl, HttpClient httpClient) : base(baseUrl)
    {
        _httpClient = httpClient;
    }

    public async Task<ICollection<ProductDto>> GetAllAsync(
        int page = 1,
        int pageSize = 20,
        string? search = null,
        CancellationToken cancellationToken = default)
    {
        var urlBuilder = new StringBuilder();
        urlBuilder.Append(BaseUrl.TrimEnd('/')).Append("/api/v1/products?");
        urlBuilder.Append($"page={page}&");
        urlBuilder.Append($"pageSize={pageSize}&");
        if (search != null) urlBuilder.Append($"search={Uri.EscapeDataString(search)}&");

        var response = await _httpClient.GetAsync(urlBuilder.ToString(), cancellationToken);
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadFromJsonAsync<ICollection<ProductDto>>(
            cancellationToken: cancellationToken) ?? new List<ProductDto>();
    }

    public async Task<ProductDto> GetByIdAsync(int id, CancellationToken cancellationToken = default)
    {
        var response = await _httpClient.GetAsync(
            $"{BaseUrl.TrimEnd('/')}/api/v1/products/{id}",
            cancellationToken);

        if (response.StatusCode == System.Net.HttpStatusCode.NotFound)
            throw new ApiException("Product not found", 404, null, null, null);

        response.EnsureSuccessStatusCode();
        return (await response.Content.ReadFromJsonAsync<ProductDto>(
            cancellationToken: cancellationToken))!;
    }
}
```

```csharp
// Kiota - Microsoft's code generation tool
// สั่ง generate:
// kiota generate -l CSharp -d https://api.example.com/swagger/v1/swagger.json
//                -o ./KiotaClient -n MyApp.ApiClient

// ตัวอย่าง Kiota generated client usage
using Microsoft.Kiota.Http.HttpClientLibrary;
using Microsoft.Kiota.Authentication.Azure;
using MyApp.ApiClient;

// สร้าง authentication provider
var credential = new DefaultAzureCredential();
var authProvider = new AzureIdentityAuthenticationProvider(credential);

// สร้าง request adapter
var adapter = new HttpClientRequestAdapter(authProvider);

// สร้าง client
var client = new ProductApiClient(adapter);

// ใช้งาน client แบบ strongly-typed
var products = await client.Api.V1.Products.GetAsync(config =>
{
    config.QueryParameters.Page = 1;
    config.QueryParameters.PageSize = 20;
    config.QueryParameters.Search = "laptop";
});

// สร้างสินค้าใหม่
var newProduct = await client.Api.V1.Products.PostAsync(new CreateProductRequest
{
    Name = "Gaming Laptop",
    Price = 45000,
    InitialStock = 10
});

Console.WriteLine($"Created: {newProduct?.Id} - {newProduct?.Name}");

// ดึงสินค้าตาม ID
var product = await client.Api.V1.Products[1].GetAsync();
Console.WriteLine($"Product: {product?.Name} - {product?.Price}");
```

```csharp
// การตั้งค่า NSwag แบบละเอียด (ผ่าน Code)
// nswag.config.cs

using NSwag;
using NSwag.CodeGeneration.CSharp;
using NSwag.CodeGeneration.TypeScript;

public class ApiClientGenerator
{
    public async Task GenerateCSharpClientAsync(string swaggerUrl, string outputPath)
    {
        // โหลด OpenAPI document
        var document = await OpenApiDocument.FromUrlAsync(swaggerUrl);

        // ตั้งค่า C# Generator
        var settings = new CSharpClientGeneratorSettings
        {
            ClassName = "{controller}ApiClient",
            CSharpGeneratorSettings =
            {
                Namespace = "MyApp.GeneratedClient",
                GenerateNullableReferenceTypes = true
            },
            GenerateClientInterfaces = true, // สร้าง Interface ด้วย
            GenerateDtoTypes = true,
            UseBaseUrl = true,
            InjectHttpClient = true,
            HttpClientType = "System.Net.Http.HttpClient",
            ExceptionClass = "ApiException",
            WrapDtoExceptions = true
        };

        var generator = new CSharpClientGenerator(document, settings);
        var code = generator.GenerateFile();

        await File.WriteAllTextAsync(outputPath, code);
        Console.WriteLine($"Generated C# client: {outputPath}");
    }

    public async Task GenerateTypeScriptClientAsync(string swaggerUrl, string outputPath)
    {
        var document = await OpenApiDocument.FromUrlAsync(swaggerUrl);

        var settings = new TypeScriptClientGeneratorSettings
        {
            ClassName = "{controller}ApiClient",
            TypeScriptGeneratorSettings =
            {
                TypeScriptVersion = 4.3m,
                GenerateConstructorInterface = true
            },
            Template = TypeScriptTemplate.Axios,
            PromiseType = PromiseType.Promise
        };

        var generator = new TypeScriptClientGenerator(document, settings);
        var code = generator.GenerateFile();

        await File.WriteAllTextAsync(outputPath, code);
        Console.WriteLine($"Generated TypeScript client: {outputPath}");
    }
}
```

---

## ขั้นตอนที่ 958: API Testing ด้วย RestSharp และ Refit

### RestSharp - HTTP Client Library

```bash
dotnet add package RestSharp
dotnet add package Refit
dotnet add package Refit.HttpClientFactory
```

```csharp
// RestSharp - การทดสอบ API แบบ fluent
using RestSharp;
using RestSharp.Authenticators;

public class ProductApiTests
{
    private readonly RestClient _client;

    public ProductApiTests()
    {
        var options = new RestClientOptions("https://api.example.com")
        {
            Authenticator = new JwtAuthenticator("your-jwt-token"),
            ThrowOnDeserializationError = true,
            MaxTimeout = 30000 // 30 วินาที
        };

        _client = new RestClient(options);
    }

    public async Task TestGetAllProducts()
    {
        var request = new RestRequest("api/v1/products")
            .AddQueryParameter("page", "1")
            .AddQueryParameter("pageSize", "10")
            .AddHeader("Accept", "application/json");

        var response = await _client.GetAsync<PagedResponse<ProductDto>>(request);

        Console.WriteLine($"Total products: {response?.TotalCount}");
        foreach (var product in response?.Items ?? Enumerable.Empty<ProductDto>())
        {
            Console.WriteLine($"  - {product.Id}: {product.Name} = ฿{product.Price:N0}");
        }
    }

    public async Task TestCreateProduct()
    {
        var request = new RestRequest("api/v1/products", Method.Post)
            .AddJsonBody(new
            {
                name = "New Product",
                price = 1500.00m,
                initialStock = 100
            });

        var response = await _client.PostAsync<ProductDto>(request);
        Console.WriteLine($"Created product ID: {response?.Id}");
    }

    public async Task TestUpdateProduct(int id)
    {
        var request = new RestRequest($"api/v1/products/{id}", Method.Put)
            .AddJsonBody(new
            {
                name = "Updated Product",
                price = 2000.00m,
                stock = 80
            });

        var response = await _client.PutAsync<ProductDto>(request);
        Console.WriteLine($"Updated: {response?.Name}");
    }

    public async Task TestDeleteProduct(int id)
    {
        var request = new RestRequest($"api/v1/products/{id}", Method.Delete);
        var response = await _client.DeleteAsync(request);

        Console.WriteLine($"Delete status: {response.StatusCode}");
    }

    // การใช้ RestSharp สำหรับ Integration Testing
    public async Task RunIntegrationTests()
    {
        Console.WriteLine("=== Running Integration Tests ===\n");

        // Test 1: Get all products
        Console.WriteLine("Test 1: Get All Products");
        await TestGetAllProducts();

        // Test 2: Create product
        Console.WriteLine("\nTest 2: Create Product");
        await TestCreateProduct();

        // Test 3: Get product by ID
        Console.WriteLine("\nTest 3: Get Product by ID");
        var getRequest = new RestRequest("api/v1/products/1");
        var product = await _client.GetAsync<ProductDto>(getRequest);
        Console.WriteLine($"Got: {product?.Name}");

        // Test 4: Update product
        Console.WriteLine("\nTest 4: Update Product");
        await TestUpdateProduct(1);

        // Test 5: Upload file (multipart)
        Console.WriteLine("\nTest 5: Upload Product Image");
        var uploadRequest = new RestRequest("api/v1/products/1/image", Method.Post);
        uploadRequest.AddFile("image", "/path/to/image.jpg", "image/jpeg");
        var uploadResponse = await _client.PostAsync(uploadRequest);
        Console.WriteLine($"Upload status: {uploadResponse.StatusCode}");
    }
}
```

```csharp
// Refit - Type-safe HTTP Client (เหมือน Retrofit ใน Java)
using Refit;
using System.ComponentModel.DataAnnotations;

// สร้าง Interface ที่ Refit จะสร้าง implementation ให้อัตโนมัติ
public interface IProductApi
{
    [Get("/api/v1/products")]
    Task<PagedResponse<ProductDto>> GetAllAsync(
        [AliasAs("page")] int page = 1,
        [AliasAs("pageSize")] int pageSize = 20,
        [AliasAs("search")] string? search = null);

    [Get("/api/v1/products/{id}")]
    Task<ProductDto> GetByIdAsync(int id);

    [Post("/api/v1/products")]
    Task<ProductDto> CreateAsync([Body] CreateProductRequest request);

    [Put("/api/v1/products/{id}")]
    Task<ProductDto> UpdateAsync(int id, [Body] UpdateProductRequest request);

    [Delete("/api/v1/products/{id}")]
    Task DeleteAsync(int id);

    [Multipart]
    [Post("/api/v1/products/{id}/image")]
    Task<ImageUploadResponse> UploadImageAsync(
        int id,
        [AliasAs("image")] StreamPart imageStream);

    // Headers
    [Headers("Accept: application/json")]
    [Get("/api/v1/products/export")]
    Task<HttpResponseMessage> ExportAsync();
}

// Order API Interface
public interface IOrderApi
{
    [Get("/api/v1/orders")]
    Task<PagedResponse<OrderDto>> GetAllAsync(
        [AliasAs("status")] string? status = null,
        [AliasAs("from")] DateTime? from = null,
        [AliasAs("to")] DateTime? to = null);

    [Post("/api/v1/orders")]
    Task<ApiResponse<OrderDto>> CreateOrderAsync([Body] CreateOrderRequest request);
}

// Program.cs - Registration
var builder = WebApplication.CreateBuilder(args);

// ลงทะเบียน Refit clients
builder.Services
    .AddRefitClient<IProductApi>(new RefitSettings
    {
        ContentSerializer = new SystemTextJsonContentSerializer(
            new JsonSerializerOptions
            {
                PropertyNamingPolicy = JsonNamingPolicy.CamelCase
            }
        )
    })
    .ConfigureHttpClient(client =>
    {
        client.BaseAddress = new Uri("https://api.example.com");
        client.DefaultRequestHeaders.Add("X-Api-Key", "your-api-key");
    })
    .AddHttpMessageHandler<AuthenticationHandler>(); // เพิ่ม middleware

builder.Services
    .AddRefitClient<IOrderApi>()
    .ConfigureHttpClient(client =>
        client.BaseAddress = new Uri("https://api.example.com"));

var app = builder.Build();

// ตัวอย่างการใช้งาน Refit
app.MapGet("/test-refit", async (IProductApi productApi) =>
{
    try
    {
        var products = await productApi.GetAllAsync(page: 1, pageSize: 5);
        return Results.Ok(products);
    }
    catch (ApiException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
    {
        return Results.NotFound("No products found");
    }
    catch (ApiException ex)
    {
        return Results.Problem($"API Error: {ex.Message}");
    }
});

app.Run();

// Authentication Handler
public class AuthenticationHandler : DelegatingHandler
{
    private readonly ITokenService _tokenService;

    public AuthenticationHandler(ITokenService tokenService)
    {
        _tokenService = tokenService;
    }

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        var token = await _tokenService.GetTokenAsync();
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);

        return await base.SendAsync(request, cancellationToken);
    }
}

// Integration Tests ด้วย Refit
public class RefitIntegrationTests
{
    private readonly IProductApi _productApi;

    public RefitIntegrationTests()
    {
        _productApi = RestService.For<IProductApi>("https://api.example.com", new RefitSettings
        {
            ContentSerializer = new SystemTextJsonContentSerializer()
        });
    }

    public async Task TestCrudOperations()
    {
        // Create
        var created = await _productApi.CreateAsync(new CreateProductRequest("Test Product", 999m, 50));
        Console.WriteLine($"Created ID: {created.Id}");

        // Read
        var fetched = await _productApi.GetByIdAsync(created.Id);
        Console.WriteLine($"Fetched: {fetched.Name}");

        // Update
        var updated = await _productApi.UpdateAsync(created.Id,
            new UpdateProductRequest("Updated Product", 1099m, 45));
        Console.WriteLine($"Updated: {updated.Name} = ฿{updated.Price}");

        // Delete
        await _productApi.DeleteAsync(created.Id);
        Console.WriteLine("Deleted successfully");
    }
}
```

---

## ขั้นตอนที่ 959: Pagination - Offset vs Cursor-based vs Keyset

### 3 รูปแบบของ Pagination

```csharp
// 1. Offset Pagination - วิธีดั้งเดิม แต่มีปัญหาเมื่อข้อมูลมาก
[HttpGet("offset")]
public async Task<IActionResult> GetWithOffset(
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20,
    [FromQuery] string? category = null)
{
    var query = _context.Products.AsQueryable();

    if (!string.IsNullOrEmpty(category))
        query = query.Where(p => p.Category == category);

    var totalCount = await query.CountAsync();

    // ข้อเสีย: OFFSET ใน SQL ช้าเมื่อ offset ใหญ่
    // เช่น OFFSET 10000 LIMIT 20 ต้องอ่าน 10020 rows แล้วทิ้ง 10000
    var items = await query
        .OrderBy(p => p.Id)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(p => new ProductDto { Id = p.Id, Name = p.Name, Price = p.Price })
        .ToListAsync();

    return Ok(new OffsetPaginationResponse<ProductDto>
    {
        Items = items,
        TotalCount = totalCount,
        Page = page,
        PageSize = pageSize,
        TotalPages = (int)Math.Ceiling((double)totalCount / pageSize),
        HasPrevious = page > 1,
        HasNext = page < (int)Math.Ceiling((double)totalCount / pageSize)
    });
}

// 2. Cursor-based Pagination - เหมาะกับ real-time data (เช่น Twitter timeline)
[HttpGet("cursor")]
public async Task<IActionResult> GetWithCursor(
    [FromQuery] string? cursor = null,  // encoded cursor (base64)
    [FromQuery] int limit = 20,
    [FromQuery] string direction = "next")
{
    var query = _context.Products.AsQueryable();

    if (!string.IsNullOrEmpty(cursor))
    {
        // Decode cursor
        var decodedCursor = DecodeCursor(cursor);
        var cursorId = decodedCursor.Id;
        var cursorCreatedAt = decodedCursor.CreatedAt;

        if (direction == "next")
        {
            // ข้อมูลหลัง cursor
            query = query.Where(p =>
                p.CreatedAt > cursorCreatedAt ||
                (p.CreatedAt == cursorCreatedAt && p.Id > cursorId));
        }
        else
        {
            // ข้อมูลก่อน cursor (ย้อนกลับ)
            query = query.Where(p =>
                p.CreatedAt < cursorCreatedAt ||
                (p.CreatedAt == cursorCreatedAt && p.Id < cursorId));
        }
    }

    var items = await query
        .OrderBy(p => p.CreatedAt)
        .ThenBy(p => p.Id)
        .Take(limit + 1) // ดึงเพิ่ม 1 เพื่อเช็คว่ามีหน้าถัดไปหรือไม่
        .Select(p => new ProductDto { Id = p.Id, Name = p.Name, Price = p.Price, CreatedAt = p.CreatedAt })
        .ToListAsync();

    var hasMore = items.Count > limit;
    if (hasMore) items = items.Take(limit).ToList();

    // สร้าง cursors
    string? nextCursor = hasMore ? EncodeCursor(items.Last()) : null;
    string? previousCursor = cursor != null ? EncodeCursor(items.First()) : null;

    return Ok(new CursorPaginationResponse<ProductDto>
    {
        Items = items,
        NextCursor = nextCursor,
        PreviousCursor = previousCursor,
        HasMore = hasMore
    });
}

private static string EncodeCursor(ProductDto item)
{
    var cursor = $"{item.Id}:{item.CreatedAt:O}";
    return Convert.ToBase64String(Encoding.UTF8.GetBytes(cursor));
}

private static (int Id, DateTime CreatedAt) DecodeCursor(string cursor)
{
    var decoded = Encoding.UTF8.GetString(Convert.FromBase64String(cursor));
    var parts = decoded.Split(':', 2);
    return (int.Parse(parts[0]), DateTime.Parse(parts[1]));
}

// 3. Keyset Pagination - เร็วที่สุด เหมาะกับ large datasets
[HttpGet("keyset")]
public async Task<IActionResult> GetWithKeyset(
    [FromQuery] int? lastId = null,  // ID ของ item สุดท้ายในหน้าก่อน
    [FromQuery] int pageSize = 20,
    [FromQuery] string? sortBy = "id",
    [FromQuery] string? search = null)
{
    var query = _context.Products.AsQueryable();

    if (!string.IsNullOrEmpty(search))
        query = query.Where(p => p.Name.Contains(search));

    // Keyset: ใช้ WHERE แทน OFFSET
    // เร็วกว่า OFFSET มากเพราะใช้ Index ได้เต็มที่
    if (lastId.HasValue)
    {
        query = sortBy switch
        {
            "name" => query.Where(p => string.Compare(p.Name, GetLastName(lastId.Value)) > 0 ||
                                      (p.Name == GetLastName(lastId.Value) && p.Id > lastId.Value)),
            "price" => query.Where(p => p.Price > GetLastPrice(lastId.Value) ||
                                       (p.Price == GetLastPrice(lastId.Value) && p.Id > lastId.Value)),
            _ => query.Where(p => p.Id > lastId.Value)
        };
    }

    var items = await query
        .OrderBy(p => p.Id) // ต้องมี unique sort key
        .Take(pageSize + 1)
        .Select(p => new ProductDto { Id = p.Id, Name = p.Name, Price = p.Price })
        .ToListAsync();

    var hasMore = items.Count > pageSize;
    if (hasMore) items = items.Take(pageSize).ToList();

    return Ok(new KeysetPaginationResponse<ProductDto>
    {
        Items = items,
        LastId = items.LastOrDefault()?.Id,
        HasMore = hasMore,
        PageSize = pageSize
    });
}

// ขาดเมธอด helper สำหรับ keyset - ในการใช้จริงต้องดึงจาก DB
private string GetLastName(int lastId) => _context.Products.Find(lastId)?.Name ?? "";
private decimal GetLastPrice(int lastId) => _context.Products.Find(lastId)?.Price ?? 0;
```

```csharp
// Response Models สำหรับ Pagination
public class OffsetPaginationResponse<T>
{
    public IEnumerable<T> Items { get; set; } = Enumerable.Empty<T>();
    public int TotalCount { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalPages { get; set; }
    public bool HasPrevious { get; set; }
    public bool HasNext { get; set; }

    // Link headers (RFC 5988)
    public PaginationLinks Links { get; set; } = new();
}

public class CursorPaginationResponse<T>
{
    public IEnumerable<T> Items { get; set; } = Enumerable.Empty<T>();
    public string? NextCursor { get; set; }
    public string? PreviousCursor { get; set; }
    public bool HasMore { get; set; }
}

public class KeysetPaginationResponse<T>
{
    public IEnumerable<T> Items { get; set; } = Enumerable.Empty<T>();
    public int? LastId { get; set; }
    public bool HasMore { get; set; }
    public int PageSize { get; set; }
}

public class PaginationLinks
{
    public string? First { get; set; }
    public string? Previous { get; set; }
    public string? Next { get; set; }
    public string? Last { get; set; }
}
```

### เปรียบเทียบ Pagination Strategies

| รูปแบบ | ข้อดี | ข้อเสีย | เหมาะกับ |
|-------|-------|---------|---------|
| Offset | ง่าย, รองรับ jump to page | ช้าเมื่อข้อมูลมาก, data drift | Admin panels ขนาดเล็ก |
| Cursor | กันปัญหา data drift | ไม่สามารถ jump to page | Social media feeds |
| Keyset | เร็วที่สุด | ต้องมี unique sorted column | Large datasets, reports |

---

## ขั้นตอนที่ 960: API Rate Limiting และ Throttling Strategies

### การทำ Rate Limiting ใน ASP.NET Core

```bash
# ตั้งแต่ .NET 7 มี built-in rate limiting
# ไม่ต้องติดตั้ง package เพิ่ม สำหรับ basic rate limiting
# หรือใช้ AspNetCoreRateLimit สำหรับ advanced features
dotnet add package AspNetCoreRateLimit
```

```csharp
// Program.cs - Built-in Rate Limiting (.NET 7+)
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();

// ตั้งค่า Rate Limiting หลายรูปแบบ
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    // 1. Fixed Window - จำกัด requests ในช่วงเวลาที่กำหนด
    options.AddFixedWindowLimiter("fixed", config =>
    {
        config.PermitLimit = 100;             // 100 requests
        config.Window = TimeSpan.FromMinutes(1); // ต่อ 1 นาที
        config.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        config.QueueLimit = 10;              // รอได้สูงสุด 10 requests
    });

    // 2. Sliding Window - คล้าย Fixed Window แต่นุ่มนวลกว่า
    options.AddSlidingWindowLimiter("sliding", config =>
    {
        config.PermitLimit = 100;
        config.Window = TimeSpan.FromMinutes(1);
        config.SegmentsPerWindow = 4; // แบ่งเป็น 4 segments (15 วินาทีต่อ segment)
        config.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        config.QueueLimit = 5;
    });

    // 3. Token Bucket - เหมาะกับ API ที่ต้องการ burst ได้บ้าง
    options.AddTokenBucketLimiter("token-bucket", config =>
    {
        config.TokenLimit = 200;                        // ขนาด bucket
        config.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        config.QueueLimit = 10;
        config.ReplenishmentPeriod = TimeSpan.FromSeconds(10); // เติม token ทุก 10 วินาที
        config.TokensPerPeriod = 20;                   // เติมครั้งละ 20 tokens
        config.AutoReplenishment = true;
    });

    // 4. Concurrency Limiter - จำกัด concurrent requests
    options.AddConcurrencyLimiter("concurrent", config =>
    {
        config.PermitLimit = 50;    // concurrent requests สูงสุด
        config.QueueProcessingOrder = QueueProcessingOrder.NewestFirst;
        config.QueueLimit = 20;
    });

    // 5. Per-user Rate Limiting (IP-based)
    options.AddPolicy("per-user", context =>
    {
        // ใช้ User ID ถ้า authenticated, ไม่งั้นใช้ IP
        var userId = context.User.FindFirst("sub")?.Value
            ?? context.Connection.RemoteIpAddress?.ToString()
            ?? "anonymous";

        return RateLimitPartition.GetFixedWindowLimiter(userId, _ =>
            new FixedWindowRateLimiterOptions
            {
                PermitLimit = 60,
                Window = TimeSpan.FromMinutes(1),
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                QueueLimit = 0
            });
    });

    // Callback เมื่อถูก reject
    options.OnRejected = async (context, cancellationToken) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;

        // เพิ่ม Retry-After header
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            context.HttpContext.Response.Headers.RetryAfter =
                ((int)retryAfter.TotalSeconds).ToString();
        }

        context.HttpContext.Response.ContentType = "application/json";
        await context.HttpContext.Response.WriteAsync(
            "{\"error\": \"Too Many Requests\", \"message\": \"คุณส่ง request มากเกินไป กรุณารอสักครู่\"}",
            cancellationToken);
    };
});

var app = builder.Build();

// ใช้ Rate Limiting middleware
app.UseRateLimiter();

app.MapControllers();
app.Run();
```

```csharp
// Controllers - การใช้ Rate Limiting บน specific endpoints
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    // จำกัด login attempts: 5 ครั้งต่อนาที
    [HttpPost("login")]
    [EnableRateLimiting("sliding")]
    public async Task<IActionResult> Login([FromBody] LoginRequest request)
    {
        // Login logic
        return Ok(new { Token = "jwt-token-here" });
    }

    // จำกัดเข้มงวดกว่าสำหรับ password reset
    [HttpPost("forgot-password")]
    [EnableRateLimiting("token-bucket")]
    public async Task<IActionResult> ForgotPassword([FromBody] ForgotPasswordRequest request)
    {
        return Ok(new { Message = "Email sent if account exists" });
    }
}

[ApiController]
[Route("api/[controller]")]
[EnableRateLimiting("per-user")] // ใช้ทั้ง controller
public class ProductsController : ControllerBase
{
    // endpoints ใน controller นี้จะใช้ per-user rate limiting
    [HttpGet]
    public IActionResult GetAll() => Ok("Products");

    [HttpPost]
    [EnableRateLimiting("fixed")] // Override สำหรับ endpoint นี้
    public IActionResult Create([FromBody] object request) => Ok("Created");

    [HttpGet("export")]
    [DisableRateLimiting] // ปิด rate limiting สำหรับ endpoint นี้
    public IActionResult Export() => Ok("Exported");
}
```

```csharp
// Advanced Rate Limiting ด้วย AspNetCoreRateLimit
// dotnet add package AspNetCoreRateLimit

using AspNetCoreRateLimit;

var builder = WebApplication.CreateBuilder(args);

// เพิ่ม memory cache
builder.Services.AddMemoryCache();

// Load configuration
builder.Services.Configure<IpRateLimitOptions>(builder.Configuration.GetSection("IpRateLimiting"));
builder.Services.Configure<ClientRateLimitOptions>(builder.Configuration.GetSection("ClientRateLimiting"));

// Register stores
builder.Services.AddInMemoryRateLimiting();

// Register resolvers
builder.Services.AddSingleton<IRateLimitConfiguration, RateLimitConfiguration>();

var app = builder.Build();

// Seed rate limit stores
await app.Services.GetRequiredService<IIpPolicyStore>().SeedAsync();
await app.Services.GetRequiredService<IClientPolicyStore>().SeedAsync();

app.UseIpRateLimiting();
app.UseClientRateLimiting();

app.MapControllers();
app.Run();
```

```json
// appsettings.json - ตั้งค่า Rate Limiting แบบละเอียด
{
  "IpRateLimiting": {
    "EnableEndpointRateLimiting": true,
    "StackBlockedRequests": false,
    "RealIpHeader": "X-Real-IP",
    "ClientIdHeader": "X-ClientId",
    "HttpStatusCode": 429,
    "GeneralRules": [
      {
        "Endpoint": "*",
        "Period": "1s",
        "Limit": 10
      },
      {
        "Endpoint": "*",
        "Period": "15m",
        "Limit": 100
      },
      {
        "Endpoint": "*",
        "Period": "12h",
        "Limit": 1000
      },
      {
        "Endpoint": "*",
        "Period": "7d",
        "Limit": 10000
      }
    ],
    "EndpointWhitelist": ["get:/health", "get:/ping"],
    "IpWhitelist": ["127.0.0.1", "::1"]
  }
}
```

```csharp
// Throttling ด้วย Custom Middleware - เพื่อควบคุมแบบละเอียด
public class ThrottlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IThrottleStore _store;
    private readonly ThrottlingOptions _options;

    public ThrottlingMiddleware(
        RequestDelegate next,
        IThrottleStore store,
        ThrottlingOptions options)
    {
        _next = next;
        _store = store;
        _options = options;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var key = GetThrottleKey(context);
        var tier = GetUserTier(context);
        var limit = GetLimitForTier(tier);

        var (isAllowed, requestCount, resetTime) = await _store.CheckAndIncrementAsync(
            key, limit, _options.WindowDuration);

        // เพิ่ม headers บอก client เกี่ยวกับ rate limit
        context.Response.Headers["X-RateLimit-Limit"] = limit.ToString();
        context.Response.Headers["X-RateLimit-Remaining"] = Math.Max(0, limit - requestCount).ToString();
        context.Response.Headers["X-RateLimit-Reset"] = new DateTimeOffset(resetTime).ToUnixTimeSeconds().ToString();

        if (!isAllowed)
        {
            context.Response.StatusCode = StatusCodes.Status429TooManyRequests;
            context.Response.ContentType = "application/json";

            var retryAfter = (int)(resetTime - DateTime.UtcNow).TotalSeconds;
            context.Response.Headers["Retry-After"] = retryAfter.ToString();

            await context.Response.WriteAsync(System.Text.Json.JsonSerializer.Serialize(new
            {
                error = "rate_limit_exceeded",
                message = "คุณส่ง request มากเกินไป",
                retryAfter = retryAfter,
                limit = limit,
                tier = tier.ToString()
            }));
            return;
        }

        await _next(context);
    }

    private string GetThrottleKey(HttpContext context)
    {
        // ใช้ user ID ถ้า authenticated
        if (context.User.Identity?.IsAuthenticated == true)
        {
            var userId = context.User.FindFirst("sub")?.Value;
            if (userId != null)
                return $"user:{userId}:{context.Request.Path}";
        }

        // ใช้ API Key ถ้ามี
        if (context.Request.Headers.TryGetValue("X-Api-Key", out var apiKey))
            return $"api-key:{apiKey}:{context.Request.Path}";

        // ใช้ IP address
        var ip = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        return $"ip:{ip}:{context.Request.Path}";
    }

    private UserTier GetUserTier(HttpContext context)
    {
        if (!context.User.Identity?.IsAuthenticated == true)
            return UserTier.Anonymous;

        var tier = context.User.FindFirst("tier")?.Value;
        return tier switch
        {
            "premium" => UserTier.Premium,
            "enterprise" => UserTier.Enterprise,
            _ => UserTier.Free
        };
    }

    private int GetLimitForTier(UserTier tier) => tier switch
    {
        UserTier.Anonymous => 20,    // 20 req/min
        UserTier.Free => 60,         // 60 req/min
        UserTier.Premium => 300,     // 300 req/min
        UserTier.Enterprise => 1000, // 1000 req/min
        _ => 20
    };
}

public enum UserTier { Anonymous, Free, Premium, Enterprise }

public class ThrottlingOptions
{
    public TimeSpan WindowDuration { get; set; } = TimeSpan.FromMinutes(1);
}

public interface IThrottleStore
{
    Task<(bool IsAllowed, int RequestCount, DateTime ResetTime)> CheckAndIncrementAsync(
        string key, int limit, TimeSpan window);
}

// Redis-based Throttle Store
public class RedisThrottleStore : IThrottleStore
{
    private readonly IDatabase _redis;

    public RedisThrottleStore(IConnectionMultiplexer redis)
    {
        _redis = redis.GetDatabase();
    }

    public async Task<(bool IsAllowed, int RequestCount, DateTime ResetTime)> CheckAndIncrementAsync(
        string key, int limit, TimeSpan window)
    {
        var now = DateTime.UtcNow;
        var windowKey = $"throttle:{key}:{now:yyyyMMddHHmm}";
        var resetTime = new DateTime(now.Year, now.Month, now.Day, now.Hour, now.Minute, 0, DateTimeKind.Utc)
            .Add(window);

        var count = await _redis.StringIncrementAsync(windowKey);
        await _redis.KeyExpireAsync(windowKey, window);

        return ((int)count <= limit, (int)count, resetTime);
    }
}

// Extension method สำหรับ register middleware
public static class ThrottlingMiddlewareExtensions
{
    public static IApplicationBuilder UseThrottling(this IApplicationBuilder app)
        => app.UseMiddleware<ThrottlingMiddleware>();

    public static IServiceCollection AddThrottling(
        this IServiceCollection services,
        Action<ThrottlingOptions>? configure = null)
    {
        var options = new ThrottlingOptions();
        configure?.Invoke(options);

        services.AddSingleton(options);
        services.AddSingleton<IThrottleStore, RedisThrottleStore>();

        return services;
    }
}
```

### สรุป Rate Limiting Algorithms

| Algorithm | วิธีทำงาน | ข้อดี | ข้อเสีย |
|-----------|----------|-------|---------|
| Fixed Window | นับ requests ในช่วงเวลา fixed | ง่าย, ใช้ memory น้อย | Edge case ที่ boundary |
| Sliding Window | หน้าต่างเวลาที่เลื่อนไปตาม requests | แม่นยำกว่า | ใช้ memory มากกว่า |
| Token Bucket | เติม token สม่ำเสมอ, ใช้ token ต่อ request | รองรับ burst ได้ | ซับซ้อนกว่า |
| Leaky Bucket | ส่ง requests ในอัตราสม่ำเสมอ | output rate คงที่ | ไม่รองรับ burst |

---

## สรุปบทที่ 96

ในบทนี้เราได้เรียนรู้เกี่ยวกับ API Design และ OpenAPI อย่างครบถ้วน:

- **Richardson Maturity Model**: เข้าใจระดับความ RESTful ตั้งแต่ Level 0-3 และ HATEOAS
- **Swashbuckle/OpenAPI**: ตั้งค่า Swagger แบบสมบูรณ์ พร้อม XML docs, Security, และ Custom Filters
- **API Versioning**: ใช้ Asp.Versioning รองรับ URL, Header, และ Query String versioning
- **Minimal APIs**: เปรียบเทียบกับ Controller-based APIs และรู้ว่าเมื่อไหรควรใช้อะไร
- **API Gateway**: ออกแบบ BFF Pattern, Aggregation, และ Transformation ด้วย YARP
- **GraphQL/REST/gRPC**: รู้จักข้อดีข้อเสียและเลือกใช้ได้ถูกต้อง
- **Code Generation**: ใช้ NSwag และ Kiota สร้าง client code อัตโนมัติ
- **API Testing**: ทดสอบด้วย RestSharp และ Refit อย่างมีประสิทธิภาพ
- **Pagination**: เลือกใช้ Offset, Cursor-based, หรือ Keyset Pagination ได้เหมาะสม
- **Rate Limiting**: ปกป้อง API ด้วย Built-in Rate Limiter และ Custom Throttling

---

## แหล่งอ้างอิงเพิ่มเติม

- [Microsoft OpenAPI Documentation](https://learn.microsoft.com/en-us/aspnet/core/tutorials/web-api-help-pages-using-swagger)
- [Asp.Versioning Documentation](https://github.com/dotnet/aspnet-api-versioning)
- [YARP Documentation](https://microsoft.github.io/reverse-proxy/)
- [HotChocolate GraphQL](https://chillicream.com/docs/hotchocolate)
- [RestSharp Documentation](https://restsharp.dev)
- [Refit Documentation](https://github.com/reactiveui/refit)

---

**[← Part 95: Performance Optimization](part95-performance.md)** | **[Part 97: Security →](part97-security.md)**
