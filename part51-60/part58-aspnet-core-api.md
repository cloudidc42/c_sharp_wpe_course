# Part 58: ASP.NET Core REST API (Steps 571-580)

> ครอบคลุม Steps 571-580: การสร้าง REST API ด้วย ASP.NET Core อย่างครบถ้วน

---

## Step 571: ASP.NET Core Setup และการกำหนดค่าเริ่มต้น

### แนวคิดพื้นฐาน

ASP.NET Core เป็น Framework สำหรับสร้าง Web Application และ API ที่ทันสมัย รวดเร็ว และข้ามแพลตฟอร์ม ใน .NET 6+ มีการปรับปรุง Program.cs ให้กระชับขึ้นด้วย Minimal Hosting Model

### Minimal API vs Controller-based

ทั้งสองแนวทางมีจุดแข็งที่แตกต่างกัน:

| คุณสมบัติ | Minimal API | Controller-based |
|---|---|---|
| ความซับซ้อน | น้อย เหมาะ Microservices | มากกว่า เหมาะ Enterprise |
| Routing | ประกาศตรงใน Program.cs | ใช้ Attribute Routing |
| ความยืดหยุ่น | จำกัด | สูงมาก |
| Testing | ซับซ้อนกว่าเล็กน้อย | ทดสอบง่ายกว่า |
| Performance | เร็วกว่าเล็กน้อย | ใกล้เคียงกัน |

**Minimal API** เหมาะสำหรับ:
- Microservices ขนาดเล็ก
- Prototype หรือ POC
- API ที่มีไม่กี่ Endpoints

**Controller-based** เหมาะสำหรับ:
- Application ขนาดใหญ่
- ทีมใหญ่ที่ต้องการ Convention ชัดเจน
- เมื่อต้องการ Filter, Model Binding ขั้นสูง

### Program.cs Setup พร้อมอธิบาย

```csharp
// Program.cs - Controller-based API
var builder = WebApplication.CreateBuilder(args);

// ===== บริการ (Services) =====
// เพิ่ม Controllers
builder.Services.AddControllers(options =>
{
    // กำหนด Global Filters
    options.Filters.Add<ValidationFilter>();
    // กำหนด Return Format เริ่มต้น
    options.ReturnHttpNotAcceptable = true;
})
.AddJsonOptions(options =>
{
    // ใช้ camelCase สำหรับ JSON
    options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    // ไม่แสดง null ใน Response
    options.JsonSerializerOptions.DefaultIgnoreCondition = 
        JsonIgnoreCondition.WhenWritingNull;
    // แปลง Enum เป็น string
    options.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
});

// เพิ่ม API Explorer สำหรับ Swagger
builder.Services.AddEndpointsApiExplorer();

// เพิ่ม Swagger/OpenAPI
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "API สำหรับจัดการสินค้า",
        Contact = new OpenApiContact
        {
            Name = "Dev Team",
            Email = "dev@example.com"
        }
    });
    
    // เพิ่ม JWT Authentication ใน Swagger
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.Http,
        BearerFormat = "JWT",
        Scheme = "Bearer",
        Description = "ใส่ JWT token โดยไม่ต้องมี prefix 'Bearer'"
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
    
    // รวม XML Comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);
});

// เพิ่ม Database Context
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// เพิ่ม AutoMapper
builder.Services.AddAutoMapper(typeof(Program));

// เพิ่ม Repositories และ Services
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IProductService, ProductService>();

// ===== CORS Configuration =====
builder.Services.AddCors(options =>
{
    // Policy สำหรับ Development
    options.AddPolicy("Development", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
    
    // Policy สำหรับ Production
    options.AddPolicy("Production", policy =>
    {
        policy.WithOrigins("https://myapp.com", "https://www.myapp.com")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Content-Type", "Authorization")
              .AllowCredentials();
    });
});

// ===== Health Checks =====
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>("database")
    .AddCheck<CustomHealthCheck>("custom-check")
    .AddUrlGroup(new Uri("https://api.external.com/health"), "external-api");

// ===== Build Application =====
var app = builder.Build();

// ===== Middleware Pipeline (ลำดับสำคัญมาก!) =====

// Development Tools
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1");
        options.RoutePrefix = string.Empty; // Swagger ที่ root
    });
    app.UseDeveloperExceptionPage();
}
else
{
    // Production Error Handling
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

app.UseHttpsRedirection();

// CORS ต้องอยู่ก่อน Authentication
app.UseCors(app.Environment.IsDevelopment() ? "Development" : "Production");

// Logging Middleware
app.UseMiddleware<RequestLoggingMiddleware>();

// Authentication & Authorization
app.UseAuthentication();
app.UseAuthorization();

// Health Check Endpoint
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

// Map Controllers
app.MapControllers();

app.Run();
```

### Health Checks แบบ Custom

```csharp
// Custom Health Check ตรวจสอบสถานะ Service
public class CustomHealthCheck : IHealthCheck
{
    private readonly IProductService _productService;
    
    public CustomHealthCheck(IProductService productService)
    {
        _productService = productService;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // ทดสอบว่า Service ทำงานได้
            var count = await _productService.GetCountAsync();
            
            return HealthCheckResult.Healthy(
                $"Service ทำงานปกติ มีสินค้า {count} รายการ");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Service มีปัญหา",
                exception: ex,
                data: new Dictionary<string, object>
                {
                    { "error", ex.Message }
                });
        }
    }
}
```

### Minimal API ตัวอย่าง

```csharp
// Program.cs - Minimal API
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// ประกาศ Endpoints โดยตรง
app.MapGet("/products", async (IProductService service) =>
{
    var products = await service.GetAllAsync();
    return Results.Ok(products);
})
.WithName("GetProducts")
.WithOpenApi();

app.MapGet("/products/{id:int}", async (int id, IProductService service) =>
{
    var product = await service.GetByIdAsync(id);
    return product is null ? Results.NotFound() : Results.Ok(product);
})
.WithName("GetProduct");

app.MapPost("/products", async (CreateProductDto dto, IProductService service) =>
{
    var product = await service.CreateAsync(dto);
    return Results.CreatedAtRoute("GetProduct", new { id = product.Id }, product);
})
.WithName("CreateProduct")
.RequireAuthorization();

app.Run();
```

---

## Step 572: Controllers และ Routes

### ApiController Attribute

`[ApiController]` เพิ่มพฤติกรรมอัตโนมัติหลายอย่าง:
- **Automatic Model Validation**: คืน 400 Bad Request อัตโนมัติเมื่อ Model ไม่ผ่าน Validation
- **Binding Source Inference**: อนุมาน `[FromBody]`, `[FromRoute]`, `[FromQuery]` อัตโนมัติ
- **ProblemDetails**: ใช้ RFC 7807 สำหรับ Error Response

```csharp
[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    private readonly ILogger<ProductsController> _logger;
    
    public ProductsController(
        IProductService productService,
        ILogger<ProductsController> logger)
    {
        _productService = productService;
        _logger = logger;
    }
}
```

### HTTP Verbs และ Route Parameters

```csharp
// GET /api/products - ดึงสินค้าทั้งหมด
[HttpGet]
public async Task<ActionResult<IEnumerable<ProductResponseDto>>> GetAll(
    [FromQuery] ProductFilterDto filter)
{
    var products = await _productService.GetAllAsync(filter);
    return Ok(products);
}

// GET /api/products/5 - ดึงสินค้าตาม ID
[HttpGet("{id:int}")]
public async Task<ActionResult<ProductResponseDto>> GetById(int id)
{
    var product = await _productService.GetByIdAsync(id);
    
    if (product is null)
        return NotFound(new { message = $"ไม่พบสินค้า ID: {id}" });
    
    return Ok(product);
}

// GET /api/products/sku/ABC-123 - ดึงสินค้าตาม SKU
[HttpGet("sku/{sku}")]
public async Task<ActionResult<ProductResponseDto>> GetBySku(string sku)
{
    var product = await _productService.GetBySkuAsync(sku);
    
    if (product is null)
        return NotFound();
    
    return Ok(product);
}

// POST /api/products - สร้างสินค้าใหม่
[HttpPost]
[ProducesResponseType(typeof(ProductResponseDto), StatusCodes.Status201Created)]
[ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
public async Task<ActionResult<ProductResponseDto>> Create(
    [FromBody] CreateProductDto dto)
{
    var product = await _productService.CreateAsync(dto);
    
    return CreatedAtAction(
        nameof(GetById),
        new { id = product.Id },
        product);
}

// PUT /api/products/5 - อัปเดตสินค้าทั้งหมด
[HttpPut("{id:int}")]
public async Task<IActionResult> Update(int id, [FromBody] UpdateProductDto dto)
{
    if (id != dto.Id)
        return BadRequest(new { message = "ID ไม่ตรงกัน" });
    
    var updated = await _productService.UpdateAsync(id, dto);
    
    if (!updated)
        return NotFound();
    
    return NoContent();
}

// PATCH /api/products/5 - อัปเดตบางฟิลด์
[HttpPatch("{id:int}")]
public async Task<IActionResult> PartialUpdate(
    int id,
    [FromBody] JsonPatchDocument<UpdateProductDto> patchDoc)
{
    var product = await _productService.GetByIdAsync(id);
    
    if (product is null)
        return NotFound();
    
    var dto = new UpdateProductDto
    {
        Name = product.Name,
        Price = product.Price,
        Stock = product.Stock
    };
    
    // ใช้ ModelState ใน ApplyTo เพื่อ Validate
    patchDoc.ApplyTo(dto, ModelState);
    
    if (!ModelState.IsValid)
        return BadRequest(ModelState);
    
    await _productService.UpdateAsync(id, dto);
    
    return NoContent();
}

// DELETE /api/products/5 - ลบสินค้า
[HttpDelete("{id:int}")]
[ProducesResponseType(StatusCodes.Status204NoContent)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<IActionResult> Delete(int id)
{
    var deleted = await _productService.DeleteAsync(id);
    
    if (!deleted)
        return NotFound();
    
    return NoContent();
}
```

### Query Strings และ Complex Filtering

```csharp
// DTO สำหรับรับ Query Parameters
public class ProductFilterDto
{
    // Filtering
    public string? Name { get; set; }
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
    public int? CategoryId { get; set; }
    public bool? IsActive { get; set; }
    
    // Sorting
    public string SortBy { get; set; } = "Name";
    public string SortOrder { get; set; } = "asc"; // asc, desc
    
    // Pagination
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 10;
    
    // ตรวจสอบ PageSize ไม่เกิน 100
    public int ValidatedPageSize => Math.Min(PageSize, 100);
}

// GET /api/products?name=apple&minPrice=10&maxPrice=100&page=1&pageSize=20&sortBy=Price&sortOrder=desc
[HttpGet]
public async Task<ActionResult<PagedResult<ProductResponseDto>>> GetAll(
    [FromQuery] ProductFilterDto filter)
{
    _logger.LogInformation(
        "ดึงสินค้า: Name={Name}, Page={Page}, PageSize={PageSize}",
        filter.Name, filter.Page, filter.ValidatedPageSize);
    
    var result = await _productService.GetPagedAsync(filter);
    
    // เพิ่ม Pagination Headers
    Response.Headers.Append("X-Total-Count", result.TotalCount.ToString());
    Response.Headers.Append("X-Total-Pages", result.TotalPages.ToString());
    Response.Headers.Append("X-Current-Page", result.CurrentPage.ToString());
    
    return Ok(result);
}
```

### ActionResult<T> vs IActionResult

```csharp
// ActionResult<T> - แนะนำใช้มากกว่าเพราะ Swagger รู้ Type
[HttpGet("{id}")]
public async Task<ActionResult<ProductResponseDto>> GetById(int id)
{
    // ส่งคืน ProductResponseDto โดยตรง - แปลงเป็น 200 OK อัตโนมัติ
    var product = await _productService.GetByIdAsync(id);
    
    if (product is null)
        return NotFound(); // IActionResult
    
    return product; // implicit conversion เป็น Ok(product)
    // หรือ return Ok(product);
}

// IActionResult - ใช้เมื่อมีหลาย Return Types ที่ไม่ตรงกัน
[HttpPost]
public async Task<IActionResult> Create([FromBody] CreateProductDto dto)
{
    var product = await _productService.CreateAsync(dto);
    return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
}
```

### Route Constraints

```csharp
// Constraint ประเภทต่างๆ
[HttpGet("{id:int}")]           // ต้องเป็น integer
[HttpGet("{id:guid}")]          // ต้องเป็น GUID
[HttpGet("{id:long}")]          // ต้องเป็น long
[HttpGet("{name:alpha}")]       // ต้องเป็น letters เท่านั้น
[HttpGet("{slug:regex(^[a-z0-9-]+$)}")] // ต้องตาม Regex
[HttpGet("{id:int:min(1)}")]    // ต้องเป็น integer >= 1
[HttpGet("{id:int:range(1,100)}")] // ต้องอยู่ในช่วง 1-100
```

---

## Step 573: DTOs และ AutoMapper

### Request DTOs

```csharp
// DTO สำหรับสร้างสินค้า
public class CreateProductDto
{
    [Required(ErrorMessage = "กรุณาใส่ชื่อสินค้า")]
    [StringLength(200, MinimumLength = 2, ErrorMessage = "ชื่อต้องมี 2-200 ตัวอักษร")]
    public string Name { get; set; } = string.Empty;
    
    [StringLength(1000, ErrorMessage = "คำอธิบายไม่เกิน 1000 ตัวอักษร")]
    public string? Description { get; set; }
    
    [Required(ErrorMessage = "กรุณาใส่ราคา")]
    [Range(0.01, 999999.99, ErrorMessage = "ราคาต้องอยู่ระหว่าง 0.01 - 999,999.99")]
    public decimal Price { get; set; }
    
    [Range(0, int.MaxValue, ErrorMessage = "จำนวนสต็อกต้องไม่น้อยกว่า 0")]
    public int Stock { get; set; }
    
    [Required(ErrorMessage = "กรุณาเลือกหมวดหมู่")]
    public int CategoryId { get; set; }
    
    [Required(ErrorMessage = "กรุณาใส่ SKU")]
    [RegularExpression(@"^[A-Z]{2,5}-\d{4,6}$", 
        ErrorMessage = "รูปแบบ SKU ไม่ถูกต้อง (เช่น ABC-12345)")]
    public string Sku { get; set; } = string.Empty;
}

// DTO สำหรับอัปเดตสินค้า
public class UpdateProductDto
{
    public int Id { get; set; }
    
    [Required(ErrorMessage = "กรุณาใส่ชื่อสินค้า")]
    [StringLength(200, MinimumLength = 2)]
    public string Name { get; set; } = string.Empty;
    
    [StringLength(1000)]
    public string? Description { get; set; }
    
    [Required]
    [Range(0.01, 999999.99)]
    public decimal Price { get; set; }
    
    [Range(0, int.MaxValue)]
    public int Stock { get; set; }
    
    public bool IsActive { get; set; }
}
```

### Response DTOs

```csharp
// DTO สำหรับส่งกลับข้อมูลสินค้า
public class ProductResponseDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string? Description { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public string Sku { get; set; } = string.Empty;
    public bool IsActive { get; set; }
    public string CategoryName { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
}

// DTO สำหรับ Pagination
public class PagedResult<T>
{
    public IEnumerable<T> Items { get; set; } = Enumerable.Empty<T>();
    public int TotalCount { get; set; }
    public int CurrentPage { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPreviousPage => CurrentPage > 1;
    public bool HasNextPage => CurrentPage < TotalPages;
}
```

### AutoMapper Profile

```csharp
// AutoMapper Profile กำหนดการแปลง Object
public class ProductMappingProfile : Profile
{
    public ProductMappingProfile()
    {
        // Product -> ProductResponseDto
        CreateMap<Product, ProductResponseDto>()
            .ForMember(dest => dest.CategoryName, 
                opt => opt.MapFrom(src => src.Category != null ? src.Category.Name : string.Empty));
        
        // CreateProductDto -> Product
        CreateMap<CreateProductDto, Product>()
            .ForMember(dest => dest.Id, opt => opt.Ignore())
            .ForMember(dest => dest.CreatedAt, opt => opt.MapFrom(_ => DateTime.UtcNow))
            .ForMember(dest => dest.UpdatedAt, opt => opt.Ignore())
            .ForMember(dest => dest.IsActive, opt => opt.MapFrom(_ => true));
        
        // UpdateProductDto -> Product (อัปเดต Existing Object)
        CreateMap<UpdateProductDto, Product>()
            .ForMember(dest => dest.CreatedAt, opt => opt.Ignore())
            .ForMember(dest => dest.UpdatedAt, opt => opt.MapFrom(_ => DateTime.UtcNow))
            .ForMember(dest => dest.Category, opt => opt.Ignore());
        
        // Map สำหรับ Category
        CreateMap<Category, CategoryResponseDto>();
        CreateMap<CreateCategoryDto, Category>();
    }
}

// การใช้งาน AutoMapper ใน Service
public class ProductService : IProductService
{
    private readonly IProductRepository _repository;
    private readonly IMapper _mapper;
    
    public ProductService(IProductRepository repository, IMapper mapper)
    {
        _repository = repository;
        _mapper = mapper;
    }
    
    public async Task<ProductResponseDto?> GetByIdAsync(int id)
    {
        var product = await _repository.GetByIdWithCategoryAsync(id);
        
        // Map Entity -> DTO
        return product is null ? null : _mapper.Map<ProductResponseDto>(product);
    }
    
    public async Task<ProductResponseDto> CreateAsync(CreateProductDto dto)
    {
        // Map DTO -> Entity
        var product = _mapper.Map<Product>(dto);
        
        await _repository.AddAsync(product);
        await _repository.SaveChangesAsync();
        
        // ดึงข้อมูลพร้อม Relations
        var created = await _repository.GetByIdWithCategoryAsync(product.Id);
        
        return _mapper.Map<ProductResponseDto>(created!);
    }
}
```

### Manual Mapping (ไม่ใช้ AutoMapper)

```csharp
// Extension Methods สำหรับ Manual Mapping
public static class ProductMappingExtensions
{
    public static ProductResponseDto ToDto(this Product product)
    {
        return new ProductResponseDto
        {
            Id = product.Id,
            Name = product.Name,
            Description = product.Description,
            Price = product.Price,
            Stock = product.Stock,
            Sku = product.Sku,
            IsActive = product.IsActive,
            CategoryName = product.Category?.Name ?? string.Empty,
            CreatedAt = product.CreatedAt,
            UpdatedAt = product.UpdatedAt
        };
    }
    
    public static Product ToEntity(this CreateProductDto dto)
    {
        return new Product
        {
            Name = dto.Name,
            Description = dto.Description,
            Price = dto.Price,
            Stock = dto.Stock,
            Sku = dto.Sku,
            CategoryId = dto.CategoryId,
            IsActive = true,
            CreatedAt = DateTime.UtcNow
        };
    }
    
    public static void UpdateEntity(this UpdateProductDto dto, Product product)
    {
        product.Name = dto.Name;
        product.Description = dto.Description;
        product.Price = dto.Price;
        product.Stock = dto.Stock;
        product.IsActive = dto.IsActive;
        product.UpdatedAt = DateTime.UtcNow;
    }
}
```

### ProblemDetails สำหรับ Error Responses

```csharp
// การตั้งค่า ProblemDetails ใน Program.cs
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Instance = 
            $"{context.HttpContext.Request.Method} {context.HttpContext.Request.Path}";
        
        context.ProblemDetails.Extensions.TryAdd(
            "requestId", 
            context.HttpContext.TraceIdentifier);
        
        context.ProblemDetails.Extensions.TryAdd(
            "timestamp",
            DateTime.UtcNow);
    };
});

// Custom Exception Middleware ที่ใช้ ProblemDetails
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;
    
    public GlobalExceptionMiddleware(
        RequestDelegate next,
        ILogger<GlobalExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (ValidationException ex)
        {
            _logger.LogWarning(ex, "Validation error occurred");
            
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            context.Response.ContentType = "application/problem+json";
            
            var problem = new ValidationProblemDetails(ex.Errors)
            {
                Title = "ข้อมูลไม่ถูกต้อง",
                Detail = ex.Message,
                Instance = context.Request.Path
            };
            
            await context.Response.WriteAsJsonAsync(problem);
        }
        catch (NotFoundException ex)
        {
            _logger.LogWarning(ex, "Resource not found");
            
            context.Response.StatusCode = StatusCodes.Status404NotFound;
            context.Response.ContentType = "application/problem+json";
            
            var problem = new ProblemDetails
            {
                Status = StatusCodes.Status404NotFound,
                Title = "ไม่พบข้อมูล",
                Detail = ex.Message,
                Instance = context.Request.Path
            };
            
            await context.Response.WriteAsJsonAsync(problem);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error occurred");
            
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            context.Response.ContentType = "application/problem+json";
            
            var problem = new ProblemDetails
            {
                Status = StatusCodes.Status500InternalServerError,
                Title = "เกิดข้อผิดพลาดภายในเซิร์ฟเวอร์",
                Detail = context.Request.HttpContext.Request.Host.Host.Contains("localhost")
                    ? ex.Message
                    : "เกิดข้อผิดพลาด กรุณาติดต่อผู้ดูแลระบบ",
                Instance = context.Request.Path
            };
            
            await context.Response.WriteAsJsonAsync(problem);
        }
    }
}
```

---

## Step 574: Middleware

### แนวคิด Middleware Pipeline

Middleware ใน ASP.NET Core ทำงานเป็น Pipeline แต่ละ Middleware สามารถ:
1. ทำงานก่อน Request ไปต่อ
2. เรียก `next(context)` เพื่อให้ Request ไปยัง Middleware ถัดไป
3. ทำงานหลัง Response กลับมา

```
Request --> [Middleware 1] --> [Middleware 2] --> [Middleware N] --> Endpoint
Response <-- [Middleware 1] <-- [Middleware 2] <-- [Middleware N] <--
```

### Custom Middleware พื้นฐาน

```csharp
// Middleware Class แบบ Convention-based
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;
    
    public RequestLoggingMiddleware(
        RequestDelegate next,
        ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        // === ทำงานก่อน Request ===
        var startTime = DateTime.UtcNow;
        var requestId = context.TraceIdentifier;
        
        _logger.LogInformation(
            "เริ่มต้น Request [{RequestId}] {Method} {Path}",
            requestId,
            context.Request.Method,
            context.Request.Path);
        
        // เรียก Middleware ถัดไป
        await _next(context);
        
        // === ทำงานหลัง Response ===
        var duration = DateTime.UtcNow - startTime;
        
        _logger.LogInformation(
            "สิ้นสุด Request [{RequestId}] {StatusCode} ใช้เวลา {DurationMs}ms",
            requestId,
            context.Response.StatusCode,
            duration.TotalMilliseconds);
    }
}

// ลงทะเบียน Middleware
// app.UseMiddleware<RequestLoggingMiddleware>();
```

### Exception Handling Middleware

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;
    private readonly IWebHostEnvironment _environment;
    
    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger,
        IWebHostEnvironment environment)
    {
        _next = next;
        _logger = logger;
        _environment = environment;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }
    
    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        var (statusCode, title) = exception switch
        {
            NotFoundException => (StatusCodes.Status404NotFound, "ไม่พบข้อมูล"),
            ValidationException => (StatusCodes.Status400BadRequest, "ข้อมูลไม่ถูกต้อง"),
            UnauthorizedException => (StatusCodes.Status401Unauthorized, "ไม่มีสิทธิ์เข้าถึง"),
            ForbiddenException => (StatusCodes.Status403Forbidden, "ถูกปฏิเสธการเข้าถึง"),
            ConflictException => (StatusCodes.Status409Conflict, "ข้อมูลซ้ำกัน"),
            _ => (StatusCodes.Status500InternalServerError, "เกิดข้อผิดพลาด")
        };
        
        _logger.LogError(exception, 
            "Exception ที่ {Path}: {Message}", 
            context.Request.Path,
            exception.Message);
        
        var problemDetails = new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Detail = _environment.IsDevelopment() 
                ? exception.Message 
                : "เกิดข้อผิดพลาด กรุณาติดต่อผู้ดูแลระบบ",
            Instance = context.Request.Path
        };
        
        if (_environment.IsDevelopment())
        {
            problemDetails.Extensions["stackTrace"] = exception.StackTrace;
            problemDetails.Extensions["exceptionType"] = exception.GetType().Name;
        }
        
        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/problem+json";
        
        await context.Response.WriteAsJsonAsync(problemDetails);
    }
}
```

### Correlation ID Middleware

```csharp
// Correlation ID ช่วยติดตาม Request ข้ามระบบ
public class CorrelationIdMiddleware
{
    private const string CorrelationIdHeader = "X-Correlation-ID";
    private readonly RequestDelegate _next;
    
    public CorrelationIdMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        // ตรวจสอบว่ามี Correlation ID ใน Header หรือไม่
        var correlationId = context.Request.Headers[CorrelationIdHeader].ToString();
        
        if (string.IsNullOrEmpty(correlationId))
        {
            correlationId = Guid.NewGuid().ToString();
        }
        
        // เพิ่ม Correlation ID ลงใน Response Header
        context.Response.Headers[CorrelationIdHeader] = correlationId;
        
        // เพิ่มลงใน Items เพื่อให้ Middleware อื่นใช้ได้
        context.Items["CorrelationId"] = correlationId;
        
        // เพิ่มลงใน Log Scope
        using var scope = context.RequestServices
            .GetRequiredService<ILogger<CorrelationIdMiddleware>>()
            .BeginScope(new Dictionary<string, object>
            {
                ["CorrelationId"] = correlationId
            });
        
        await _next(context);
    }
}
```

### Rate Limiting Middleware (.NET 8)

```csharp
// Program.cs - ตั้งค่า Rate Limiting
builder.Services.AddRateLimiter(options =>
{
    // Global Rate Limit
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
    {
        return RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.User.Identity?.Name 
                ?? context.Connection.RemoteIpAddress?.ToString() 
                ?? "anonymous",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            });
    });
    
    // Named Policies สำหรับ Endpoint เฉพาะ
    options.AddFixedWindowLimiter("api", opt =>
    {
        opt.PermitLimit = 10;
        opt.Window = TimeSpan.FromSeconds(10);
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit = 2;
    });
    
    // Sliding Window สำหรับ Upload
    options.AddSlidingWindowLimiter("upload", opt =>
    {
        opt.PermitLimit = 5;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.SegmentsPerWindow = 6;
    });
    
    // Token Bucket สำหรับ Premium Users
    options.AddTokenBucketLimiter("premium", opt =>
    {
        opt.TokenLimit = 100;
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit = 10;
        opt.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        opt.TokensPerPeriod = 20;
        opt.AutoReplenishment = true;
    });
    
    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            context.HttpContext.Response.Headers.RetryAfter = 
                ((int)retryAfter.TotalSeconds).ToString();
        }
        
        await context.HttpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = StatusCodes.Status429TooManyRequests,
            Title = "ส่ง Request มากเกินไป",
            Detail = "กรุณารอสักครู่แล้วลองใหม่"
        }, token);
    };
});

// ใช้งาน Rate Limiting Middleware
app.UseRateLimiter();

// ใช้ Named Policy กับ Endpoint เฉพาะ
[HttpPost("upload")]
[EnableRateLimiting("upload")]
public async Task<IActionResult> Upload([FromForm] IFormFile file)
{
    // ...
}

// ปิด Rate Limiting สำหรับ Endpoint เฉพาะ
[HttpGet("health")]
[DisableRateLimiting]
public IActionResult Health() => Ok("healthy");
```

---

## Step 575: Authentication & Authorization

### JWT Bearer Authentication Setup

```csharp
// Program.cs - ตั้งค่า JWT Authentication
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        // Validate ลายเซ็น Token
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!)),
        
        // Validate Issuer (ผู้ออก Token)
        ValidateIssuer = true,
        ValidIssuer = builder.Configuration["Jwt:Issuer"],
        
        // Validate Audience (ผู้รับ Token)
        ValidateAudience = true,
        ValidAudience = builder.Configuration["Jwt:Audience"],
        
        // Validate Expiry
        ValidateLifetime = true,
        ClockSkew = TimeSpan.Zero // ไม่มี Tolerance
    };
    
    // รับ Token จาก Cookie ด้วย (Optional)
    options.Events = new JwtBearerEvents
    {
        OnMessageReceived = context =>
        {
            // รับ Token จาก Cookie แทน Header
            context.Token = context.Request.Cookies["access_token"];
            return Task.CompletedTask;
        },
        
        OnAuthenticationFailed = context =>
        {
            if (context.Exception is SecurityTokenExpiredException)
            {
                context.Response.Headers.Append("Token-Expired", "true");
            }
            return Task.CompletedTask;
        },
        
        OnChallenge = context =>
        {
            // Override Default Challenge Response
            context.HandleResponse();
            context.Response.StatusCode = StatusCodes.Status401Unauthorized;
            context.Response.ContentType = "application/problem+json";
            
            var problem = new ProblemDetails
            {
                Status = StatusCodes.Status401Unauthorized,
                Title = "ไม่มีสิทธิ์เข้าถึง",
                Detail = "กรุณาเข้าสู่ระบบก่อน"
            };
            
            return context.Response.WriteAsJsonAsync(problem);
        }
    };
});
```

### JWT Token Service

```csharp
public class JwtTokenService
{
    private readonly IConfiguration _config;
    
    public JwtTokenService(IConfiguration config)
    {
        _config = config;
    }
    
    public string GenerateAccessToken(User user, IList<string> roles)
    {
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new(ClaimTypes.Email, user.Email),
            new(ClaimTypes.Name, user.FullName),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat, 
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64)
        };
        
        // เพิ่ม Roles เป็น Claims
        claims.AddRange(roles.Select(role => new Claim(ClaimTypes.Role, role)));
        
        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_config["Jwt:Key"]!));
        
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var token = new JwtSecurityToken(
            issuer: _config["Jwt:Issuer"],
            audience: _config["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(
                int.Parse(_config["Jwt:AccessTokenExpireMinutes"] ?? "15")),
            signingCredentials: credentials
        );
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
    
    public string GenerateRefreshToken()
    {
        var randomBytes = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes);
    }
    
    public ClaimsPrincipal? GetPrincipalFromExpiredToken(string token)
    {
        var tokenValidationParameters = new TokenValidationParameters
        {
            ValidateAudience = false,
            ValidateIssuer = false,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(_config["Jwt:Key"]!)),
            ValidateLifetime = false // อนุญาต Token หมดอายุ
        };
        
        var tokenHandler = new JwtSecurityTokenHandler();
        var principal = tokenHandler.ValidateToken(
            token, tokenValidationParameters, out SecurityToken securityToken);
        
        if (securityToken is not JwtSecurityToken jwtSecurityToken ||
            !jwtSecurityToken.Header.Alg.Equals(
                SecurityAlgorithms.HmacSha256,
                StringComparison.InvariantCultureIgnoreCase))
        {
            throw new SecurityTokenException("Token ไม่ถูกต้อง");
        }
        
        return principal;
    }
}
```

### Authorize Attributes

```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize] // ต้อง Login ก่อน
public class ProductsController : ControllerBase
{
    // GET /api/products - ทุกคนที่ Login แล้ว
    [HttpGet]
    [AllowAnonymous] // ยกเว้นไม่ต้อง Login สำหรับ Endpoint นี้
    public async Task<ActionResult<IEnumerable<ProductResponseDto>>> GetAll()
    {
        // ...
    }
    
    // GET /api/products/5 - ต้อง Login เท่านั้น
    [HttpGet("{id}")]
    public async Task<ActionResult<ProductResponseDto>> GetById(int id)
    {
        // ดึง User ปัจจุบันจาก Claims
        var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
        // ...
    }
    
    // POST /api/products - ต้องเป็น Admin
    [HttpPost]
    [Authorize(Roles = "Admin")]
    public async Task<ActionResult<ProductResponseDto>> Create(CreateProductDto dto)
    {
        // ...
    }
    
    // DELETE /api/products/5 - ต้องเป็น Admin หรือ SuperAdmin
    [HttpDelete("{id}")]
    [Authorize(Roles = "Admin,SuperAdmin")]
    public async Task<IActionResult> Delete(int id)
    {
        // ...
    }
    
    // PUT /api/products/5 - ใช้ Policy
    [HttpPut("{id}")]
    [Authorize(Policy = "CanEditProducts")]
    public async Task<IActionResult> Update(int id, UpdateProductDto dto)
    {
        // ...
    }
}
```

### Policy-based Authorization

```csharp
// กำหนด Policies ใน Program.cs
builder.Services.AddAuthorization(options =>
{
    // Policy แบบ Role
    options.AddPolicy("CanEditProducts", policy =>
        policy.RequireRole("Admin", "Manager"));
    
    // Policy แบบ Claim
    options.AddPolicy("HasApiAccess", policy =>
        policy.RequireClaim("api_access", "true"));
    
    // Policy แบบ Custom Requirement
    options.AddPolicy("CanDeleteProducts", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(21)));
    
    // Policy แบบ Assertion (ง่ายที่สุด)
    options.AddPolicy("IsProductOwner", policy =>
        policy.RequireAssertion(context =>
        {
            var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
            return userId != null;
        }));
    
    // Fallback Policy - ถ้าไม่มี [Authorize] ก็ต้อง Authenticated
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});

// Custom Requirement
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }
    
    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}

// Custom Requirement Handler
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var birthDateClaim = context.User.FindFirst("birthdate");
        
        if (birthDateClaim is null)
        {
            return Task.CompletedTask;
        }
        
        if (DateTime.TryParse(birthDateClaim.Value, out var birthDate))
        {
            var age = DateTime.Today.Year - birthDate.Year;
            
            if (age >= requirement.MinimumAge)
            {
                context.Succeed(requirement);
            }
        }
        
        return Task.CompletedTask;
    }
}

// ลงทะเบียน Handler
builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
```

---

## Step 576-580: Complete API สำหรับ Products

### Step 576: ProductsController พร้อม Full CRUD

```csharp
// Controllers/ProductsController.cs
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
[Authorize]
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;
    private readonly ILogger<ProductsController> _logger;
    
    public ProductsController(
        IProductService service,
        ILogger<ProductsController> logger)
    {
        _service = service;
        _logger = logger;
    }
    
    /// <summary>
    /// ดึงรายการสินค้าทั้งหมด (รองรับการกรอง เรียงลำดับ และแบ่งหน้า)
    /// </summary>
    /// <param name="filter">เงื่อนไขการกรองและแบ่งหน้า</param>
    /// <returns>รายการสินค้าพร้อม Pagination</returns>
    [HttpGet]
    [AllowAnonymous]
    [ProducesResponseType(typeof(PagedResult<ProductResponseDto>), StatusCodes.Status200OK)]
    public async Task<ActionResult<PagedResult<ProductResponseDto>>> GetAll(
        [FromQuery] ProductFilterDto filter)
    {
        var result = await _service.GetPagedAsync(filter);
        
        Response.Headers.Append("X-Total-Count", result.TotalCount.ToString());
        Response.Headers.Append("X-Total-Pages", result.TotalPages.ToString());
        
        return Ok(result);
    }
    
    /// <summary>
    /// ดึงสินค้าตาม ID
    /// </summary>
    [HttpGet("{id:int}", Name = "GetProductById")]
    [AllowAnonymous]
    [ProducesResponseType(typeof(ProductResponseDto), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductResponseDto>> GetById(int id)
    {
        var product = await _service.GetByIdAsync(id);
        
        if (product is null)
            return Problem(
                statusCode: StatusCodes.Status404NotFound,
                title: "ไม่พบสินค้า",
                detail: $"ไม่พบสินค้า ID: {id}");
        
        return Ok(product);
    }
    
    /// <summary>
    /// สร้างสินค้าใหม่
    /// </summary>
    [HttpPost]
    [Authorize(Roles = "Admin,Manager")]
    [ProducesResponseType(typeof(ProductResponseDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status409Conflict)]
    public async Task<ActionResult<ProductResponseDto>> Create(
        [FromBody] CreateProductDto dto)
    {
        // ตรวจสอบ SKU ซ้ำ
        if (await _service.SkuExistsAsync(dto.Sku))
        {
            return Problem(
                statusCode: StatusCodes.Status409Conflict,
                title: "SKU ซ้ำกัน",
                detail: $"SKU '{dto.Sku}' มีอยู่แล้วในระบบ");
        }
        
        var product = await _service.CreateAsync(dto);
        
        _logger.LogInformation(
            "สร้างสินค้าใหม่: ID={Id}, Name={Name}",
            product.Id, product.Name);
        
        return CreatedAtRoute(
            "GetProductById",
            new { id = product.Id },
            product);
    }
    
    /// <summary>
    /// อัปเดตสินค้าทั้งหมด
    /// </summary>
    [HttpPut("{id:int}")]
    [Authorize(Roles = "Admin,Manager")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Update(int id, [FromBody] UpdateProductDto dto)
    {
        if (id != dto.Id)
            return BadRequest(new { message = "ID ใน URL และ Body ไม่ตรงกัน" });
        
        var updated = await _service.UpdateAsync(id, dto);
        
        if (!updated)
            return NotFound();
        
        return NoContent();
    }
    
    /// <summary>
    /// ลบสินค้า
    /// </summary>
    [HttpDelete("{id:int}")]
    [Authorize(Roles = "Admin")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id)
    {
        var deleted = await _service.DeleteAsync(id);
        
        if (!deleted)
            return NotFound();
        
        _logger.LogWarning("ลบสินค้า ID: {Id}", id);
        
        return NoContent();
    }
}
```

### Step 577: Filtering, Sorting, Pagination

```csharp
// Service Layer สำหรับ Paged Query
public class ProductService : IProductService
{
    private readonly AppDbContext _context;
    private readonly IMapper _mapper;
    
    public ProductService(AppDbContext context, IMapper mapper)
    {
        _context = context;
        _mapper = mapper;
    }
    
    public async Task<PagedResult<ProductResponseDto>> GetPagedAsync(
        ProductFilterDto filter)
    {
        // เริ่มต้นจาก IQueryable
        var query = _context.Products
            .Include(p => p.Category)
            .AsNoTracking()
            .AsQueryable();
        
        // === Filtering ===
        if (!string.IsNullOrWhiteSpace(filter.Name))
        {
            query = query.Where(p => 
                p.Name.Contains(filter.Name));
        }
        
        if (filter.MinPrice.HasValue)
        {
            query = query.Where(p => p.Price >= filter.MinPrice.Value);
        }
        
        if (filter.MaxPrice.HasValue)
        {
            query = query.Where(p => p.Price <= filter.MaxPrice.Value);
        }
        
        if (filter.CategoryId.HasValue)
        {
            query = query.Where(p => p.CategoryId == filter.CategoryId.Value);
        }
        
        if (filter.IsActive.HasValue)
        {
            query = query.Where(p => p.IsActive == filter.IsActive.Value);
        }
        
        // === นับจำนวนทั้งหมด ===
        var totalCount = await query.CountAsync();
        
        // === Sorting ===
        query = filter.SortBy?.ToLower() switch
        {
            "name" => filter.SortOrder == "desc" 
                ? query.OrderByDescending(p => p.Name)
                : query.OrderBy(p => p.Name),
            "price" => filter.SortOrder == "desc"
                ? query.OrderByDescending(p => p.Price)
                : query.OrderBy(p => p.Price),
            "stock" => filter.SortOrder == "desc"
                ? query.OrderByDescending(p => p.Stock)
                : query.OrderBy(p => p.Stock),
            "createdat" => filter.SortOrder == "desc"
                ? query.OrderByDescending(p => p.CreatedAt)
                : query.OrderBy(p => p.CreatedAt),
            _ => query.OrderBy(p => p.Id)
        };
        
        // === Pagination ===
        var pageSize = filter.ValidatedPageSize;
        var page = Math.Max(1, filter.Page);
        
        var items = await query
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync();
        
        var dtos = _mapper.Map<IEnumerable<ProductResponseDto>>(items);
        
        return new PagedResult<ProductResponseDto>
        {
            Items = dtos,
            TotalCount = totalCount,
            CurrentPage = page,
            PageSize = pageSize
        };
    }
}
```

### Step 578: File Upload Endpoint

```csharp
// DTO สำหรับ File Upload
public class ProductImageUploadDto
{
    [Required(ErrorMessage = "กรุณาเลือกไฟล์รูปภาพ")]
    public IFormFile Image { get; set; } = null!;
    
    [StringLength(500, ErrorMessage = "คำอธิบายรูปไม่เกิน 500 ตัวอักษร")]
    public string? AltText { get; set; }
}

// File Upload Controller Methods
[HttpPost("{id:int}/images")]
[Authorize(Roles = "Admin,Manager")]
[Consumes("multipart/form-data")]
[ProducesResponseType(typeof(ProductImageDto), StatusCodes.Status201Created)]
[ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
[RequestSizeLimit(10 * 1024 * 1024)] // 10 MB
public async Task<ActionResult<ProductImageDto>> UploadImage(
    int id,
    [FromForm] ProductImageUploadDto dto)
{
    // ตรวจสอบว่าสินค้ามีอยู่
    if (!await _service.ExistsAsync(id))
        return NotFound($"ไม่พบสินค้า ID: {id}");
    
    var file = dto.Image;
    
    // Validate ประเภทไฟล์
    var allowedExtensions = new[] { ".jpg", ".jpeg", ".png", ".webp" };
    var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
    
    if (!allowedExtensions.Contains(extension))
    {
        return BadRequest(new ValidationProblemDetails
        {
            Title = "ประเภทไฟล์ไม่ถูกต้อง",
            Errors = 
            {
                ["image"] = new[] { 
                    $"รองรับเฉพาะ {string.Join(", ", allowedExtensions)}" 
                }
            }
        });
    }
    
    // Validate ขนาดไฟล์
    if (file.Length > 5 * 1024 * 1024) // 5 MB
    {
        return BadRequest(new ValidationProblemDetails
        {
            Title = "ไฟล์ใหญ่เกินไป",
            Errors = 
            {
                ["image"] = new[] { "ขนาดไฟล์ต้องไม่เกิน 5 MB" }
            }
        });
    }
    
    // Validate MIME Type จริงๆ (ไม่เชื่อ Extension อย่างเดียว)
    using var memoryStream = new MemoryStream();
    await file.CopyToAsync(memoryStream);
    var bytes = memoryStream.ToArray();
    
    if (!IsValidImage(bytes))
    {
        return BadRequest(new ValidationProblemDetails
        {
            Title = "ไฟล์ไม่ใช่รูปภาพ",
            Errors = 
            {
                ["image"] = new[] { "ไฟล์ไม่ใช่รูปภาพที่ถูกต้อง" }
            }
        });
    }
    
    // บันทึกไฟล์
    var fileName = $"{Guid.NewGuid()}{extension}";
    var uploadPath = Path.Combine(
        Directory.GetCurrentDirectory(), "uploads", "products", id.ToString());
    
    Directory.CreateDirectory(uploadPath);
    
    var filePath = Path.Combine(uploadPath, fileName);
    
    await using var fileStream = new FileStream(filePath, FileMode.Create);
    await file.CopyToAsync(fileStream);
    
    // บันทึก URL ลงใน Database
    var imageUrl = $"/uploads/products/{id}/{fileName}";
    var image = await _service.AddImageAsync(id, imageUrl, dto.AltText);
    
    _logger.LogInformation(
        "อัปโหลดรูปภาพสำหรับสินค้า ID: {ProductId}, ไฟล์: {FileName}",
        id, fileName);
    
    return Created(imageUrl, image);
}

// ตรวจสอบ Magic Bytes ของรูปภาพ
private static bool IsValidImage(byte[] bytes)
{
    // JPEG: FF D8 FF
    if (bytes.Length >= 3 && 
        bytes[0] == 0xFF && bytes[1] == 0xD8 && bytes[2] == 0xFF)
        return true;
    
    // PNG: 89 50 4E 47
    if (bytes.Length >= 4 && 
        bytes[0] == 0x89 && bytes[1] == 0x50 && 
        bytes[2] == 0x4E && bytes[3] == 0x47)
        return true;
    
    // WebP: 52 49 46 46 ... 57 45 42 50
    if (bytes.Length >= 12 &&
        bytes[0] == 0x52 && bytes[1] == 0x49 && 
        bytes[2] == 0x46 && bytes[3] == 0x46 &&
        bytes[8] == 0x57 && bytes[9] == 0x45 && 
        bytes[10] == 0x42 && bytes[11] == 0x50)
        return true;
    
    return false;
}

// ดาวน์โหลดไฟล์
[HttpGet("{id:int}/images/{imageId:int}/download")]
[AllowAnonymous]
public async Task<IActionResult> DownloadImage(int id, int imageId)
{
    var image = await _service.GetImageAsync(id, imageId);
    
    if (image is null)
        return NotFound();
    
    var filePath = Path.Combine(
        Directory.GetCurrentDirectory(), image.FilePath.TrimStart('/'));
    
    if (!System.IO.File.Exists(filePath))
        return NotFound("ไฟล์ไม่พบในระบบ");
    
    var fileBytes = await System.IO.File.ReadAllBytesAsync(filePath);
    var contentType = GetContentType(filePath);
    
    return File(fileBytes, contentType, Path.GetFileName(filePath));
}

private static string GetContentType(string path)
{
    return Path.GetExtension(path).ToLowerInvariant() switch
    {
        ".jpg" or ".jpeg" => "image/jpeg",
        ".png" => "image/png",
        ".webp" => "image/webp",
        _ => "application/octet-stream"
    };
}
```

### Step 579: Versioned API v1/v2

```csharp
// Program.cs - ตั้งค่า API Versioning
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true; // เพิ่ม api-supported-versions header
    
    // รองรับหลายวิธีส่ง Version
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),     // /api/v1/products
        new HeaderApiVersionReader("X-API-Version"), // Header
        new QueryStringApiVersionReader("api-version") // ?api-version=1.0
    );
});

builder.Services.AddVersionedApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// V1 Controller
[ApiController]
[Route("api/v{version:apiVersion}/products")]
[ApiVersion("1.0")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProductResponseDtoV1>>> GetAll()
    {
        // V1 ส่งคืนข้อมูลพื้นฐาน
        // ...
    }
}

// V2 Controller - เพิ่ม Features ใหม่
[ApiController]
[Route("api/v{version:apiVersion}/products")]
[ApiVersion("2.0")]
[ApiVersion("2.1")] // รองรับหลาย Minor Version
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<PagedResult<ProductResponseDtoV2>>> GetAll(
        [FromQuery] ProductFilterDto filter)
    {
        // V2 เพิ่ม Pagination และ Filtering
        // ...
    }
    
    // Endpoint ที่มีใน V2 เท่านั้น
    [HttpGet("trending")]
    [MapToApiVersion("2.0")]
    public async Task<ActionResult<IEnumerable<ProductResponseDtoV2>>> GetTrending()
    {
        // ...
    }
    
    // Feature ใหม่ใน V2.1
    [HttpGet("recommendations")]
    [MapToApiVersion("2.1")]
    public async Task<ActionResult<IEnumerable<ProductResponseDtoV2>>> GetRecommendations()
    {
        // ...
    }
}

// Swagger ที่รองรับหลาย Version
builder.Services.AddSwaggerGen(options =>
{
    // สร้าง Swagger Doc สำหรับแต่ละ Version
    var provider = builder.Services
        .BuildServiceProvider()
        .GetRequiredService<IApiVersionDescriptionProvider>();
    
    foreach (var description in provider.ApiVersionDescriptions)
    {
        options.SwaggerDoc(description.GroupName, new OpenApiInfo
        {
            Title = $"Products API {description.ApiVersion}",
            Version = description.ApiVersion.ToString(),
            Description = description.IsDeprecated 
                ? "API Version นี้ถูก Deprecate แล้ว" 
                : "API ที่ใช้งานอยู่"
        });
    }
});
```

### Step 580: Integration Tests ด้วย WebApplicationFactory

```csharp
// Tests/ProductsApiIntegrationTests.cs
public class ProductsApiIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public ProductsApiIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // แทนที่ Database ด้วย In-Memory Database
                var descriptor = services.SingleOrDefault(d =>
                    d.ServiceType == typeof(DbContextOptions<AppDbContext>));
                
                if (descriptor != null)
                    services.Remove(descriptor);
                
                services.AddDbContext<AppDbContext>(options =>
                    options.UseInMemoryDatabase($"TestDb_{Guid.NewGuid()}"));
                
                // Seed ข้อมูลทดสอบ
                var sp = services.BuildServiceProvider();
                using var scope = sp.CreateScope();
                var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
                db.Database.EnsureCreated();
                SeedTestData(db);
            });
        });
        
        _client = _factory.CreateClient();
    }
    
    private static void SeedTestData(AppDbContext db)
    {
        if (!db.Categories.Any())
        {
            db.Categories.Add(new Category { Id = 1, Name = "Electronics" });
            db.SaveChanges();
        }
        
        if (!db.Products.Any())
        {
            db.Products.AddRange(
                new Product
                {
                    Id = 1,
                    Name = "Laptop",
                    Price = 50000,
                    Stock = 10,
                    Sku = "LAP-001",
                    CategoryId = 1,
                    IsActive = true,
                    CreatedAt = DateTime.UtcNow
                },
                new Product
                {
                    Id = 2,
                    Name = "Mouse",
                    Price = 500,
                    Stock = 50,
                    Sku = "MOU-001",
                    CategoryId = 1,
                    IsActive = true,
                    CreatedAt = DateTime.UtcNow
                }
            );
            db.SaveChanges();
        }
    }
    
    [Fact]
    public async Task GetAll_ReturnsOkWithProducts()
    {
        // Act
        var response = await _client.GetAsync("/api/v1/products");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var content = await response.Content.ReadFromJsonAsync<PagedResult<ProductResponseDto>>();
        content.Should().NotBeNull();
        content!.Items.Should().NotBeEmpty();
        content.TotalCount.Should().BeGreaterThan(0);
    }
    
    [Fact]
    public async Task GetById_WithValidId_ReturnsProduct()
    {
        // Act
        var response = await _client.GetAsync("/api/v1/products/1");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var product = await response.Content.ReadFromJsonAsync<ProductResponseDto>();
        product.Should().NotBeNull();
        product!.Id.Should().Be(1);
        product.Name.Should().Be("Laptop");
    }
    
    [Fact]
    public async Task GetById_WithInvalidId_ReturnsNotFound()
    {
        // Act
        var response = await _client.GetAsync("/api/v1/products/9999");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
    
    [Fact]
    public async Task Create_WithValidData_ReturnsCreated()
    {
        // Arrange - ต้องมี Token ก่อน
        var token = GenerateTestToken("Admin");
        _client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);
        
        var dto = new CreateProductDto
        {
            Name = "Keyboard",
            Price = 1500,
            Stock = 30,
            Sku = "KEY-001",
            CategoryId = 1
        };
        
        // Act
        var response = await _client.PostAsJsonAsync("/api/v1/products", dto);
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
        
        var created = await response.Content.ReadFromJsonAsync<ProductResponseDto>();
        created.Should().NotBeNull();
        created!.Name.Should().Be("Keyboard");
        created.Price.Should().Be(1500);
    }
    
    [Fact]
    public async Task Create_WithInvalidData_ReturnsBadRequest()
    {
        // Arrange
        var token = GenerateTestToken("Admin");
        _client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);
        
        var dto = new CreateProductDto
        {
            Name = "", // ชื่อว่าง - ไม่ valid
            Price = -100, // ราคาติดลบ - ไม่ valid
            Sku = "invalid" // รูปแบบ SKU ไม่ถูกต้อง
        };
        
        // Act
        var response = await _client.PostAsJsonAsync("/api/v1/products", dto);
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
        
        var problem = await response.Content
            .ReadFromJsonAsync<ValidationProblemDetails>();
        problem.Should().NotBeNull();
        problem!.Errors.Should().ContainKey("Name");
        problem.Errors.Should().ContainKey("Price");
    }
    
    [Fact]
    public async Task Create_WithoutAuth_ReturnsUnauthorized()
    {
        // Arrange - ไม่ใส่ Token
        _client.DefaultRequestHeaders.Authorization = null;
        
        var dto = new CreateProductDto
        {
            Name = "Test Product",
            Price = 100,
            Sku = "TST-001",
            CategoryId = 1
        };
        
        // Act
        var response = await _client.PostAsJsonAsync("/api/v1/products", dto);
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }
    
    [Fact]
    public async Task Delete_WithAdminRole_ReturnsNoContent()
    {
        // Arrange
        var token = GenerateTestToken("Admin");
        _client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);
        
        // Act
        var response = await _client.DeleteAsync("/api/v1/products/2");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);
        
        // ตรวจสอบว่าลบแล้วจริง
        var getResponse = await _client.GetAsync("/api/v1/products/2");
        getResponse.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
    
    [Fact]
    public async Task GetAll_WithFiltering_ReturnsFilteredResults()
    {
        // Act
        var response = await _client.GetAsync(
            "/api/v1/products?name=Laptop&minPrice=10000&maxPrice=100000");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var result = await response.Content
            .ReadFromJsonAsync<PagedResult<ProductResponseDto>>();
        
        result.Should().NotBeNull();
        result!.Items.Should().AllSatisfy(p =>
        {
            p.Name.Should().Contain("Laptop", StringComparison.OrdinalIgnoreCase);
            p.Price.Should().BeGreaterThan(10000);
            p.Price.Should().BeLessThan(100000);
        });
    }
    
    [Fact]
    public async Task GetAll_WithPagination_ReturnsPaginatedResults()
    {
        // Act
        var response = await _client.GetAsync(
            "/api/v1/products?page=1&pageSize=1");
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        response.Headers.Should().ContainKey("X-Total-Count");
        response.Headers.Should().ContainKey("X-Total-Pages");
        
        var result = await response.Content
            .ReadFromJsonAsync<PagedResult<ProductResponseDto>>();
        
        result.Should().NotBeNull();
        result!.Items.Should().HaveCount(1);
        result.PageSize.Should().Be(1);
    }
    
    private static string GenerateTestToken(string role)
    {
        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes("test-secret-key-for-testing-purposes-only"));
        
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, "test-user-id"),
            new Claim(ClaimTypes.Email, "test@example.com"),
            new Claim(ClaimTypes.Role, role)
        };
        
        var token = new JwtSecurityToken(
            issuer: "test-issuer",
            audience: "test-audience",
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: credentials);
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

### appsettings.json ตัวอย่าง

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=ProductsDb;Trusted_Connection=True;"
  },
  "Jwt": {
    "Key": "your-super-secret-key-minimum-32-characters-long",
    "Issuer": "https://api.example.com",
    "Audience": "https://app.example.com",
    "AccessTokenExpireMinutes": "15",
    "RefreshTokenExpireDays": "7"
  },
  "Cors": {
    "AllowedOrigins": [
      "https://app.example.com",
      "https://www.example.com"
    ]
  },
  "FileUpload": {
    "MaxFileSizeMB": 5,
    "AllowedExtensions": [".jpg", ".jpeg", ".png", ".webp"],
    "UploadPath": "uploads/products"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

### Project Structure แนะนำ

```
MyApi/
├── Controllers/
│   ├── v1/
│   │   └── ProductsController.cs
│   └── v2/
│       └── ProductsController.cs
├── DTOs/
│   ├── Requests/
│   │   ├── CreateProductDto.cs
│   │   └── UpdateProductDto.cs
│   └── Responses/
│       ├── ProductResponseDto.cs
│       └── PagedResult.cs
├── Entities/
│   ├── Product.cs
│   └── Category.cs
├── Infrastructure/
│   ├── Data/
│   │   ├── AppDbContext.cs
│   │   └── Migrations/
│   └── Repositories/
│       └── ProductRepository.cs
├── Mappings/
│   └── ProductMappingProfile.cs
├── Middleware/
│   ├── ExceptionHandlingMiddleware.cs
│   ├── CorrelationIdMiddleware.cs
│   └── RequestLoggingMiddleware.cs
├── Services/
│   ├── Interfaces/
│   │   └── IProductService.cs
│   └── ProductService.cs
├── Validators/
│   └── CreateProductDtoValidator.cs
└── Program.cs
```

---

## สรุป Steps 571-580

| Step | หัวข้อ | สิ่งสำคัญที่เรียน |
|------|--------|-------------------|
| 571 | ASP.NET Core Setup | Program.cs, Swagger, CORS, Health Checks |
| 572 | Controllers & Routes | [ApiController], HTTP Verbs, ActionResult<T> |
| 573 | DTOs & AutoMapper | Data Transfer, Mapping, ProblemDetails |
| 574 | Middleware | Custom Middleware, Exception Handling, Rate Limiting |
| 575 | Auth & Authorization | JWT, [Authorize], Policy-based |
| 576 | Full CRUD Controller | Complete ProductsController |
| 577 | Filter/Sort/Pagination | IQueryable, Dynamic Query Building |
| 578 | File Upload | IFormFile, Validation, Magic Bytes |
| 579 | API Versioning | URL/Header/Query Versioning |
| 580 | Integration Tests | WebApplicationFactory, In-Memory DB |

### Best Practices สรุป

1. **ใช้ `[ApiController]`** เสมอสำหรับ API Controllers
2. **ใช้ `ActionResult<T>`** แทน `IActionResult` เพื่อให้ Swagger สร้าง Schema ได้ถูกต้อง
3. **ตรวจสอบ Input** ด้วย Data Annotations และ FluentValidation
4. **ใช้ DTOs** แยก Domain Model จาก API Contract เสมอ
5. **Handle Exceptions** ใน Middleware ไม่ใช่ใน Controller
6. **Log ทุก Request** ด้วย Structured Logging
7. **ใช้ ProblemDetails** ตามมาตรฐาน RFC 7807
8. **Test Integration** ด้วย WebApplicationFactory
9. **Version API** ตั้งแต่เริ่มต้น
10. **ใช้ Rate Limiting** ป้องกัน Abuse

---

## การนำทาง

- **ก่อนหน้า**: [Part 57: Blazor Introduction](part57-blazor-intro.md)
- **ถัดไป**: [Part 59: SignalR Real-time Communication](part59-signalr.md)
