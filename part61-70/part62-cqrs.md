# Part 62: CQRS (Command Query Responsibility Segregation)
## ขั้นตอนที่ 611-620: การแยก Command และ Query อย่างมืออาชีพ

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจหลักการ CQRS และว่าทำไมต้องใช้
- ใช้ MediatR สร้าง Commands และ Queries
- Pipeline Behaviors สำหรับ Validation, Logging, Caching
- Read Models ที่ปรับแต่งสำหรับการ Query
- สร้างระบบ Product Catalog ด้วย CQRS ครบวงจร

---

## ขั้นตอนที่ 611: CQRS Overview

### CQRS คืออะไร?

CQRS ย่อมาจาก **Command Query Responsibility Segregation** หรือ "การแยกความรับผิดชอบระหว่าง Command และ Query"

แนวคิดหลักคือ: **การเปลี่ยนแปลงข้อมูล (Write) และการอ่านข้อมูล (Read) ควรใช้ Model คนละชุดกัน**

```
Traditional Architecture (ปัญหา):
┌─────────────────────────────────────────────────┐
│              Single Model (Product)              │
│  - ใช้สำหรับ Create/Update/Delete              │
│  - ใช้สำหรับ GetById/GetList/Search            │
│  → Model เดียวทำหน้าที่ทั้งหมด = ซับซ้อน        │
└─────────────────────────────────────────────────┘

CQRS Architecture (แก้ปัญหา):
┌────────────────────┐    ┌────────────────────────┐
│   Command Side     │    │      Query Side         │
│   (Write Model)    │    │     (Read Model)        │
│                    │    │                         │
│ - CreateProduct    │    │ - GetProductById        │
│ - UpdateProduct    │    │ - GetProductList        │
│ - DeleteProduct    │    │ - SearchProducts        │
│                    │    │                         │
│ → เน้นความถูกต้อง  │    │ → เน้นความเร็ว          │
│   ของ Business     │    │   และ Performance       │
│   Logic            │    │                         │
└────────────────────┘    └────────────────────────┘
```

### ทำไมต้องใช้ CQRS?

**ปัญหาในระบบที่ไม่ใช้ CQRS:**

1. **Scalability ไม่สมดุล**: โดยทั่วไป Read มากกว่า Write 10-100 เท่า แต่ใช้ Model เดียวกัน
2. **Model ซับซ้อน**: Product entity ต้องรองรับทั้ง Business Rules และ Query Requirements
3. **Performance**: Query ต้อง Load ทั้ง Aggregate แม้จะต้องการแค่บางฟิลด์
4. **Locking**: Write operations lock records ที่ Read operations กำลังใช้

**ประโยชน์ของ CQRS:**
- Read Model ปรับแต่งได้อิสระ ไม่กระทบ Business Logic
- Scale Read และ Write แยกกันได้
- Query ง่ายกว่าเพราะไม่ต้องยึดตาม Domain Model
- เพิ่ม Cache ฝั่ง Read ได้โดยไม่กระทบ Write

### Simple CQRS vs Full CQRS

```
Simple CQRS (ฐานข้อมูลเดียว, Model ต่างกัน):
┌─────────────┐              ┌─────────────┐
│  Commands   │──────────┐   │   Queries   │
│  (Write)    │          │   │   (Read)    │
└─────────────┘          │   └─────────────┘
                         ▼          │
                  ┌─────────────┐   │
                  │  Same DB    │◄──┘
                  │  (SQL)      │
                  └─────────────┘
→ ง่ายกว่า, เหมาะกับส่วนใหญ่ของ Application

Full CQRS (ฐานข้อมูลแยก, Eventual Consistency):
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│  Commands   │───►│   Write DB   │    │   Queries   │
│  (Write)    │    │  (SQL/NoSQL) │    │   (Read)    │
└─────────────┘    └──────┬───────┘    └──────▲──────┘
                          │                    │
                          │    Domain Events   │
                          └───────────────────►│
                                        ┌──────┴──────┐
                                        │   Read DB   │
                                        │  (NoSQL/    │
                                        │   Elastic)  │
                                        └─────────────┘
→ ซับซ้อนกว่า, เหมาะเมื่อ Read/Write ต้องการ Scale แยกกันจริงๆ
```

---

## ขั้นตอนที่ 612: MediatR Commands

### ติดตั้ง Package ที่จำเป็น

```bash
dotnet add package MediatR
dotnet add package MediatR.Extensions.Microsoft.DependencyInjection
dotnet add package FluentValidation
dotnet add package FluentValidation.DependencyInjectionExtensions
```

### Result Pattern สำหรับ Commands

แทนที่จะ throw exception เราใช้ Result<T> เพื่อส่งกลับผลลัพธ์อย่างชัดเจน:

```csharp
// Application/Common/Result.cs
namespace ProductCatalog.Application.Common;

public class Result<T>
{
    public bool IsSuccess { get; private set; }
    public T? Value { get; private set; }
    public string? Error { get; private set; }
    public IReadOnlyList<string> Errors { get; private set; } = new List<string>();

    private Result() { }

    public static Result<T> Success(T value) =>
        new() { IsSuccess = true, Value = value };

    public static Result<T> Failure(string error) =>
        new() { IsSuccess = false, Error = error };

    public static Result<T> Failure(IEnumerable<string> errors) =>
        new() { IsSuccess = false, Errors = errors.ToList() };

    public static implicit operator bool(Result<T> result) => result.IsSuccess;
}

public class Result
{
    public bool IsSuccess { get; private set; }
    public string? Error { get; private set; }
    public IReadOnlyList<string> Errors { get; private set; } = new List<string>();

    public static Result Success() => new() { IsSuccess = true };
    public static Result Failure(string error) => new() { IsSuccess = false, Error = error };
    public static Result Failure(IEnumerable<string> errors) =>
        new() { IsSuccess = false, Errors = errors.ToList() };
}
```

### CreateProductCommand

```csharp
// Application/Products/Commands/CreateProduct/CreateProductCommand.cs
namespace ProductCatalog.Application.Products.Commands.CreateProduct;

// Command คือ Intent (ความตั้งใจ) ที่จะเปลี่ยนแปลง State
public record CreateProductCommand(
    string Name,
    string Description,
    decimal Price,
    int StockQuantity,
    string CategoryName
) : IRequest<Result<Guid>>;
// ↑ IRequest<TResponse> บอก MediatR ว่า Command นี้คืน Result<Guid>

// Application/Products/Commands/CreateProduct/CreateProductCommandHandler.cs
public class CreateProductCommandHandler
    : IRequestHandler<CreateProductCommand, Result<Guid>>
{
    private readonly IProductRepository _repository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly ILogger<CreateProductCommandHandler> _logger;

    public CreateProductCommandHandler(
        IProductRepository repository,
        IUnitOfWork unitOfWork,
        ILogger<CreateProductCommandHandler> logger)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
        _logger = logger;
    }

    public async Task<Result<Guid>> Handle(
        CreateProductCommand request,
        CancellationToken cancellationToken)
    {
        // ตรวจสอบว่ามีชื่อซ้ำหรือไม่
        var existingProduct = await _repository.FindByNameAsync(request.Name, cancellationToken);
        if (existingProduct is not null)
            return Result<Guid>.Failure($"Product with name '{request.Name}' already exists.");

        // สร้าง Domain Entity (Business Logic อยู่ใน Entity)
        var product = Product.Create(
            request.Name,
            request.Description,
            request.Price,
            request.StockQuantity,
            request.CategoryName
        );

        await _repository.AddAsync(product, cancellationToken);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        _logger.LogInformation("Product created with Id: {ProductId}", product.Id);

        return Result<Guid>.Success(product.Id);
    }
}
```

### UpdateProductCommand

```csharp
// Application/Products/Commands/UpdateProduct/UpdateProductCommand.cs
public record UpdateProductCommand(
    Guid Id,
    string Name,
    string Description,
    decimal Price,
    int StockQuantity
) : IRequest<Result>;

public class UpdateProductCommandHandler : IRequestHandler<UpdateProductCommand, Result>
{
    private readonly IProductRepository _repository;
    private readonly IUnitOfWork _unitOfWork;

    public UpdateProductCommandHandler(
        IProductRepository repository,
        IUnitOfWork unitOfWork)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
    }

    public async Task<Result> Handle(
        UpdateProductCommand request,
        CancellationToken cancellationToken)
    {
        var product = await _repository.GetByIdAsync(request.Id, cancellationToken);
        if (product is null)
            return Result.Failure($"Product with Id '{request.Id}' not found.");

        // เรียก Domain Method เพื่อ Update (ไม่ Set Property โดยตรง)
        product.UpdateDetails(request.Name, request.Description, request.Price);
        product.SetStockQuantity(request.StockQuantity);

        _repository.Update(product);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        return Result.Success();
    }
}
```

### DeleteProductCommand

```csharp
// Application/Products/Commands/DeleteProduct/DeleteProductCommand.cs
public record DeleteProductCommand(Guid Id) : IRequest<Result>;

public class DeleteProductCommandHandler : IRequestHandler<DeleteProductCommand, Result>
{
    private readonly IProductRepository _repository;
    private readonly IUnitOfWork _unitOfWork;

    public DeleteProductCommandHandler(
        IProductRepository repository,
        IUnitOfWork unitOfWork)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
    }

    public async Task<Result> Handle(
        DeleteProductCommand request,
        CancellationToken cancellationToken)
    {
        var product = await _repository.GetByIdAsync(request.Id, cancellationToken);
        if (product is null)
            return Result.Failure($"Product with Id '{request.Id}' not found.");

        // Soft delete หรือ Hard delete ขึ้นอยู่กับ Business Rule
        product.MarkAsDeleted();

        _repository.Update(product);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        return Result.Success();
    }
}
```

---

## ขั้นตอนที่ 613: MediatR Queries

Queries ต่างจาก Commands ตรงที่: **ไม่เปลี่ยนแปลง State** และ **เน้นความเร็ว**

### GetProductByIdQuery

```csharp
// Application/Products/Queries/GetProductById/GetProductByIdQuery.cs
public record GetProductByIdQuery(Guid Id) : IRequest<Result<ProductDto>>;

// DTO สำหรับ Read (ต่างจาก Domain Entity)
public record ProductDto(
    Guid Id,
    string Name,
    string Description,
    decimal Price,
    int StockQuantity,
    string CategoryName,
    DateTime CreatedAt,
    bool IsAvailable
);

public class GetProductByIdQueryHandler
    : IRequestHandler<GetProductByIdQuery, Result<ProductDto>>
{
    private readonly IApplicationDbContext _context;

    public GetProductByIdQueryHandler(IApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<Result<ProductDto>> Handle(
        GetProductByIdQuery request,
        CancellationToken cancellationToken)
    {
        // AsNoTracking() → ไม่ track entity ทำให้ Query เร็วขึ้น
        // Select() → ดึงเฉพาะ field ที่ต้องการ (ไม่ Load ทั้ง Entity)
        var product = await _context.Products
            .AsNoTracking()
            .Where(p => p.Id == request.Id && !p.IsDeleted)
            .Select(p => new ProductDto(
                p.Id,
                p.Name,
                p.Description,
                p.Price,
                p.StockQuantity,
                p.Category.Name,   // ← Join กับ Category
                p.CreatedAt,
                p.StockQuantity > 0
            ))
            .FirstOrDefaultAsync(cancellationToken);

        if (product is null)
            return Result<ProductDto>.Failure($"Product with Id '{request.Id}' not found.");

        return Result<ProductDto>.Success(product);
    }
}
```

### GetProductListQuery พร้อม Filtering และ Paging

```csharp
// Application/Products/Queries/GetProductList/GetProductListQuery.cs

// Paged Result สำหรับรองรับ Pagination
public record PagedResult<T>(
    IReadOnlyList<T> Items,
    int TotalCount,
    int PageNumber,
    int PageSize
)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasNextPage => PageNumber < TotalPages;
    public bool HasPreviousPage => PageNumber > 1;
}

// Query พร้อม Filter Parameters
public record GetProductListQuery(
    string? SearchTerm = null,
    string? CategoryName = null,
    decimal? MinPrice = null,
    decimal? MaxPrice = null,
    bool? InStockOnly = null,
    int PageNumber = 1,
    int PageSize = 20,
    string SortBy = "Name",
    bool SortDescending = false
) : IRequest<Result<PagedResult<ProductDto>>>;

public class GetProductListQueryHandler
    : IRequestHandler<GetProductListQuery, Result<PagedResult<ProductDto>>>
{
    private readonly IApplicationDbContext _context;

    public GetProductListQueryHandler(IApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<Result<PagedResult<ProductDto>>> Handle(
        GetProductListQuery request,
        CancellationToken cancellationToken)
    {
        // เริ่มจาก IQueryable ที่ยังไม่ execute
        var query = _context.Products
            .AsNoTracking()
            .Where(p => !p.IsDeleted)
            .AsQueryable();

        // Apply Filters (เพิ่มทีละ condition)
        if (!string.IsNullOrWhiteSpace(request.SearchTerm))
        {
            var term = request.SearchTerm.ToLower();
            query = query.Where(p =>
                p.Name.ToLower().Contains(term) ||
                p.Description.ToLower().Contains(term));
        }

        if (!string.IsNullOrWhiteSpace(request.CategoryName))
            query = query.Where(p => p.Category.Name == request.CategoryName);

        if (request.MinPrice.HasValue)
            query = query.Where(p => p.Price >= request.MinPrice.Value);

        if (request.MaxPrice.HasValue)
            query = query.Where(p => p.Price <= request.MaxPrice.Value);

        if (request.InStockOnly == true)
            query = query.Where(p => p.StockQuantity > 0);

        // Count ก่อน Pagination (สำหรับ TotalCount)
        var totalCount = await query.CountAsync(cancellationToken);

        // Apply Sorting
        query = request.SortBy switch
        {
            "Price" => request.SortDescending
                ? query.OrderByDescending(p => p.Price)
                : query.OrderBy(p => p.Price),
            "CreatedAt" => request.SortDescending
                ? query.OrderByDescending(p => p.CreatedAt)
                : query.OrderBy(p => p.CreatedAt),
            _ => request.SortDescending
                ? query.OrderByDescending(p => p.Name)
                : query.OrderBy(p => p.Name)
        };

        // Apply Pagination และ Projection
        var items = await query
            .Skip((request.PageNumber - 1) * request.PageSize)
            .Take(request.PageSize)
            .Select(p => new ProductDto(
                p.Id,
                p.Name,
                p.Description,
                p.Price,
                p.StockQuantity,
                p.Category.Name,
                p.CreatedAt,
                p.StockQuantity > 0
            ))
            .ToListAsync(cancellationToken);

        var result = new PagedResult<ProductDto>(
            items,
            totalCount,
            request.PageNumber,
            request.PageSize
        );

        return Result<PagedResult<ProductDto>>.Success(result);
    }
}
```

---

## ขั้นตอนที่ 614: Pipeline Behaviors

Pipeline Behaviors คือ "Middleware" สำหรับ MediatR ทำงานก่อน/หลัง Handler

```
Request Flow:
[Request] → [LoggingBehavior] → [ValidationBehavior] → [CachingBehavior] → [Handler] → [Response]
              ↑ timing start     ↑ validate input        ↑ cache check
                                                                            ↓
[Response] ← [LoggingBehavior] ← [ValidationBehavior] ← [CachingBehavior] ← [Handler]
               ↑ timing end        (pass through)         ↑ cache store
```

### ValidationBehavior

```csharp
// Application/Common/Behaviors/ValidationBehavior.cs
using FluentValidation;

public class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
    where TResponse : class
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        // ถ้าไม่มี Validator สำหรับ Request นี้ ให้ผ่านไปเลย
        if (!_validators.Any())
            return await next();

        // Run all validators concurrently
        var context = new ValidationContext<TRequest>(request);
        var validationResults = await Task.WhenAll(
            _validators.Select(v => v.ValidateAsync(context, cancellationToken)));

        // รวม Errors ทั้งหมด
        var failures = validationResults
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Any())
        {
            // ต้อง create Result<> ด้วย Reflection เพราะ Generic constraint ซับซ้อน
            var errors = failures.Select(f => f.ErrorMessage).ToList();
            throw new ValidationException(failures);
        }

        return await next();
    }
}

// Validator สำหรับ CreateProductCommand
public class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("ชื่อสินค้าต้องไม่ว่างเปล่า")
            .MaximumLength(200).WithMessage("ชื่อสินค้าต้องไม่เกิน 200 ตัวอักษร");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("ราคาต้องมากกว่า 0")
            .LessThanOrEqualTo(999999.99m).WithMessage("ราคาต้องไม่เกิน 999,999.99");

        RuleFor(x => x.StockQuantity)
            .GreaterThanOrEqualTo(0).WithMessage("จำนวนสต็อกต้องไม่น้อยกว่า 0");

        RuleFor(x => x.CategoryName)
            .NotEmpty().WithMessage("หมวดหมู่ต้องไม่ว่างเปล่า");
    }
}
```

### LoggingBehavior

```csharp
// Application/Common/Behaviors/LoggingBehavior.cs
public class LoggingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;

        _logger.LogInformation(
            "Handling {RequestName}: {@Request}",
            requestName, request);

        var stopwatch = Stopwatch.StartNew();

        try
        {
            var response = await next();
            stopwatch.Stop();

            _logger.LogInformation(
                "Handled {RequestName} in {ElapsedMs}ms",
                requestName, stopwatch.ElapsedMilliseconds);

            // แจ้งเตือนถ้าใช้เวลานานเกินไป (Long Running Query)
            if (stopwatch.ElapsedMilliseconds > 500)
            {
                _logger.LogWarning(
                    "Long running request detected: {RequestName} took {ElapsedMs}ms",
                    requestName, stopwatch.ElapsedMilliseconds);
            }

            return response;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            _logger.LogError(ex,
                "Error handling {RequestName} after {ElapsedMs}ms",
                requestName, stopwatch.ElapsedMilliseconds);
            throw;
        }
    }
}
```

### CachingBehavior

```csharp
// Application/Common/Behaviors/CachingBehavior.cs

// Marker Interface สำหรับ Queries ที่ต้องการ Cache
public interface ICacheable
{
    string CacheKey { get; }
    TimeSpan? CacheDuration { get; }
}

// Query ที่ต้องการ Cache ให้ implement ICacheable
public record GetProductByIdQuery(Guid Id)
    : IRequest<Result<ProductDto>>, ICacheable
{
    // Cache key unique per product
    public string CacheKey => $"product:{Id}";
    public TimeSpan? CacheDuration => TimeSpan.FromMinutes(5);
}

public class CachingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<CachingBehavior<TRequest, TResponse>> _logger;

    public CachingBehavior(
        IMemoryCache cache,
        ILogger<CachingBehavior<TRequest, TResponse>> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        // ถ้า Request ไม่ใช่ Cacheable ให้ผ่านไปเลย
        if (request is not ICacheable cacheable)
            return await next();

        var cacheKey = cacheable.CacheKey;

        // ลองดึงจาก Cache ก่อน
        if (_cache.TryGetValue(cacheKey, out TResponse? cachedResponse))
        {
            _logger.LogDebug("Cache hit for {CacheKey}", cacheKey);
            return cachedResponse!;
        }

        // ไม่มีใน Cache ให้เรียก Handler จริงๆ
        _logger.LogDebug("Cache miss for {CacheKey}", cacheKey);
        var response = await next();

        // เก็บใน Cache
        var cacheOptions = new MemoryCacheEntryOptions();
        if (cacheable.CacheDuration.HasValue)
            cacheOptions.SetAbsoluteExpiration(cacheable.CacheDuration.Value);
        else
            cacheOptions.SetAbsoluteExpiration(TimeSpan.FromMinutes(10));

        _cache.Set(cacheKey, response, cacheOptions);

        return response;
    }
}
```

### TransactionBehavior (สำหรับ Commands เท่านั้น)

```csharp
// Application/Common/Behaviors/TransactionBehavior.cs

// Marker Interface สำหรับ Commands ที่ต้องการ Transaction
public interface ITransactional { }

// ใส่ ITransactional ใน Commands ที่ต้องการ
public record CreateProductCommand(...) : IRequest<Result<Guid>>, ITransactional;

public class TransactionBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IDbContext _dbContext;
    private readonly ILogger<TransactionBehavior<TRequest, TResponse>> _logger;

    public TransactionBehavior(
        IDbContext dbContext,
        ILogger<TransactionBehavior<TRequest, TResponse>> logger)
    {
        _dbContext = dbContext;
        _logger = logger;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        // ถ้าไม่ใช่ Command ที่ต้องการ Transaction ให้ผ่านไป
        if (request is not ITransactional)
            return await next();

        // เริ่ม Transaction
        await using var transaction = await _dbContext.Database
            .BeginTransactionAsync(cancellationToken);

        _logger.LogDebug("Transaction started for {RequestName}", typeof(TRequest).Name);

        try
        {
            var response = await next();
            await transaction.CommitAsync(cancellationToken);
            _logger.LogDebug("Transaction committed for {RequestName}", typeof(TRequest).Name);
            return response;
        }
        catch (Exception ex)
        {
            await transaction.RollbackAsync(cancellationToken);
            _logger.LogError(ex, "Transaction rolled back for {RequestName}", typeof(TRequest).Name);
            throw;
        }
    }
}
```

---

## ขั้นตอนที่ 615: Read Models

Read Models คือ Model ที่ออกแบบมาเพื่อการ Query โดยเฉพาะ มักเป็น Denormalized (รวมข้อมูลจากหลาย Table ไว้ในที่เดียว)

### ProductReadModel

```csharp
// Infrastructure/ReadModels/ProductReadModel.cs
// Read Model สำหรับ Product Catalog (Denormalized)
public class ProductReadModel
{
    public Guid Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public bool IsAvailable { get; set; }

    // ข้อมูล Category (Denormalized - ไม่ต้อง JOIN แล้ว)
    public Guid CategoryId { get; set; }
    public string CategoryName { get; set; } = string.Empty;
    public string CategorySlug { get; set; } = string.Empty;

    // ข้อมูล Supplier (Denormalized)
    public string SupplierName { get; set; } = string.Empty;
    public string SupplierCountry { get; set; } = string.Empty;

    // Metadata
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }

    // Computed fields (คำนวณไว้แล้ว ไม่ต้องคำนวณทุกครั้งที่ Query)
    public string PriceFormatted => Price.ToString("C2");
    public string StockStatus => StockQuantity switch
    {
        0 => "หมดสต็อก",
        <= 5 => "เหลือน้อย",
        _ => "มีสินค้า"
    };
}
```

### การสร้าง Read Model จาก Domain Events

```csharp
// Infrastructure/ReadModels/ProductReadModelProjection.cs
// Projection: แปลง Domain Events → Read Model

public class ProductReadModelProjection
    : INotificationHandler<ProductCreatedEvent>,
      INotificationHandler<ProductUpdatedEvent>,
      INotificationHandler<ProductDeletedEvent>
{
    private readonly IReadDbContext _readDb;

    public ProductReadModelProjection(IReadDbContext readDb)
    {
        _readDb = readDb;
    }

    // เมื่อสร้าง Product ใหม่
    public async Task Handle(
        ProductCreatedEvent notification,
        CancellationToken cancellationToken)
    {
        var readModel = new ProductReadModel
        {
            Id = notification.ProductId,
            Name = notification.Name,
            Description = notification.Description,
            Price = notification.Price,
            StockQuantity = notification.StockQuantity,
            IsAvailable = notification.StockQuantity > 0,
            CategoryId = notification.CategoryId,
            CategoryName = notification.CategoryName,
            CategorySlug = notification.CategorySlug,
            SupplierName = notification.SupplierName,
            SupplierCountry = notification.SupplierCountry,
            CreatedAt = notification.OccurredAt
        };

        await _readDb.ProductReadModels.AddAsync(readModel, cancellationToken);
        await _readDb.SaveChangesAsync(cancellationToken);
    }

    // เมื่ออัพเดท Product
    public async Task Handle(
        ProductUpdatedEvent notification,
        CancellationToken cancellationToken)
    {
        var readModel = await _readDb.ProductReadModels
            .FindAsync(new object[] { notification.ProductId }, cancellationToken);

        if (readModel is null) return;

        readModel.Name = notification.Name;
        readModel.Description = notification.Description;
        readModel.Price = notification.Price;
        readModel.StockQuantity = notification.StockQuantity;
        readModel.IsAvailable = notification.StockQuantity > 0;
        readModel.UpdatedAt = notification.OccurredAt;

        await _readDb.SaveChangesAsync(cancellationToken);
    }

    // เมื่อลบ Product
    public async Task Handle(
        ProductDeletedEvent notification,
        CancellationToken cancellationToken)
    {
        var readModel = await _readDb.ProductReadModels
            .FindAsync(new object[] { notification.ProductId }, cancellationToken);

        if (readModel is not null)
        {
            _readDb.ProductReadModels.Remove(readModel);
            await _readDb.SaveChangesAsync(cancellationToken);
        }
    }
}
```

### Sync vs Async Projection

```
Synchronous Projection (ง่าย แต่ coupling):
┌─────────────┐  save  ┌─────────────┐  update  ┌──────────────┐
│  Command    │───────►│  Write DB   │─────────►│  Read Model  │
│  Handler    │        │             │  (sync)  │   (same TX)  │
└─────────────┘        └─────────────┘          └──────────────┘
→ ง่ายกว่า, Consistent ทันที
→ Write DB และ Read Model อยู่ใน Transaction เดียวกัน

Asynchronous Projection (Outbox Pattern):
┌─────────────┐  save  ┌─────────────┐  publish  ┌─────────────┐
│  Command    │───────►│  Write DB   │──────────►│   Outbox    │
│  Handler    │        │  + Outbox   │           │   Table     │
└─────────────┘        └─────────────┘           └──────┬──────┘
                                                         │ background
                                                         │ worker
                                                         ▼
                                                  ┌──────────────┐
                                                  │  Read Model  │
                                                  │   Updated    │
                                                  └──────────────┘
→ ซับซ้อนกว่า, Eventual Consistency
→ Write DB และ Read Model อาจ Inconsistent ชั่วคราว
→ เหมาะเมื่อ Read DB เป็น NoSQL หรือ Elastic
```

---

## ขั้นตอนที่ 616-620: Complete CQRS Implementation

### โครงสร้าง Project

```
ProductCatalog/
├── ProductCatalog.Domain/
│   ├── Entities/
│   │   └── Product.cs
│   ├── Events/
│   │   ├── ProductCreatedEvent.cs
│   │   ├── ProductUpdatedEvent.cs
│   │   └── ProductDeletedEvent.cs
│   └── Interfaces/
│       ├── IProductRepository.cs
│       └── IUnitOfWork.cs
│
├── ProductCatalog.Application/
│   ├── Common/
│   │   ├── Result.cs
│   │   ├── PagedResult.cs
│   │   └── Behaviors/
│   │       ├── ValidationBehavior.cs
│   │       ├── LoggingBehavior.cs
│   │       ├── CachingBehavior.cs
│   │       └── TransactionBehavior.cs
│   ├── Products/
│   │   ├── Commands/
│   │   │   ├── CreateProduct/
│   │   │   │   ├── CreateProductCommand.cs
│   │   │   │   ├── CreateProductCommandHandler.cs
│   │   │   │   └── CreateProductCommandValidator.cs
│   │   │   ├── UpdateProduct/
│   │   │   └── DeleteProduct/
│   │   └── Queries/
│   │       ├── GetProductById/
│   │       └── GetProductList/
│   └── DependencyInjection.cs
│
├── ProductCatalog.Infrastructure/
│   ├── Persistence/
│   │   ├── ApplicationDbContext.cs
│   │   └── ProductRepository.cs
│   ├── ReadModels/
│   │   └── ProductReadModelProjection.cs
│   └── DependencyInjection.cs
│
└── ProductCatalog.Api/
    ├── Controllers/
    │   └── ProductsController.cs
    └── Program.cs
```

### Domain Entity

```csharp
// Domain/Entities/Product.cs
namespace ProductCatalog.Domain.Entities;

public class Product
{
    public Guid Id { get; private set; }
    public string Name { get; private set; } = string.Empty;
    public string Description { get; private set; } = string.Empty;
    public decimal Price { get; private set; }
    public int StockQuantity { get; private set; }
    public string CategoryName { get; private set; } = string.Empty;
    public bool IsDeleted { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? UpdatedAt { get; private set; }

    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    private Product() { }

    public static Product Create(
        string name,
        string description,
        decimal price,
        int stockQuantity,
        string categoryName)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Name cannot be empty", nameof(name));
        if (price <= 0)
            throw new ArgumentException("Price must be greater than 0", nameof(price));
        if (stockQuantity < 0)
            throw new ArgumentException("Stock quantity cannot be negative", nameof(stockQuantity));

        var product = new Product
        {
            Id = Guid.NewGuid(),
            Name = name,
            Description = description,
            Price = price,
            StockQuantity = stockQuantity,
            CategoryName = categoryName,
            IsDeleted = false,
            CreatedAt = DateTime.UtcNow
        };

        // เพิ่ม Domain Event เพื่อ notify ส่วนอื่นๆ
        product._domainEvents.Add(new ProductCreatedEvent(
            product.Id, product.Name, product.Price));

        return product;
    }

    public void UpdateDetails(string name, string description, decimal price)
    {
        if (IsDeleted) throw new InvalidOperationException("Cannot update deleted product.");
        if (price <= 0) throw new ArgumentException("Price must be greater than 0", nameof(price));

        Name = name;
        Description = description;
        Price = price;
        UpdatedAt = DateTime.UtcNow;

        _domainEvents.Add(new ProductUpdatedEvent(Id, Name, Price));
    }

    public void SetStockQuantity(int quantity)
    {
        if (quantity < 0)
            throw new ArgumentException("Stock quantity cannot be negative", nameof(quantity));

        StockQuantity = quantity;
        UpdatedAt = DateTime.UtcNow;
    }

    public void MarkAsDeleted()
    {
        if (IsDeleted) throw new InvalidOperationException("Product is already deleted.");
        IsDeleted = true;
        UpdatedAt = DateTime.UtcNow;

        _domainEvents.Add(new ProductDeletedEvent(Id));
    }

    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

### Domain Events

```csharp
// Domain/Events/ProductCreatedEvent.cs
public interface IDomainEvent : INotification { }

public record ProductCreatedEvent(
    Guid ProductId,
    string Name,
    decimal Price
) : IDomainEvent;

public record ProductUpdatedEvent(
    Guid ProductId,
    string NewName,
    decimal NewPrice
) : IDomainEvent;

public record ProductDeletedEvent(Guid ProductId) : IDomainEvent;
```

### Cache Invalidation บน Commands

```csharp
// Application/Products/Commands/UpdateProduct/UpdateProductCommandHandler.cs
public class UpdateProductCommandHandler : IRequestHandler<UpdateProductCommand, Result>
{
    private readonly IProductRepository _repository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly IMemoryCache _cache;

    public UpdateProductCommandHandler(
        IProductRepository repository,
        IUnitOfWork unitOfWork,
        IMemoryCache cache)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
        _cache = cache;
    }

    public async Task<Result> Handle(
        UpdateProductCommand request,
        CancellationToken cancellationToken)
    {
        var product = await _repository.GetByIdAsync(request.Id, cancellationToken);
        if (product is null)
            return Result.Failure($"Product with Id '{request.Id}' not found.");

        product.UpdateDetails(request.Name, request.Description, request.Price);
        product.SetStockQuantity(request.StockQuantity);

        _repository.Update(product);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        // Invalidate Cache เมื่อ Update สำเร็จ
        _cache.Remove($"product:{request.Id}");

        // Invalidate List Caches ด้วย (เพราะ List อาจมีข้อมูลเก่า)
        _cache.Remove("product:list:all");

        return Result.Success();
    }
}
```

### Infrastructure Layer

```csharp
// Infrastructure/Persistence/ApplicationDbContext.cs
namespace ProductCatalog.Infrastructure.Persistence;

public class ApplicationDbContext : DbContext, IApplicationDbContext
{
    private readonly IMediator _mediator;

    public ApplicationDbContext(
        DbContextOptions<ApplicationDbContext> options,
        IMediator mediator) : base(options)
    {
        _mediator = mediator;
    }

    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Name).IsRequired().HasMaxLength(200);
            entity.Property(e => e.Price).HasColumnType("decimal(18,2)");
            entity.HasIndex(e => e.Name);
            entity.HasQueryFilter(e => !e.IsDeleted); // Global Filter: ซ่อน deleted items
        });
    }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // Dispatch Domain Events ก่อน Save
        var entitiesWithEvents = ChangeTracker
            .Entries<Product>()
            .Select(e => e.Entity)
            .Where(e => e.DomainEvents.Any())
            .ToList();

        var result = await base.SaveChangesAsync(cancellationToken);

        // Dispatch หลัง Save (Synchronous Projection)
        foreach (var entity in entitiesWithEvents)
        {
            foreach (var domainEvent in entity.DomainEvents)
                await _mediator.Publish(domainEvent, cancellationToken);

            entity.ClearDomainEvents();
        }

        return result;
    }
}
```

### Repository Implementation

```csharp
// Infrastructure/Persistence/ProductRepository.cs
namespace ProductCatalog.Infrastructure.Persistence;

public class ProductRepository : IProductRepository
{
    private readonly ApplicationDbContext _context;

    public ProductRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<Product?> GetByIdAsync(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        // Tracking ON สำหรับ Write operations
        return await _context.Products
            .FirstOrDefaultAsync(p => p.Id == id, cancellationToken);
    }

    public async Task<Product?> FindByNameAsync(
        string name,
        CancellationToken cancellationToken = default)
    {
        return await _context.Products
            .AsNoTracking()
            .FirstOrDefaultAsync(p => p.Name == name, cancellationToken);
    }

    public async Task AddAsync(
        Product product,
        CancellationToken cancellationToken = default)
    {
        await _context.Products.AddAsync(product, cancellationToken);
    }

    public void Update(Product product)
    {
        _context.Products.Update(product);
    }
}
```

### API Controller

```csharp
// Api/Controllers/ProductsController.cs
namespace ProductCatalog.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;

    public ProductsController(IMediator mediator)
    {
        _mediator = mediator;
    }

    // GET api/products/{id}
    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetById(Guid id, CancellationToken ct)
    {
        var result = await _mediator.Send(new GetProductByIdQuery(id), ct);
        return result.IsSuccess
            ? Ok(result.Value)
            : NotFound(result.Error);
    }

    // GET api/products?searchTerm=laptop&pageNumber=1&pageSize=20
    [HttpGet]
    public async Task<IActionResult> GetList(
        [FromQuery] GetProductListQuery query,
        CancellationToken ct)
    {
        var result = await _mediator.Send(query, ct);
        return result.IsSuccess
            ? Ok(result.Value)
            : BadRequest(result.Errors);
    }

    // POST api/products
    [HttpPost]
    public async Task<IActionResult> Create(
        [FromBody] CreateProductCommand command,
        CancellationToken ct)
    {
        var result = await _mediator.Send(command, ct);
        if (!result.IsSuccess)
            return BadRequest(result.Errors);

        return CreatedAtAction(
            nameof(GetById),
            new { id = result.Value },
            new { id = result.Value });
    }

    // PUT api/products/{id}
    [HttpPut("{id:guid}")]
    public async Task<IActionResult> Update(
        Guid id,
        [FromBody] UpdateProductRequest request,
        CancellationToken ct)
    {
        var command = new UpdateProductCommand(
            id,
            request.Name,
            request.Description,
            request.Price,
            request.StockQuantity);

        var result = await _mediator.Send(command, ct);
        return result.IsSuccess
            ? NoContent()
            : result.Error?.Contains("not found") == true
                ? NotFound(result.Error)
                : BadRequest(result.Error);
    }

    // DELETE api/products/{id}
    [HttpDelete("{id:guid}")]
    public async Task<IActionResult> Delete(Guid id, CancellationToken ct)
    {
        var result = await _mediator.Send(new DeleteProductCommand(id), ct);
        return result.IsSuccess
            ? NoContent()
            : NotFound(result.Error);
    }
}

// Request DTO สำหรับ PUT endpoint
public record UpdateProductRequest(
    string Name,
    string Description,
    decimal Price,
    int StockQuantity
);
```

### DependencyInjection Setup

```csharp
// Application/DependencyInjection.cs
namespace ProductCatalog.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        var assembly = Assembly.GetExecutingAssembly();

        // ลงทะเบียน MediatR พร้อม Handlers ทั้งหมดใน Assembly
        services.AddMediatR(config =>
        {
            config.RegisterServicesFromAssembly(assembly);

            // เพิ่ม Pipeline Behaviors ตามลำดับ (จากนอกเข้าใน)
            config.AddBehavior(typeof(IPipelineBehavior<,>),
                typeof(LoggingBehavior<,>));
            config.AddBehavior(typeof(IPipelineBehavior<,>),
                typeof(ValidationBehavior<,>));
            config.AddBehavior(typeof(IPipelineBehavior<,>),
                typeof(CachingBehavior<,>));
            config.AddBehavior(typeof(IPipelineBehavior<,>),
                typeof(TransactionBehavior<,>));
        });

        // ลงทะเบียน FluentValidation Validators ทั้งหมด
        services.AddValidatorsFromAssembly(assembly);

        // Memory Cache สำหรับ CachingBehavior
        services.AddMemoryCache();

        return services;
    }
}

// Infrastructure/DependencyInjection.cs
namespace ProductCatalog.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // EF Core
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection")));

        // Interface registrations
        services.AddScoped<IApplicationDbContext>(
            provider => provider.GetRequiredService<ApplicationDbContext>());

        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IUnitOfWork, UnitOfWork>();

        return services;
    }
}
```

### Program.cs (Complete Setup)

```csharp
// Api/Program.cs
using ProductCatalog.Application;
using ProductCatalog.Infrastructure;

var builder = WebApplication.CreateBuilder(args);

// === Add Services ===

// Application Layer (MediatR + FluentValidation + Behaviors)
builder.Services.AddApplication();

// Infrastructure Layer (EF Core + Repositories)
builder.Services.AddInfrastructure(builder.Configuration);

// API Layer
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Exception Handling Middleware
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

// Logging
builder.Logging.AddConsole();

var app = builder.Build();

// === Configure Pipeline ===

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();

    // Auto-migrate ใน Development
    using var scope = app.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    await db.Database.MigrateAsync();
}

app.UseExceptionHandler();
app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

### Global Exception Handler

```csharp
// Api/Infrastructure/GlobalExceptionHandler.cs
using FluentValidation;
using Microsoft.AspNetCore.Diagnostics;

public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception, "Unhandled exception: {Message}", exception.Message);

        var (statusCode, title, errors) = exception switch
        {
            ValidationException ve => (
                StatusCodes.Status400BadRequest,
                "Validation Error",
                ve.Errors.Select(e => e.ErrorMessage).ToList()
            ),
            NotFoundException nfe => (
                StatusCodes.Status404NotFound,
                "Not Found",
                new List<string> { nfe.Message }
            ),
            _ => (
                StatusCodes.Status500InternalServerError,
                "Server Error",
                new List<string> { "An unexpected error occurred." }
            )
        };

        httpContext.Response.StatusCode = statusCode;
        await httpContext.Response.WriteAsJsonAsync(new
        {
            Title = title,
            Status = statusCode,
            Errors = errors
        }, cancellationToken);

        return true;
    }
}
```

### การใช้งาน (ตัวอย่าง HTTP Requests)

```http
### สร้าง Product
POST https://localhost:5001/api/products
Content-Type: application/json

{
    "name": "MacBook Pro 16",
    "description": "Apple MacBook Pro 16 inch M3 Max",
    "price": 89900.00,
    "stockQuantity": 15,
    "categoryName": "Laptops"
}

### Response: 201 Created
{
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
}

---

### ดึง Product ตาม Id
GET https://localhost:5001/api/products/3fa85f64-5717-4562-b3fc-2c963f66afa6

### Response: 200 OK
{
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "name": "MacBook Pro 16",
    "description": "Apple MacBook Pro 16 inch M3 Max",
    "price": 89900.00,
    "stockQuantity": 15,
    "categoryName": "Laptops",
    "createdAt": "2024-01-15T10:30:00Z",
    "isAvailable": true
}

---

### ค้นหา Products พร้อม Filtering
GET https://localhost:5001/api/products?searchTerm=MacBook&minPrice=50000&pageNumber=1&pageSize=10&sortBy=Price

### Response: 200 OK
{
    "items": [...],
    "totalCount": 3,
    "pageNumber": 1,
    "pageSize": 10,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
}

---

### อัพเดท Product
PUT https://localhost:5001/api/products/3fa85f64-5717-4562-b3fc-2c963f66afa6
Content-Type: application/json

{
    "name": "MacBook Pro 16 (2024)",
    "description": "Updated description",
    "price": 92900.00,
    "stockQuantity": 20
}

### Response: 204 No Content

---

### ลบ Product
DELETE https://localhost:5001/api/products/3fa85f64-5717-4562-b3fc-2c963f66afa6

### Response: 204 No Content
```

---

## สรุป CQRS Pattern

### ตารางเปรียบเทียบ Command vs Query

| ด้าน | Command | Query |
|------|---------|-------|
| วัตถุประสงค์ | เปลี่ยนแปลง State | อ่านข้อมูลเท่านั้น |
| Return Type | Result / Result<Id> | Result<T> / Result<PagedResult<T>> |
| Validation | มี (FluentValidation) | มีน้อย (range check เท่านั้น) |
| Transaction | ใช้ (TransactionBehavior) | ไม่ใช้ |
| Caching | ไม่ (+ invalidate cache) | ใช้ (CachingBehavior) |
| DB Tracking | ON (ต้องการ change tracking) | OFF (AsNoTracking) |
| Model ที่ใช้ | Domain Entity (Business Logic) | DTO/Read Model (Projection) |
| Performance | เน้น Correctness | เน้น Speed |

### ตารางสรุป Pipeline Behaviors

| Behavior | ทำงานกับ | ทำอะไร |
|----------|----------|--------|
| LoggingBehavior | ทุก Request | บันทึก timing และ errors |
| ValidationBehavior | ทุก Request ที่มี Validator | ตรวจสอบ input |
| CachingBehavior | Queries ที่ implement ICacheable | Cache results |
| TransactionBehavior | Commands ที่ implement ITransactional | Wrap ใน DB transaction |

### เมื่อไหรควรใช้ CQRS?

```
✅ ควรใช้ CQRS เมื่อ:
- Read operations ซับซ้อนหรือหลากหลาย (Dashboard, Reports)
- Read/Write มี Scalability ต่างกันมาก
- ต้องการ Cache ฝั่ง Read โดยไม่กระทบ Write
- Domain Model ซับซ้อน (DDD Aggregate)
- มี Multiple Read Views ของข้อมูลชุดเดียวกัน

❌ ไม่ควรใช้ CQRS เมื่อ:
- Application เล็ก, Logic ง่าย (CRUD อย่างเดียว)
- ทีมเล็ก ยังไม่คุ้นกับ Pattern นี้
- Read/Write มีความซับซ้อนเท่ากัน
- ไม่มีความต้องการ Scale แยกกัน
```

---

## Navigation

| | |
|---|---|
| ◀ Previous | [Part 61: DDD Introduction](part61-ddd-intro.md) |
| ▶ Next | [Part 63: Event Sourcing](part63-event-sourcing.md) |

---

*Part 62: CQRS — ขั้นตอนที่ 611-620 | C# WPE Course*
