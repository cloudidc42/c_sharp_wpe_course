# Part 85: Capstone - Advanced Search (ระบบค้นหาขั้นสูง)

## ShopThai Project: Steps 841-850

> ในส่วนนี้เราจะพัฒนาระบบค้นหาขั้นสูงสำหรับโปรเจค ShopThai ครอบคลุมตั้งแต่ Full-text search ด้วย PostgreSQL ไปจนถึงการ integrate กับ Elasticsearch พร้อมรองรับภาษาไทย การค้นหาตามพิกัดสถานที่ และการวิเคราะห์พฤติกรรมการค้นหา

---

## Step 841: Full-Text Search ด้วย PostgreSQL (tsvector, GIN Index)

### แนวคิด PostgreSQL Full-Text Search

PostgreSQL มีความสามารถในการทำ Full-text search โดยใช้ `tsvector` เพื่อเก็บข้อความที่ผ่านการประมวลผลแล้ว และ `tsquery` สำหรับการค้นหา ข้อดีคือไม่ต้องพึ่งระบบภายนอก แต่สำหรับ production scale อาจต้องใช้ Elasticsearch เพิ่มเติม

### สร้าง Migration สำหรับ Full-Text Search

```csharp
// Migrations/AddFullTextSearchToProducts.cs
using Microsoft.EntityFrameworkCore.Migrations;
using Npgsql.EntityFrameworkCore.PostgreSQL.Metadata;

public partial class AddFullTextSearchToProducts : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // เพิ่มคอลัมน์ tsvector สำหรับเก็บ search vector
        migrationBuilder.AddColumn<string>(
            name: "SearchVector",
            table: "Products",
            type: "tsvector",
            nullable: true);

        // สร้าง GIN index สำหรับเพิ่มความเร็วในการค้นหา
        migrationBuilder.Sql(
            @"CREATE INDEX IX_Products_SearchVector 
              ON ""Products"" 
              USING GIN (""SearchVector"")");

        // สร้าง trigger เพื่ออัปเดต search vector อัตโนมัติ
        migrationBuilder.Sql(
            @"CREATE OR REPLACE FUNCTION update_product_search_vector()
              RETURNS TRIGGER AS $$
              BEGIN
                NEW.""SearchVector"" :=
                  setweight(to_tsvector('english', coalesce(NEW.""Name"", '')), 'A') ||
                  setweight(to_tsvector('english', coalesce(NEW.""Description"", '')), 'B') ||
                  setweight(to_tsvector('english', coalesce(NEW.""Category"", '')), 'C') ||
                  setweight(to_tsvector('simple', coalesce(NEW.""Tags"", '')), 'D');
                RETURN NEW;
              END;
              $$ LANGUAGE plpgsql;");

        migrationBuilder.Sql(
            @"CREATE TRIGGER products_search_vector_update
              BEFORE INSERT OR UPDATE ON ""Products""
              FOR EACH ROW EXECUTE FUNCTION update_product_search_vector();");

        // อัปเดต search vector สำหรับข้อมูลที่มีอยู่แล้ว
        migrationBuilder.Sql(
            @"UPDATE ""Products"" SET ""SearchVector"" =
                setweight(to_tsvector('english', coalesce(""Name"", '')), 'A') ||
                setweight(to_tsvector('english', coalesce(""Description"", '')), 'B') ||
                setweight(to_tsvector('english', coalesce(""Category"", '')), 'C') ||
                setweight(to_tsvector('simple', coalesce(""Tags"", '')), 'D');");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("DROP TRIGGER IF EXISTS products_search_vector_update ON \"Products\";");
        migrationBuilder.Sql("DROP FUNCTION IF EXISTS update_product_search_vector();");
        migrationBuilder.Sql("DROP INDEX IF EXISTS IX_Products_SearchVector;");
        migrationBuilder.DropColumn(name: "SearchVector", table: "Products");
    }
}
```

### Product Entity และ DbContext Configuration

```csharp
// Models/Product.cs
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
using NpgsqlTypes;

public class Product
{
    public int Id { get; set; }

    [Required, MaxLength(200)]
    public string Name { get; set; } = string.Empty;

    public string Description { get; set; } = string.Empty;

    [Column(TypeName = "decimal(18,2)")]
    public decimal Price { get; set; }

    public int StockQuantity { get; set; }

    [MaxLength(100)]
    public string Category { get; set; } = string.Empty;

    public string Tags { get; set; } = string.Empty;

    public double AverageRating { get; set; }
    public int ReviewCount { get; set; }

    // พิกัดสถานที่ร้านค้า
    public double? Latitude { get; set; }
    public double? Longitude { get; set; }

    // คอลัมน์ tsvector สำหรับ Full-text search
    [Column(TypeName = "tsvector")]
    public NpgsqlTsVector? SearchVector { get; set; }

    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;

    // Relationships
    public int StoreId { get; set; }
    public Store Store { get; set; } = null!;
    public ICollection<Review> Reviews { get; set; } = new List<Review>();
    public ICollection<ProductImage> Images { get; set; } = new List<ProductImage>();
}
```

```csharp
// Data/ApplicationDbContext.cs - การ configure Full-text search
public class ApplicationDbContext : DbContext
{
    public DbSet<Product> Products { get; set; }
    public DbSet<Store> Stores { get; set; }
    public DbSet<Review> Reviews { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>(entity =>
        {
            // Configure tsvector column
            entity.Property(p => p.SearchVector)
                .HasColumnType("tsvector")
                .IsRequired(false);

            // Computed column สำหรับ search
            entity.HasGeneratedTsVectorColumn(
                p => p.SearchVector,
                "english",
                p => new { p.Name, p.Description, p.Category });

            // GIN Index
            entity.HasIndex(p => p.SearchVector)
                .HasMethod("GIN");

            // B-Tree indexes สำหรับ filtering
            entity.HasIndex(p => p.Category);
            entity.HasIndex(p => p.Price);
            entity.HasIndex(p => p.AverageRating);
        });
    }
}
```

### PostgreSQL Search Service

```csharp
// Services/PostgresSearchService.cs
using Microsoft.EntityFrameworkCore;
using NpgsqlTypes;

public class PostgresSearchService
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<PostgresSearchService> _logger;

    public PostgresSearchService(
        ApplicationDbContext context,
        ILogger<PostgresSearchService> logger)
    {
        _context = context;
        _logger = logger;
    }

    public async Task<SearchResult<Product>> SearchProductsAsync(
        string query,
        int page = 1,
        int pageSize = 20)
    {
        _logger.LogInformation("Searching PostgreSQL for: {Query}", query);

        // แปลง query เป็น tsquery format
        var tsQuery = query.Trim()
            .Split(' ', StringSplitOptions.RemoveEmptyEntries)
            .Select(w => w + ":*")  // prefix search
            .Aggregate((a, b) => $"{a} & {b}");

        var searchQuery = _context.Products
            .Where(p => p.SearchVector!.Matches(
                EF.Functions.ToTsQuery("english", tsQuery)))
            .OrderByDescending(p =>
                p.SearchVector!.RankCoverDensity(
                    EF.Functions.ToTsQuery("english", tsQuery)));

        var totalCount = await searchQuery.CountAsync();

        var products = await searchQuery
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .Include(p => p.Images)
            .Include(p => p.Store)
            .ToListAsync();

        return new SearchResult<Product>
        {
            Items = products,
            TotalCount = totalCount,
            Page = page,
            PageSize = pageSize,
            TotalPages = (int)Math.Ceiling((double)totalCount / pageSize)
        };
    }

    // ค้นหาด้วย headline (highlight ข้อความที่ตรงกัน)
    public async Task<List<ProductSearchDto>> SearchWithHighlightAsync(string query)
    {
        var tsQuery = query.Trim()
            .Split(' ', StringSplitOptions.RemoveEmptyEntries)
            .Select(w => w + ":*")
            .Aggregate((a, b) => $"{a} & {b}");

        return await _context.Products
            .Where(p => p.SearchVector!.Matches(
                EF.Functions.ToTsQuery("english", tsQuery)))
            .Select(p => new ProductSearchDto
            {
                Id = p.Id,
                Name = p.Name,
                Price = p.Price,
                // สร้าง highlight จากชื่อสินค้า
                Headline = EF.Functions.TsHeadline(
                    "english",
                    p.Name + " " + p.Description,
                    EF.Functions.ToTsQuery("english", tsQuery),
                    "StartSel=<mark>, StopSel=</mark>, MaxWords=50, MinWords=10")
            })
            .Take(50)
            .ToListAsync();
    }
}
```

---

## Step 842: Elasticsearch Integration ด้วย NEST/Elastic.Clients.Elasticsearch

### ติดตั้ง NuGet Packages

```xml
<!-- ShopThai.API.csproj -->
<PackageReference Include="Elastic.Clients.Elasticsearch" Version="8.11.0" />
<PackageReference Include="Elastic.Transport" Version="0.4.16" />
```

### ElasticSearch Configuration

```csharp
// Configuration/ElasticsearchSettings.cs
public class ElasticsearchSettings
{
    public string Uri { get; set; } = "http://localhost:9200";
    public string Username { get; set; } = "elastic";
    public string Password { get; set; } = string.Empty;
    public string ProductsIndex { get; set; } = "shopthai-products";
    public string StoresIndex { get; set; } = "shopthai-stores";
    public bool EnableDebugMode { get; set; } = false;
    public int BulkBatchSize { get; set; } = 500;
}
```

```csharp
// Extensions/ElasticsearchExtensions.cs
using Elastic.Clients.Elasticsearch;
using Elastic.Transport;

public static class ElasticsearchExtensions
{
    public static IServiceCollection AddElasticsearch(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        var settings = configuration
            .GetSection("Elasticsearch")
            .Get<ElasticsearchSettings>()!;

        services.Configure<ElasticsearchSettings>(
            configuration.GetSection("Elasticsearch"));

        var elasticSettings = new ElasticsearchClientSettings(new Uri(settings.Uri))
            .Authentication(new BasicAuthentication(settings.Username, settings.Password))
            .DefaultIndex(settings.ProductsIndex)
            .EnableDebugMode(settings.EnableDebugMode
                ? d => Console.WriteLine(d.DebugInformation)
                : null)
            .RequestTimeout(TimeSpan.FromSeconds(30))
            .MaximumRetries(3)
            .MaxRetryTimeout(TimeSpan.FromSeconds(60));

        var client = new ElasticsearchClient(elasticSettings);
        services.AddSingleton(client);
        services.AddScoped<IElasticsearchService, ElasticsearchService>();

        return services;
    }
}
```

### Product Document Model

```csharp
// Models/Elasticsearch/ProductDocument.cs
using Elastic.Clients.Elasticsearch.Mapping;

public class ProductDocument
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string NameThai { get; set; } = string.Empty;  // ชื่อภาษาไทย
    public string Description { get; set; } = string.Empty;
    public string DescriptionThai { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public decimal? OriginalPrice { get; set; }
    public int StockQuantity { get; set; }
    public string Category { get; set; } = string.Empty;
    public string SubCategory { get; set; } = string.Empty;
    public List<string> Tags { get; set; } = new();
    public string Brand { get; set; } = string.Empty;
    public double AverageRating { get; set; }
    public int ReviewCount { get; set; }
    public int SalesCount { get; set; }
    public int ViewCount { get; set; }
    public bool IsActive { get; set; } = true;
    public bool IsFeatured { get; set; }
    public bool IsOnSale { get; set; }
    public string ImageUrl { get; set; } = string.Empty;
    public List<string> ImageUrls { get; set; } = new();

    // ข้อมูลร้านค้า
    public StoreInfo Store { get; set; } = null!;

    // พิกัด (สำหรับ geo-search)
    public GeoPoint? Location { get; set; }

    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}

public class StoreInfo
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
    public double Rating { get; set; }
}

public class GeoPoint
{
    public double Lat { get; set; }
    public double Lon { get; set; }
}
```

### สร้าง Index Mapping

```csharp
// Services/ElasticsearchIndexService.cs
using Elastic.Clients.Elasticsearch;
using Elastic.Clients.Elasticsearch.IndexManagement;
using Elastic.Clients.Elasticsearch.Mapping;
using Elastic.Clients.Elasticsearch.Analysis;

public class ElasticsearchIndexService
{
    private readonly ElasticsearchClient _client;
    private readonly ElasticsearchSettings _settings;
    private readonly ILogger<ElasticsearchIndexService> _logger;

    public ElasticsearchIndexService(
        ElasticsearchClient client,
        IOptions<ElasticsearchSettings> settings,
        ILogger<ElasticsearchIndexService> logger)
    {
        _client = client;
        _settings = settings.Value;
        _logger = logger;
    }

    public async Task CreateProductsIndexAsync()
    {
        var indexName = _settings.ProductsIndex;

        // ตรวจสอบว่า index มีอยู่แล้วหรือยัง
        var existsResponse = await _client.Indices.ExistsAsync(indexName);
        if (existsResponse.Exists)
        {
            _logger.LogInformation("Index {IndexName} already exists", indexName);
            return;
        }

        var createResponse = await _client.Indices.CreateAsync(indexName, c => c
            .Settings(s => s
                .NumberOfShards(3)
                .NumberOfReplicas(1)
                .Analysis(a => a
                    // Thai analyzer configuration
                    .Analyzers(an => an
                        .Custom("thai_analyzer", ca => ca
                            .Tokenizer("thai")
                            .Filter(new[]
                            {
                                "lowercase",
                                "thai_stop",
                                "thai_stemmer"
                            }))
                        .Custom("thai_search_analyzer", ca => ca
                            .Tokenizer("thai")
                            .Filter(new[]
                            {
                                "lowercase",
                                "thai_stop"
                            }))
                        .Custom("autocomplete_analyzer", ca => ca
                            .Tokenizer("autocomplete_tokenizer")
                            .Filter(new[] { "lowercase" }))
                        .Custom("autocomplete_search_analyzer", ca => ca
                            .Tokenizer("standard")
                            .Filter(new[] { "lowercase" })))
                    .Tokenizers(t => t
                        .EdgeNGram("autocomplete_tokenizer", eng => eng
                            .MinGram(2)
                            .MaxGram(20)
                            .TokenChars(new[]
                            {
                                TokenChar.Letter,
                                TokenChar.Digit
                            })))
                    .TokenFilters(tf => tf
                        .Stop("thai_stop", s => s
                            .StopWords("_thai_"))
                        .Stemmer("thai_stemmer", s => s
                            .Language("thai")))))
            .Mappings(m => m
                .Properties<ProductDocument>(p => p
                    .IntegerNumber(f => f.Id)
                    .Text(f => f.Name, t => t
                        .Analyzer("standard")
                        .SearchAnalyzer("standard")
                        .Fields(ff => ff
                            .Keyword("keyword", k => k.IgnoreAbove(256))
                            .Text("autocomplete", ac => ac
                                .Analyzer("autocomplete_analyzer")
                                .SearchAnalyzer("autocomplete_search_analyzer"))))
                    .Text(f => f.NameThai, t => t
                        .Analyzer("thai_analyzer")
                        .SearchAnalyzer("thai_search_analyzer"))
                    .Text(f => f.Description, t => t.Analyzer("standard"))
                    .Text(f => f.DescriptionThai, t => t.Analyzer("thai_analyzer"))
                    .FloatNumber(f => f.Price)
                    .FloatNumber(f => f.OriginalPrice)
                    .IntegerNumber(f => f.StockQuantity)
                    .Keyword(f => f.Category)
                    .Keyword(f => f.SubCategory)
                    .Keyword(f => f.Tags)
                    .Keyword(f => f.Brand)
                    .FloatNumber(f => f.AverageRating)
                    .IntegerNumber(f => f.ReviewCount)
                    .IntegerNumber(f => f.SalesCount)
                    .IntegerNumber(f => f.ViewCount)
                    .Boolean(f => f.IsActive)
                    .Boolean(f => f.IsFeatured)
                    .Boolean(f => f.IsOnSale)
                    .Keyword(f => f.ImageUrl)
                    .Object(f => f.Store, o => o
                        .Properties(sp => sp
                            .IntegerNumber(sf => sf.Id)
                            .Keyword(sf => sf.Name)
                            .Keyword(sf => sf.City)
                            .FloatNumber(sf => sf.Rating)))
                    .GeoPoint(f => f.Location)
                    .Date(f => f.CreatedAt)
                    .Date(f => f.UpdatedAt))));

        if (!createResponse.IsValidResponse)
        {
            throw new Exception(
                $"Failed to create index: {createResponse.ElasticsearchServerError?.Error?.Reason}");
        }

        _logger.LogInformation("Created index: {IndexName}", indexName);
    }
}
```

---

## Step 843: Search Indexing Pipeline (EF Core → Elasticsearch Sync)

### Product Indexer Service

```csharp
// Services/ProductIndexerService.cs
using Elastic.Clients.Elasticsearch;
using Elastic.Clients.Elasticsearch.Core.Bulk;

public interface IProductIndexerService
{
    Task IndexProductAsync(int productId);
    Task IndexProductsAsync(IEnumerable<int> productIds);
    Task BulkIndexAllProductsAsync(IProgress<int>? progress = null);
    Task DeleteProductFromIndexAsync(int productId);
    Task UpdateProductInIndexAsync(int productId);
}

public class ProductIndexerService : IProductIndexerService
{
    private readonly ApplicationDbContext _context;
    private readonly ElasticsearchClient _client;
    private readonly ElasticsearchSettings _settings;
    private readonly ILogger<ProductIndexerService> _logger;

    public ProductIndexerService(
        ApplicationDbContext context,
        ElasticsearchClient client,
        IOptions<ElasticsearchSettings> settings,
        ILogger<ProductIndexerService> logger)
    {
        _context = context;
        _client = client;
        _settings = settings.Value;
        _logger = logger;
    }

    public async Task IndexProductAsync(int productId)
    {
        var product = await _context.Products
            .Include(p => p.Store)
            .Include(p => p.Images)
            .Include(p => p.Reviews)
            .FirstOrDefaultAsync(p => p.Id == productId);

        if (product == null)
        {
            _logger.LogWarning("Product {ProductId} not found for indexing", productId);
            return;
        }

        var document = MapToDocument(product);
        var indexResponse = await _client.IndexAsync(
            document,
            i => i.Index(_settings.ProductsIndex).Id(document.Id.ToString()));

        if (!indexResponse.IsValidResponse)
        {
            _logger.LogError(
                "Failed to index product {ProductId}: {Error}",
                productId,
                indexResponse.ElasticsearchServerError?.Error?.Reason);
            throw new Exception($"Elasticsearch indexing failed for product {productId}");
        }

        _logger.LogDebug("Indexed product {ProductId}", productId);
    }

    public async Task BulkIndexAllProductsAsync(IProgress<int>? progress = null)
    {
        var batchSize = _settings.BulkBatchSize;
        var totalCount = await _context.Products.CountAsync();
        var processedCount = 0;

        _logger.LogInformation("Starting bulk index of {Total} products", totalCount);

        for (int offset = 0; offset < totalCount; offset += batchSize)
        {
            var products = await _context.Products
                .Include(p => p.Store)
                .Include(p => p.Images)
                .OrderBy(p => p.Id)
                .Skip(offset)
                .Take(batchSize)
                .ToListAsync();

            var documents = products.Select(MapToDocument).ToList();

            var bulkResponse = await _client.BulkAsync(b => b
                .Index(_settings.ProductsIndex)
                .IndexMany(documents, (bd, doc) => bd.Id(doc.Id.ToString())));

            if (bulkResponse.Errors)
            {
                var errors = bulkResponse.ItemsWithErrors;
                foreach (var error in errors)
                {
                    _logger.LogError(
                        "Bulk index error for item {Id}: {Error}",
                        error.Id,
                        error.Error?.Reason);
                }
            }

            processedCount += products.Count;
            progress?.Report(processedCount);

            _logger.LogInformation(
                "Indexed {Processed}/{Total} products",
                processedCount,
                totalCount);
        }

        _logger.LogInformation("Bulk indexing completed");
    }

    public async Task DeleteProductFromIndexAsync(int productId)
    {
        var response = await _client.DeleteAsync(
            _settings.ProductsIndex,
            productId.ToString());

        if (!response.IsValidResponse && response.Result != Result.NotFound)
        {
            _logger.LogError(
                "Failed to delete product {ProductId} from index",
                productId);
        }
    }

    public async Task UpdateProductInIndexAsync(int productId)
    {
        // ทำ re-index แทน update เพื่อความง่าย
        await IndexProductAsync(productId);
    }

    private ProductDocument MapToDocument(Product product)
    {
        return new ProductDocument
        {
            Id = product.Id,
            Name = product.Name,
            NameThai = product.NameThai ?? string.Empty,
            Description = product.Description,
            DescriptionThai = product.DescriptionThai ?? string.Empty,
            Price = product.Price,
            OriginalPrice = product.OriginalPrice,
            StockQuantity = product.StockQuantity,
            Category = product.Category,
            SubCategory = product.SubCategory ?? string.Empty,
            Tags = product.Tags.Split(',', StringSplitOptions.RemoveEmptyEntries).ToList(),
            Brand = product.Brand ?? string.Empty,
            AverageRating = product.AverageRating,
            ReviewCount = product.ReviewCount,
            SalesCount = product.SalesCount,
            ViewCount = product.ViewCount,
            IsActive = product.IsActive,
            IsFeatured = product.IsFeatured,
            IsOnSale = product.IsOnSale,
            ImageUrl = product.Images.FirstOrDefault()?.Url ?? string.Empty,
            ImageUrls = product.Images.Select(i => i.Url).ToList(),
            Store = new StoreInfo
            {
                Id = product.Store.Id,
                Name = product.Store.Name,
                City = product.Store.City,
                Rating = product.Store.Rating
            },
            Location = (product.Latitude.HasValue && product.Longitude.HasValue)
                ? new GeoPoint
                {
                    Lat = product.Latitude.Value,
                    Lon = product.Longitude.Value
                }
                : null,
            CreatedAt = product.CreatedAt,
            UpdatedAt = product.UpdatedAt
        };
    }
}
```

### Background Sync Worker

```csharp
// Workers/SearchSyncWorker.cs
using Microsoft.Extensions.Hosting;

public class SearchSyncWorker : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<SearchSyncWorker> _logger;
    private readonly Channel<SearchSyncMessage> _channel;

    public SearchSyncWorker(
        IServiceProvider serviceProvider,
        ILogger<SearchSyncWorker> logger,
        Channel<SearchSyncMessage> channel)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
        _channel = channel;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Search sync worker started");

        await foreach (var message in _channel.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                using var scope = _serviceProvider.CreateScope();
                var indexer = scope.ServiceProvider
                    .GetRequiredService<IProductIndexerService>();

                switch (message.Operation)
                {
                    case SyncOperation.Index:
                        await indexer.IndexProductAsync(message.ProductId);
                        break;
                    case SyncOperation.Delete:
                        await indexer.DeleteProductFromIndexAsync(message.ProductId);
                        break;
                    case SyncOperation.Update:
                        await indexer.UpdateProductInIndexAsync(message.ProductId);
                        break;
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex,
                    "Error processing search sync for product {ProductId}",
                    message.ProductId);
            }
        }
    }
}

public record SearchSyncMessage(int ProductId, SyncOperation Operation);
public enum SyncOperation { Index, Update, Delete }
```

---

## Step 844: Faceted Search (กรองตาม Category, Price Range, Rating, Tags)

### Search Request Model

```csharp
// Models/Search/SearchRequest.cs
public class ProductSearchRequest
{
    public string Query { get; set; } = string.Empty;
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 20;

    // Filters
    public List<string> Categories { get; set; } = new();
    public List<string> SubCategories { get; set; } = new();
    public List<string> Brands { get; set; } = new();
    public List<string> Tags { get; set; } = new();
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
    public double? MinRating { get; set; }
    public bool? IsOnSale { get; set; }
    public bool? InStock { get; set; }

    // Geo search
    public double? Latitude { get; set; }
    public double? Longitude { get; set; }
    public double? RadiusKm { get; set; }

    // Sorting
    public SearchSortBy SortBy { get; set; } = SearchSortBy.Relevance;
    public bool SortDescending { get; set; } = true;
}

public enum SearchSortBy
{
    Relevance,
    Price,
    Rating,
    NewestFirst,
    Popularity,
    Distance
}
```

### Faceted Search Service

```csharp
// Services/FacetedSearchService.cs
using Elastic.Clients.Elasticsearch;
using Elastic.Clients.Elasticsearch.QueryDsl;

public class FacetedSearchService
{
    private readonly ElasticsearchClient _client;
    private readonly ElasticsearchSettings _settings;

    public FacetedSearchService(
        ElasticsearchClient client,
        IOptions<ElasticsearchSettings> settings)
    {
        _client = client;
        _settings = settings.Value;
    }

    public async Task<FacetedSearchResult> SearchWithFacetsAsync(
        ProductSearchRequest request)
    {
        var searchRequest = new SearchRequest(_settings.ProductsIndex)
        {
            From = (request.Page - 1) * request.PageSize,
            Size = request.PageSize,
            Query = BuildQuery(request),
            Aggregations = BuildAggregations(request),
            Sort = BuildSort(request),
            // คืน source fields ที่จำเป็น
            Source = new SourceConfig(new SourceFilter
            {
                Includes = new[]
                {
                    "id", "name", "nameThai", "price", "originalPrice",
                    "category", "brand", "averageRating", "reviewCount",
                    "imageUrl", "isOnSale", "isFeatured", "store.name",
                    "store.city", "salesCount"
                }
            })
        };

        var response = await _client.SearchAsync<ProductDocument>(searchRequest);

        if (!response.IsValidResponse)
        {
            throw new Exception(
                $"Search failed: {response.ElasticsearchServerError?.Error?.Reason}");
        }

        return new FacetedSearchResult
        {
            Products = response.Documents.ToList(),
            TotalCount = (int)(response.Total ?? 0),
            Page = request.Page,
            PageSize = request.PageSize,
            TotalPages = (int)Math.Ceiling((double)(response.Total ?? 0) / request.PageSize),
            Facets = ExtractFacets(response)
        };
    }

    private Query BuildQuery(ProductSearchRequest request)
    {
        var mustClauses = new List<Query>();
        var filterClauses = new List<Query>();
        var shouldClauses = new List<Query>();

        // Active products เท่านั้น
        filterClauses.Add(new TermQuery("isActive") { Value = true });

        // Full-text search query
        if (!string.IsNullOrWhiteSpace(request.Query))
        {
            // Multi-match กับ boosting
            shouldClauses.Add(new MultiMatchQuery
            {
                Query = request.Query,
                Fields = new[]
                {
                    "name^5",        // ชื่อสินค้ามีน้ำหนักสูงสุด
                    "nameThai^5",
                    "brand^3",
                    "category^2",
                    "description^1",
                    "descriptionThai^1",
                    "tags^2"
                },
                Type = TextQueryType.BestFields,
                Fuzziness = new Fuzziness("AUTO"),
                MinimumShouldMatch = "75%"
            });

            // Exact phrase match bonus
            shouldClauses.Add(new MatchPhraseQuery("name")
            {
                Query = request.Query,
                Boost = 10
            });

            mustClauses.Add(new BoolQuery { Should = shouldClauses, MinimumShouldMatch = 1 });
        }

        // Category filter
        if (request.Categories.Any())
        {
            filterClauses.Add(new TermsQuery
            {
                Field = "category",
                Terms = new TermsQueryField(
                    request.Categories.Select(c => FieldValue.String(c)).ToArray())
            });
        }

        // Brand filter
        if (request.Brands.Any())
        {
            filterClauses.Add(new TermsQuery
            {
                Field = "brand",
                Terms = new TermsQueryField(
                    request.Brands.Select(b => FieldValue.String(b)).ToArray())
            });
        }

        // Price range filter
        if (request.MinPrice.HasValue || request.MaxPrice.HasValue)
        {
            filterClauses.Add(new NumberRangeQuery("price")
            {
                Gte = (double?)request.MinPrice,
                Lte = (double?)request.MaxPrice
            });
        }

        // Rating filter
        if (request.MinRating.HasValue)
        {
            filterClauses.Add(new NumberRangeQuery("averageRating")
            {
                Gte = request.MinRating.Value
            });
        }

        // Sale filter
        if (request.IsOnSale.HasValue)
        {
            filterClauses.Add(new TermQuery("isOnSale")
            {
                Value = request.IsOnSale.Value
            });
        }

        // In-stock filter
        if (request.InStock == true)
        {
            filterClauses.Add(new NumberRangeQuery("stockQuantity") { Gt = 0 });
        }

        // Geo distance filter
        if (request.Latitude.HasValue && request.Longitude.HasValue && request.RadiusKm.HasValue)
        {
            filterClauses.Add(new GeoDistanceQuery
            {
                Field = "location",
                Location = new GeoLocation(request.Latitude.Value, request.Longitude.Value),
                Distance = $"{request.RadiusKm.Value}km"
            });
        }

        return new BoolQuery
        {
            Must = mustClauses.Any() ? mustClauses : null,
            Filter = filterClauses.Any() ? filterClauses : null
        };
    }

    private AggregationDictionary BuildAggregations(ProductSearchRequest request)
    {
        return new AggregationDictionary
        {
            // Category facet
            ["categories"] = new TermsAggregation
            {
                Field = "category",
                Size = 20
            },
            // Sub-category facet
            ["subCategories"] = new TermsAggregation
            {
                Field = "subCategory",
                Size = 30
            },
            // Brand facet
            ["brands"] = new TermsAggregation
            {
                Field = "brand",
                Size = 20
            },
            // Tags facet
            ["tags"] = new TermsAggregation
            {
                Field = "tags",
                Size = 30
            },
            // Price ranges (histogram)
            ["priceRanges"] = new RangeAggregation
            {
                Field = "price",
                Ranges = new[]
                {
                    new AggregationRange { Key = "under_500", To = 500 },
                    new AggregationRange { Key = "500_1000", From = 500, To = 1000 },
                    new AggregationRange { Key = "1000_3000", From = 1000, To = 3000 },
                    new AggregationRange { Key = "3000_10000", From = 3000, To = 10000 },
                    new AggregationRange { Key = "over_10000", From = 10000 }
                }
            },
            // Rating facet
            ["ratings"] = new RangeAggregation
            {
                Field = "averageRating",
                Ranges = new[]
                {
                    new AggregationRange { Key = "4_plus", From = 4 },
                    new AggregationRange { Key = "3_plus", From = 3, To = 4 },
                    new AggregationRange { Key = "2_plus", From = 2, To = 3 }
                }
            },
            // Price stats
            ["priceStats"] = new StatsAggregation { Field = "price" },
            // On sale count
            ["onSaleCount"] = new FilterAggregation
            {
                Filter = new TermQuery("isOnSale") { Value = true }
            }
        };
    }

    private SortOptionsCollection BuildSort(ProductSearchRequest request)
    {
        return request.SortBy switch
        {
            SearchSortBy.Price => new SortOptionsCollection
            {
                SortOptions.Field("price",
                    new FieldSort
                    {
                        Order = request.SortDescending ? SortOrder.Desc : SortOrder.Asc
                    })
            },
            SearchSortBy.Rating => new SortOptionsCollection
            {
                SortOptions.Field("averageRating", new FieldSort { Order = SortOrder.Desc }),
                SortOptions.Field("reviewCount", new FieldSort { Order = SortOrder.Desc })
            },
            SearchSortBy.NewestFirst => new SortOptionsCollection
            {
                SortOptions.Field("createdAt", new FieldSort { Order = SortOrder.Desc })
            },
            SearchSortBy.Popularity => new SortOptionsCollection
            {
                SortOptions.Field("salesCount", new FieldSort { Order = SortOrder.Desc }),
                SortOptions.Field("viewCount", new FieldSort { Order = SortOrder.Desc })
            },
            _ => new SortOptionsCollection
            {
                SortOptions.Score()
            }
        };
    }

    private SearchFacets ExtractFacets(SearchResponse<ProductDocument> response)
    {
        var facets = new SearchFacets();

        if (response.Aggregations == null) return facets;

        // Extract category facets
        if (response.Aggregations.TryGetValue("categories", out var categoryAgg) &&
            categoryAgg is StringTermsAggregate categoryTerms)
        {
            facets.Categories = categoryTerms.Buckets
                .Select(b => new FacetItem { Value = b.Key.ToString(), Count = (int)b.DocCount })
                .ToList();
        }

        // Extract brand facets
        if (response.Aggregations.TryGetValue("brands", out var brandAgg) &&
            brandAgg is StringTermsAggregate brandTerms)
        {
            facets.Brands = brandTerms.Buckets
                .Select(b => new FacetItem { Value = b.Key.ToString(), Count = (int)b.DocCount })
                .ToList();
        }

        // Extract price ranges
        if (response.Aggregations.TryGetValue("priceRanges", out var priceAgg) &&
            priceAgg is RangeAggregate priceRanges)
        {
            facets.PriceRanges = priceRanges.Buckets
                .Select(b => new PriceRangeFacet
                {
                    Key = b.Key,
                    Count = (int)b.DocCount,
                    From = b.From,
                    To = b.To
                })
                .ToList();
        }

        return facets;
    }
}
```

---

## Step 845: Autocomplete ด้วย Prefix Search

### Autocomplete Service

```csharp
// Services/AutocompleteService.cs
public class AutocompleteService
{
    private readonly ElasticsearchClient _client;
    private readonly ElasticsearchSettings _settings;
    private readonly IDistributedCache _cache;

    public AutocompleteService(
        ElasticsearchClient client,
        IOptions<ElasticsearchSettings> settings,
        IDistributedCache cache)
    {
        _client = client;
        _settings = settings.Value;
        _cache = cache;
    }

    public async Task<AutocompleteResult> GetSuggestionsAsync(
        string prefix,
        int maxSuggestions = 10)
    {
        if (string.IsNullOrWhiteSpace(prefix) || prefix.Length < 2)
        {
            return new AutocompleteResult();
        }

        // ตรวจสอบ cache ก่อน
        var cacheKey = $"autocomplete:{prefix.ToLowerInvariant()}";
        var cached = await _cache.GetStringAsync(cacheKey);
        if (cached != null)
        {
            return System.Text.Json.JsonSerializer.Deserialize<AutocompleteResult>(cached)!;
        }

        var result = new AutocompleteResult();

        // Completion suggester สำหรับ product names
        var suggestRequest = new SearchRequest(_settings.ProductsIndex)
        {
            Size = 0,
            Suggest = new Dictionary<string, SuggestContainer>
            {
                ["product-suggestions"] = new SuggestContainer
                {
                    Prefix = prefix,
                    Completion = new CompletionSuggester
                    {
                        Field = "name.autocomplete",
                        Size = maxSuggestions,
                        SkipDuplicates = true,
                        FuzzyOptions = new SuggestFuzziness
                        {
                            Fuzziness = new Fuzziness("AUTO"),
                            PrefixLength = 3
                        }
                    }
                }
            }
        };

        var suggestResponse = await _client.SearchAsync<ProductDocument>(suggestRequest);

        // ดึงผลลัพธ์จาก completion suggester
        if (suggestResponse.Suggest?.ContainsKey("product-suggestions") == true)
        {
            var options = suggestResponse.Suggest["product-suggestions"]
                .FirstOrDefault()?.Options;

            result.ProductSuggestions = options?
                .Select(o => new SuggestionItem
                {
                    Text = o.Text,
                    Score = o.Score ?? 0
                })
                .ToList() ?? new List<SuggestionItem>();
        }

        // Prefix search สำหรับ categories
        var categoryResponse = await _client.SearchAsync<ProductDocument>(s => s
            .Index(_settings.ProductsIndex)
            .Size(0)
            .Query(q => q
                .Prefix(p => p
                    .Field("category")
                    .Value(prefix.ToLowerInvariant())))
            .Aggregations(a => a
                .Terms("categories", t => t
                    .Field("category")
                    .Size(5))));

        result.CategorySuggestions = ExtractCategorySuggestions(categoryResponse);

        // Popular searches ที่เริ่มต้นด้วย prefix
        result.PopularSearches = await GetPopularSearchesAsync(prefix, 5);

        // Cache ผลลัพธ์ 5 นาที
        await _cache.SetStringAsync(
            cacheKey,
            System.Text.Json.JsonSerializer.Serialize(result),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
            });

        return result;
    }

    private async Task<List<string>> GetPopularSearchesAsync(string prefix, int count)
    {
        // ดึงจาก Redis sorted set ที่เก็บ popular searches
        // (จะ implement ใน Step 850)
        return await Task.FromResult(new List<string>());
    }

    private List<SuggestionItem> ExtractCategorySuggestions(
        SearchResponse<ProductDocument> response)
    {
        if (response.Aggregations?.TryGetValue("categories", out var agg) == true &&
            agg is StringTermsAggregate terms)
        {
            return terms.Buckets
                .Select(b => new SuggestionItem
                {
                    Text = b.Key.ToString(),
                    Score = b.DocCount
                })
                .ToList();
        }
        return new List<SuggestionItem>();
    }
}

public class AutocompleteResult
{
    public List<SuggestionItem> ProductSuggestions { get; set; } = new();
    public List<SuggestionItem> CategorySuggestions { get; set; } = new();
    public List<string> PopularSearches { get; set; } = new();
}

public class SuggestionItem
{
    public string Text { get; set; } = string.Empty;
    public double Score { get; set; }
}
```

---

## Step 846: Relevance Scoring และ Boosting

### Relevance Scoring Strategy

```csharp
// Services/RelevanceScoringService.cs
public class RelevanceScoringService
{
    // สร้าง Function Score Query สำหรับ custom relevance
    public Query BuildRelevanceQuery(string searchText, UserContext? userContext = null)
    {
        var baseQuery = new MultiMatchQuery
        {
            Query = searchText,
            Fields = new[]
            {
                "name^10",
                "nameThai^10",
                "brand^5",
                "category^3",
                "description^1",
                "descriptionThai^1",
                "tags^2"
            },
            Type = TextQueryType.CrossFields,
            Fuzziness = new Fuzziness("AUTO"),
            PrefixLength = 2
        };

        // Function Score สำหรับ custom boosting
        return new FunctionScoreQuery
        {
            Query = baseQuery,
            Functions = new[]
            {
                // Boost สินค้าขายดี
                new ScoreFunctionDescriptor
                {
                    Filter = new ExistsQuery { Field = "salesCount" },
                    FieldValueFactor = new FieldValueFactorScoreFunction
                    {
                        Field = "salesCount",
                        Factor = 0.1,
                        Modifier = FieldValueFactorModifier.Log1P,
                        Missing = 1
                    }
                },
                // Boost สินค้า rating สูง
                new ScoreFunctionDescriptor
                {
                    Filter = new NumberRangeQuery("averageRating") { Gte = 4.0 },
                    Weight = 2.0
                },
                // Boost สินค้า featured
                new ScoreFunctionDescriptor
                {
                    Filter = new TermQuery("isFeatured") { Value = true },
                    Weight = 3.0
                },
                // Boost สินค้า on sale
                new ScoreFunctionDescriptor
                {
                    Filter = new TermQuery("isOnSale") { Value = true },
                    Weight = 1.5
                },
                // Decay function: สินค้าใหม่ได้คะแนนสูงกว่า
                new ScoreFunctionDescriptor
                {
                    Gauss = new DecayFunctionKeys
                    {
                        Field = "createdAt",
                        GaussPlacement = new DecayPlacement<DateMath, Time>
                        {
                            Origin = DateMath.Now,
                            Scale = new Time("30d"),
                            Offset = new Time("7d"),
                            Decay = 0.5
                        }
                    }
                }
            },
            ScoreMode = FunctionScoreMode.Sum,
            BoostMode = FunctionBoostMode.Multiply,
            MaxBoost = 10.0
        };
    }

    // Personalized boosting ตาม user history
    public Query BuildPersonalizedQuery(
        string searchText,
        List<string> preferredCategories,
        List<string> recentlyViewedBrands)
    {
        var baseQuery = BuildRelevanceQuery(searchText);

        var shouldClauses = new List<Query> { baseQuery };

        // Boost ตาม preferred categories
        foreach (var category in preferredCategories.Take(3))
        {
            shouldClauses.Add(new TermQuery("category")
            {
                Value = category,
                Boost = 2.0f
            });
        }

        // Boost ตาม recently viewed brands
        foreach (var brand in recentlyViewedBrands.Take(3))
        {
            shouldClauses.Add(new TermQuery("brand")
            {
                Value = brand,
                Boost = 1.5f
            });
        }

        return new BoolQuery
        {
            Should = shouldClauses,
            MinimumShouldMatch = 1
        };
    }
}
```

---

## Step 847: Multi-language Thai Text Search

### Thai Language Analyzer Configuration

```csharp
// Services/ThaiSearchService.cs

// appsettings.json configuration
/*
{
  "Elasticsearch": {
    "ThaiAnalyzer": {
      "Tokenizer": "thai",
      "Filter": ["lowercase", "thai_stop"],
      "CharFilter": ["html_strip"]
    }
  }
}
*/

public class ThaiSearchService
{
    private readonly ElasticsearchClient _client;
    private readonly ElasticsearchSettings _settings;

    public ThaiSearchService(
        ElasticsearchClient client,
        IOptions<ElasticsearchSettings> settings)
    {
        _client = client;
        _settings = settings.Value;
    }

    // ค้นหาด้วยภาษาไทย
    public async Task<SearchResponse<ProductDocument>> SearchThaiAsync(
        string thaiQuery,
        int size = 20)
    {
        return await _client.SearchAsync<ProductDocument>(s => s
            .Index(_settings.ProductsIndex)
            .Size(size)
            .Query(q => q
                .Bool(b => b
                    .Should(sh => sh
                        // ค้นหาในชื่อภาษาไทย
                        .Match(m => m
                            .Field("nameThai")
                            .Query(thaiQuery)
                            .Analyzer("thai_search_analyzer")
                            .Boost(5)),
                        sh => sh
                        // ค้นหาในคำอธิบายภาษาไทย
                        .Match(m => m
                            .Field("descriptionThai")
                            .Query(thaiQuery)
                            .Analyzer("thai_search_analyzer")
                            .Boost(1)),
                        sh => sh
                        // ค้นหาใน tags
                        .Match(m => m
                            .Field("tags")
                            .Query(thaiQuery)
                            .Boost(2)))
                    .MinimumShouldMatch(1))));
    }

    // ค้นหาแบบ cross-language (ไทย + อังกฤษ)
    public async Task<SearchResponse<ProductDocument>> SearchCrossLanguageAsync(
        string query,
        int size = 20)
    {
        return await _client.SearchAsync<ProductDocument>(s => s
            .Index(_settings.ProductsIndex)
            .Size(size)
            .Query(q => q
                .Bool(b => b
                    .Should(
                        // English search
                        sh => sh.MultiMatch(m => m
                            .Fields(new[] { "name^5", "description^1", "brand^3", "tags^2" })
                            .Query(query)
                            .Type(TextQueryType.BestFields)
                            .Fuzziness(new Fuzziness("AUTO"))),
                        // Thai search
                        sh => sh.MultiMatch(m => m
                            .Fields(new[] { "nameThai^5", "descriptionThai^1" })
                            .Query(query)
                            .Analyzer("thai_search_analyzer"))
                    )
                    .MinimumShouldMatch(1))));
    }

    // Transliteration helper - แปลงคำอ่านภาษาอังกฤษเป็นไทย
    public async Task<List<ProductDocument>> SearchWithTransliterationAsync(
        string query)
    {
        // ลอง search ทั้งแบบ original และ transliterated
        var tasks = new[]
        {
            SearchCrossLanguageAsync(query),
            SearchCrossLanguageAsync(Transliterate(query))
        };

        var results = await Task.WhenAll(tasks);

        // รวมผลลัพธ์และ deduplicate
        return results
            .SelectMany(r => r.Documents)
            .GroupBy(p => p.Id)
            .Select(g => g.First())
            .Take(20)
            .ToList();
    }

    private string Transliterate(string text)
    {
        // Simple Thai-English transliteration table
        var transliterationMap = new Dictionary<string, string>
        {
            {"กรุงเทพ", "bangkok"},
            {"เสื้อผ้า", "clothes"},
            {"อาหาร", "food"},
            // เพิ่มเติมตามต้องการ
        };

        foreach (var kvp in transliterationMap)
        {
            text = text.Replace(kvp.Key, kvp.Value, StringComparison.OrdinalIgnoreCase);
        }

        return text;
    }
}
```

---

## Step 848: Geo-Search สำหรับร้านค้าใกล้เคียง

### Geo Search Service

```csharp
// Services/GeoSearchService.cs
public class GeoSearchService
{
    private readonly ElasticsearchClient _client;
    private readonly ElasticsearchSettings _settings;

    public GeoSearchService(
        ElasticsearchClient client,
        IOptions<ElasticsearchSettings> settings)
    {
        _client = client;
        _settings = settings.Value;
    }

    // ค้นหาสินค้าจากร้านค้าใกล้เคียง
    public async Task<GeoSearchResult> SearchNearbyProductsAsync(
        double latitude,
        double longitude,
        double radiusKm,
        string? productQuery = null,
        int size = 20)
    {
        var filterClauses = new List<Query>
        {
            // Geo-distance filter
            new GeoDistanceQuery
            {
                Field = "location",
                Location = new GeoLocation(latitude, longitude),
                Distance = $"{radiusKm}km",
                DistanceType = GeoDistanceType.Arc
            },
            new TermQuery("isActive") { Value = true }
        };

        Query mainQuery;

        if (!string.IsNullOrWhiteSpace(productQuery))
        {
            mainQuery = new BoolQuery
            {
                Must = new List<Query>
                {
                    new MultiMatchQuery
                    {
                        Query = productQuery,
                        Fields = new[] { "name^5", "nameThai^5", "category^2", "brand^3" },
                        Fuzziness = new Fuzziness("AUTO")
                    }
                },
                Filter = filterClauses
            };
        }
        else
        {
            mainQuery = new BoolQuery { Filter = filterClauses };
        }

        var response = await _client.SearchAsync<ProductDocument>(s => s
            .Index(_settings.ProductsIndex)
            .Size(size)
            .Query(q => mainQuery)
            .Sort(so => so
                // เรียงตามระยะทาง
                .GeoDistance(g => g
                    .Field("location")
                    .Location(new GeoLocation(latitude, longitude))
                    .Order(SortOrder.Asc)
                    .Unit(DistanceUnit.Kilometers)
                    .Mode(SortMode.Min)))
            .ScriptFields(sf => sf
                // คำนวณระยะทางใน script field
                .Add("distance", sfd => sfd
                    .Script(sc => sc
                        .Source(
                            @"doc['location'].arcDistance(params.lat, params.lon) / 1000")
                        .Params(new Dictionary<string, object>
                        {
                            { "lat", latitude },
                            { "lon", longitude }
                        })))));

        return new GeoSearchResult
        {
            Products = response.Hits.Select(h => new ProductWithDistance
            {
                Product = h.Source!,
                DistanceKm = h.Fields?.TryGetValue("distance", out var dist) == true
                    ? ((JsonElement)dist).GetDouble()
                    : 0
            }).ToList(),
            TotalCount = (int)(response.Total ?? 0),
            SearchCenter = new GeoPoint { Lat = latitude, Lon = longitude },
            RadiusKm = radiusKm
        };
    }

    // ค้นหาร้านค้าใกล้เคียงพร้อม aggregation ตาม store
    public async Task<List<NearbyStore>> FindNearbyStoresAsync(
        double latitude,
        double longitude,
        double radiusKm)
    {
        var response = await _client.SearchAsync<ProductDocument>(s => s
            .Index(_settings.ProductsIndex)
            .Size(0)  // ไม่ต้องการ documents, แค่ aggregation
            .Query(q => q
                .GeoDistance(g => g
                    .Field("location")
                    .Location(new GeoLocation(latitude, longitude))
                    .Distance($"{radiusKm}km")))
            .Aggregations(a => a
                .Terms("stores", t => t
                    .Field("store.id")
                    .Size(50)
                    .Aggregations(sa => sa
                        .Min("min_price", m => m.Field("price"))
                        .Max("max_price", m => m.Field("price"))
                        .Avg("avg_rating", av => av.Field("averageRating"))
                        .ValueCount("product_count", vc => vc.Field("id"))
                        .TopHits("store_info", th => th
                            .Size(1)
                            .Source(src => src
                                .Includes(new[]
                                {
                                    "store.name",
                                    "store.city",
                                    "store.rating",
                                    "location"
                                })))))));

        var nearbyStores = new List<NearbyStore>();

        if (response.Aggregations?.TryGetValue("stores", out var storesAgg) == true &&
            storesAgg is LongTermsAggregate storeTerms)
        {
            foreach (var bucket in storeTerms.Buckets)
            {
                var topHit = bucket.Aggregations.TryGetValue("store_info", out var th)
                    ? (th as TopHitsAggregate)?.Hits.Hits.FirstOrDefault()
                    : null;

                var storeInfo = topHit?.Source?.As<ProductDocument>()?.Store;
                var location = topHit?.Source?.As<ProductDocument>()?.Location;

                if (storeInfo != null)
                {
                    var distance = location != null
                        ? CalculateDistance(latitude, longitude, location.Lat, location.Lon)
                        : 0;

                    nearbyStores.Add(new NearbyStore
                    {
                        StoreId = (int)bucket.Key,
                        StoreName = storeInfo.Name,
                        City = storeInfo.City,
                        StoreRating = storeInfo.Rating,
                        ProductCount = (int)(bucket.DocCount),
                        DistanceKm = distance,
                        Location = location
                    });
                }
            }
        }

        return nearbyStores.OrderBy(s => s.DistanceKm).ToList();
    }

    private double CalculateDistance(
        double lat1, double lon1,
        double lat2, double lon2)
    {
        const double R = 6371; // Radius of Earth in km
        var dLat = (lat2 - lat1) * Math.PI / 180;
        var dLon = (lon2 - lon1) * Math.PI / 180;
        var a = Math.Sin(dLat / 2) * Math.Sin(dLat / 2) +
                Math.Cos(lat1 * Math.PI / 180) * Math.Cos(lat2 * Math.PI / 180) *
                Math.Sin(dLon / 2) * Math.Sin(dLon / 2);
        var c = 2 * Math.Atan2(Math.Sqrt(a), Math.Sqrt(1 - a));
        return R * c;
    }
}

public class GeoSearchResult
{
    public List<ProductWithDistance> Products { get; set; } = new();
    public int TotalCount { get; set; }
    public GeoPoint SearchCenter { get; set; } = null!;
    public double RadiusKm { get; set; }
}

public class NearbyStore
{
    public int StoreId { get; set; }
    public string StoreName { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
    public double StoreRating { get; set; }
    public int ProductCount { get; set; }
    public double DistanceKm { get; set; }
    public GeoPoint? Location { get; set; }
}
```

---

## Step 849: Similar Products Recommendation (Collaborative Filtering)

### แนวคิด Collaborative Filtering

Collaborative Filtering แนะนำสินค้าโดยอิงจากพฤติกรรมของผู้ใช้ที่คล้ายกัน เราจะ implement แบบ Item-based CF ซึ่งดูว่าสินค้าใดมักถูกซื้อร่วมกัน

```csharp
// Services/RecommendationService.cs
using StackExchange.Redis;

public class RecommendationService
{
    private readonly ApplicationDbContext _context;
    private readonly ElasticsearchClient _client;
    private readonly IDatabase _redisDb;
    private readonly ElasticsearchSettings _settings;

    public RecommendationService(
        ApplicationDbContext context,
        ElasticsearchClient client,
        IConnectionMultiplexer redis,
        IOptions<ElasticsearchSettings> settings)
    {
        _context = context;
        _client = client;
        _redisDb = redis.GetDatabase();
        _settings = settings.Value;
    }

    // ค้นหาสินค้าที่คล้ายกันโดยใช้ Elasticsearch More Like This
    public async Task<List<ProductDocument>> GetSimilarProductsAsync(
        int productId,
        int count = 10)
    {
        // ใช้ More Like This query ของ Elasticsearch
        var response = await _client.SearchAsync<ProductDocument>(s => s
            .Index(_settings.ProductsIndex)
            .Size(count)
            .Query(q => q
                .MoreLikeThis(mlt => mlt
                    .Fields(new[] { "name", "nameThai", "category", "tags", "brand" })
                    .Like(l => l
                        .Document(d => d
                            .Index(_settings.ProductsIndex)
                            .Id(productId.ToString())))
                    .MinTermFrequency(1)
                    .MaxQueryTerms(25)
                    .MinDocFrequency(1)
                    .MinimumShouldMatch("30%")))
            .Query(q => q
                .Bool(b => b
                    .Must(m => m
                        .MoreLikeThis(mlt => mlt
                            .Fields(new[]
                            {
                                "name^3", "nameThai^3",
                                "category^2", "brand^2",
                                "tags^1"
                            })
                            .Like(l => l
                                .Document(d => d
                                    .Index(_settings.ProductsIndex)
                                    .Id(productId.ToString())))
                            .MinTermFrequency(1)
                            .MaxQueryTerms(30)))
                    .MustNot(mn => mn
                        .Ids(ids => ids.Values(productId.ToString())))
                    .Filter(f => f
                        .Term(t => t.Field("isActive").Value(true))))));

        return response.Documents.ToList();
    }

    // Item-based Collaborative Filtering จาก order history
    public async Task<List<int>> GetFrequentlyBoughtTogetherAsync(
        int productId,
        int count = 5)
    {
        // ดึงจาก Redis cache ก่อน
        var cacheKey = $"fbt:{productId}";
        var cached = await _redisDb.StringGetAsync(cacheKey);
        if (cached.HasValue)
        {
            return System.Text.Json.JsonSerializer.Deserialize<List<int>>(cached!)!;
        }

        // ค้นหาจาก database
        var relatedProductIds = await _context.OrderItems
            .Where(oi => oi.ProductId == productId)
            .SelectMany(oi => oi.Order.OrderItems
                .Where(other => other.ProductId != productId)
                .Select(other => other.ProductId))
            .GroupBy(pid => pid)
            .OrderByDescending(g => g.Count())
            .Take(count)
            .Select(g => g.Key)
            .ToListAsync();

        // Cache ผลลัพธ์ 1 ชั่วโมง
        await _redisDb.StringSetAsync(
            cacheKey,
            System.Text.Json.JsonSerializer.Serialize(relatedProductIds),
            TimeSpan.FromHours(1));

        return relatedProductIds;
    }

    // User-based recommendations จาก viewing history
    public async Task<List<ProductDocument>> GetPersonalizedRecommendationsAsync(
        int userId,
        int count = 20)
    {
        // ดึง products ที่ user เคยดูหรือซื้อ
        var userHistory = await _context.UserActivityLogs
            .Where(al => al.UserId == userId && al.CreatedAt >= DateTime.UtcNow.AddDays(-30))
            .Select(al => new { al.ProductId, al.ActivityType, al.CreatedAt })
            .OrderByDescending(al => al.CreatedAt)
            .Take(50)
            .ToListAsync();

        if (!userHistory.Any())
        {
            return await GetTrendingProductsAsync(count);
        }

        var viewedProductIds = userHistory
            .Select(h => h.ProductId)
            .Distinct()
            .ToList();

        // ใช้ Elasticsearch More Like This กับ products ที่ user เคยดู
        var response = await _client.SearchAsync<ProductDocument>(s => s
            .Index(_settings.ProductsIndex)
            .Size(count)
            .Query(q => q
                .Bool(b => b
                    .Must(m => m
                        .MoreLikeThis(mlt => mlt
                            .Fields(new[]
                            {
                                "category^3", "brand^2",
                                "tags^2", "name^1"
                            })
                            .Like(viewedProductIds
                                .Take(10)
                                .Select(pid => new LikeDocument
                                {
                                    Index = _settings.ProductsIndex,
                                    Id = pid.ToString()
                                })
                                .ToArray())
                            .MinTermFrequency(1)))
                    .MustNot(mn => mn
                        .Ids(ids => ids
                            .Values(viewedProductIds.Select(id => id.ToString()).ToArray())))
                    .Filter(f => f
                        .Term(t => t.Field("isActive").Value(true))))));

        return response.Documents.ToList();
    }

    // Trending products
    public async Task<List<ProductDocument>> GetTrendingProductsAsync(int count = 20)
    {
        var thirtyDaysAgo = DateTime.UtcNow.AddDays(-30);

        var response = await _client.SearchAsync<ProductDocument>(s => s
            .Index(_settings.ProductsIndex)
            .Size(count)
            .Query(q => q
                .Bool(b => b
                    .Filter(f => f
                        .Term(t => t.Field("isActive").Value(true)))))
            .Sort(so => so
                .Field("salesCount", new FieldSort { Order = SortOrder.Desc })
                .Field("viewCount", new FieldSort { Order = SortOrder.Desc })));

        return response.Documents.ToList();
    }
}
```

---

## Step 850: Search Analytics และ A/B Testing สำหรับ Ranking

### Search Event Tracking

```csharp
// Services/SearchAnalyticsService.cs
using StackExchange.Redis;

public class SearchAnalyticsService
{
    private readonly ApplicationDbContext _context;
    private readonly IDatabase _redisDb;
    private readonly ILogger<SearchAnalyticsService> _logger;

    public SearchAnalyticsService(
        ApplicationDbContext context,
        IConnectionMultiplexer redis,
        ILogger<SearchAnalyticsService> logger)
    {
        _context = context;
        _redisDb = redis.GetDatabase();
        _logger = logger;
    }

    // บันทึก search event
    public async Task TrackSearchAsync(SearchEvent searchEvent)
    {
        try
        {
            // บันทึกลง database
            _context.SearchEvents.Add(searchEvent);
            await _context.SaveChangesAsync();

            // อัปเดต popular searches ใน Redis sorted set
            var dateKey = $"popular_searches:{DateTime.UtcNow:yyyyMMdd}";
            await _redisDb.SortedSetIncrementAsync(
                dateKey,
                searchEvent.Query.ToLowerInvariant(),
                1);

            // Set expiry 7 วัน
            await _redisDb.KeyExpireAsync(dateKey, TimeSpan.FromDays(7));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to track search event");
        }
    }

    // บันทึก click event (user คลิกสินค้าจาก search results)
    public async Task TrackClickAsync(
        string searchId,
        int productId,
        int position,
        string abVariant)
    {
        try
        {
            var clickEvent = new SearchClickEvent
            {
                SearchId = searchId,
                ProductId = productId,
                Position = position,
                AbVariant = abVariant,
                ClickedAt = DateTime.UtcNow
            };

            _context.SearchClickEvents.Add(clickEvent);
            await _context.SaveChangesAsync();

            // อัปเดต CTR stats ใน Redis
            var statsKey = $"search_stats:{abVariant}:{DateTime.UtcNow:yyyyMMdd}";
            await _redisDb.HashIncrementAsync(statsKey, "clicks");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to track click event");
        }
    }

    // ดึง popular searches
    public async Task<List<string>> GetPopularSearchesAsync(
        int count = 10,
        int daysBack = 7)
    {
        var popularSearches = new Dictionary<string, double>();

        // รวมข้อมูลจากหลายวัน
        for (int i = 0; i < daysBack; i++)
        {
            var dateKey = $"popular_searches:{DateTime.UtcNow.AddDays(-i):yyyyMMdd}";
            var daySearches = await _redisDb.SortedSetRangeByRankWithScoresAsync(
                dateKey, 0, count - 1, Order.Descending);

            foreach (var entry in daySearches)
            {
                var query = entry.Element.ToString();
                if (popularSearches.ContainsKey(query))
                    popularSearches[query] += entry.Score;
                else
                    popularSearches[query] = entry.Score;
            }
        }

        return popularSearches
            .OrderByDescending(kvp => kvp.Value)
            .Take(count)
            .Select(kvp => kvp.Key)
            .ToList();
    }

    // Search Analytics Report
    public async Task<SearchAnalyticsReport> GetAnalyticsReportAsync(
        DateTime startDate,
        DateTime endDate)
    {
        var searchEvents = await _context.SearchEvents
            .Where(se => se.SearchedAt >= startDate && se.SearchedAt <= endDate)
            .ToListAsync();

        var clickEvents = await _context.SearchClickEvents
            .Where(ce => ce.ClickedAt >= startDate && ce.ClickedAt <= endDate)
            .ToListAsync();

        var totalSearches = searchEvents.Count;
        var uniqueQueries = searchEvents.Select(se => se.Query.ToLowerInvariant()).Distinct().Count();
        var zeroResultSearches = searchEvents.Count(se => se.ResultCount == 0);
        var totalClicks = clickEvents.Count;

        // คำนวณ Click-Through Rate
        var ctr = totalSearches > 0 ? (double)totalClicks / totalSearches : 0;

        // Top queries ที่ไม่มีผลลัพธ์ (สิ่งที่ต้องปรับปรุง)
        var noResultQueries = searchEvents
            .Where(se => se.ResultCount == 0)
            .GroupBy(se => se.Query.ToLowerInvariant())
            .OrderByDescending(g => g.Count())
            .Take(20)
            .Select(g => new { Query = g.Key, Count = g.Count() })
            .ToList();

        // A/B Test comparison
        var abTestResults = clickEvents
            .GroupBy(ce => ce.AbVariant)
            .Select(g => new AbTestVariantResult
            {
                Variant = g.Key,
                Clicks = g.Count(),
                AveragePosition = g.Average(ce => ce.Position),
                UniqueSearchIds = g.Select(ce => ce.SearchId).Distinct().Count()
            })
            .ToList();

        return new SearchAnalyticsReport
        {
            StartDate = startDate,
            EndDate = endDate,
            TotalSearches = totalSearches,
            UniqueQueries = uniqueQueries,
            ZeroResultRate = totalSearches > 0
                ? (double)zeroResultSearches / totalSearches
                : 0,
            ClickThroughRate = ctr,
            AverageResultsPerSearch = searchEvents.Any()
                ? searchEvents.Average(se => se.ResultCount)
                : 0,
            TopNoResultQueries = noResultQueries
                .Select(q => q.Query)
                .ToList(),
            AbTestResults = abTestResults
        };
    }
}
```

### A/B Testing สำหรับ Search Ranking

```csharp
// Services/SearchAbTestService.cs
public class SearchAbTestService
{
    private readonly IDistributedCache _cache;

    public SearchAbTestService(IDistributedCache cache)
    {
        _cache = cache;
    }

    // กำหนด variant สำหรับ user (50/50 split)
    public string GetVariantForUser(string userId)
    {
        // Hash userId เพื่อให้ user เดิมได้ variant เดิมเสมอ
        var hash = Math.Abs(userId.GetHashCode());
        return hash % 2 == 0 ? "control" : "treatment";
    }

    // ปรับ search query ตาม variant
    public SearchRequest ApplyVariantStrategy(
        SearchRequest baseRequest,
        string variant)
    {
        if (variant == "treatment")
        {
            // Treatment group: ใช้ Function Score ที่เน้น freshness มากกว่า
            return ApplyFreshnessBoostStrategy(baseRequest);
        }

        // Control group: ใช้ default relevance scoring
        return baseRequest;
    }

    private SearchRequest ApplyFreshnessBoostStrategy(SearchRequest request)
    {
        // ปรับ boost ให้สินค้าใหม่มีคะแนนสูงขึ้น 20%
        // (implementation ขึ้นอยู่กับ query structure)
        return request;
    }
}
```

### Search Controller

```csharp
// Controllers/SearchController.cs
[ApiController]
[Route("api/[controller]")]
public class SearchController : ControllerBase
{
    private readonly FacetedSearchService _facetedSearch;
    private readonly AutocompleteService _autocomplete;
    private readonly GeoSearchService _geoSearch;
    private readonly RecommendationService _recommendations;
    private readonly SearchAnalyticsService _analytics;
    private readonly SearchAbTestService _abTest;

    public SearchController(
        FacetedSearchService facetedSearch,
        AutocompleteService autocomplete,
        GeoSearchService geoSearch,
        RecommendationService recommendations,
        SearchAnalyticsService analytics,
        SearchAbTestService abTest)
    {
        _facetedSearch = facetedSearch;
        _autocomplete = autocomplete;
        _geoSearch = geoSearch;
        _recommendations = recommendations;
        _analytics = analytics;
        _abTest = abTest;
    }

    [HttpGet]
    public async Task<IActionResult> Search([FromQuery] ProductSearchRequest request)
    {
        var userId = User.FindFirst("sub")?.Value ?? "anonymous";
        var variant = _abTest.GetVariantForUser(userId);
        var searchId = Guid.NewGuid().ToString();

        var result = await _facetedSearch.SearchWithFacetsAsync(request);

        // Track search event (fire-and-forget)
        _ = _analytics.TrackSearchAsync(new SearchEvent
        {
            SearchId = searchId,
            Query = request.Query,
            UserId = userId,
            ResultCount = result.TotalCount,
            Page = request.Page,
            AbVariant = variant,
            Filters = System.Text.Json.JsonSerializer.Serialize(new
            {
                request.Categories,
                request.MinPrice,
                request.MaxPrice,
                request.MinRating
            }),
            SearchedAt = DateTime.UtcNow
        });

        return Ok(new
        {
            searchId,
            variant,
            result.Products,
            result.TotalCount,
            result.Page,
            result.PageSize,
            result.TotalPages,
            result.Facets
        });
    }

    [HttpGet("autocomplete")]
    public async Task<IActionResult> Autocomplete(
        [FromQuery] string prefix,
        [FromQuery] int max = 10)
    {
        var result = await _autocomplete.GetSuggestionsAsync(prefix, max);
        return Ok(result);
    }

    [HttpGet("nearby")]
    public async Task<IActionResult> SearchNearby(
        [FromQuery] double lat,
        [FromQuery] double lon,
        [FromQuery] double radius = 5.0,
        [FromQuery] string? q = null)
    {
        var result = await _geoSearch.SearchNearbyProductsAsync(lat, lon, radius, q);
        return Ok(result);
    }

    [HttpGet("{productId}/similar")]
    public async Task<IActionResult> GetSimilarProducts(
        int productId,
        [FromQuery] int count = 10)
    {
        var products = await _recommendations.GetSimilarProductsAsync(productId, count);
        return Ok(products);
    }

    [HttpGet("{productId}/frequently-bought-together")]
    public async Task<IActionResult> GetFrequentlyBoughtTogether(int productId)
    {
        var productIds = await _recommendations.GetFrequentlyBoughtTogetherAsync(productId);
        return Ok(productIds);
    }

    [HttpGet("recommendations")]
    [Authorize]
    public async Task<IActionResult> GetPersonalizedRecommendations(
        [FromQuery] int count = 20)
    {
        var userId = int.Parse(User.FindFirst("sub")?.Value ?? "0");
        var products = await _recommendations.GetPersonalizedRecommendationsAsync(userId, count);
        return Ok(products);
    }

    [HttpPost("track-click")]
    public async Task<IActionResult> TrackClick([FromBody] TrackClickRequest request)
    {
        await _analytics.TrackClickAsync(
            request.SearchId,
            request.ProductId,
            request.Position,
            request.AbVariant);
        return NoContent();
    }

    [HttpGet("analytics")]
    [Authorize(Roles = "Admin")]
    public async Task<IActionResult> GetAnalytics(
        [FromQuery] DateTime startDate,
        [FromQuery] DateTime endDate)
    {
        var report = await _analytics.GetAnalyticsReportAsync(startDate, endDate);
        return Ok(report);
    }

    [HttpGet("trending")]
    public async Task<IActionResult> GetTrending([FromQuery] int count = 10)
    {
        var searches = await _analytics.GetPopularSearchesAsync(count);
        return Ok(searches);
    }
}
```

### Program.cs Configuration

```csharp
// Program.cs - เพิ่ม search services
var builder = WebApplication.CreateBuilder(args);

// Elasticsearch
builder.Services.AddElasticsearch(builder.Configuration);

// Search Services
builder.Services.AddScoped<FacetedSearchService>();
builder.Services.AddScoped<AutocompleteService>();
builder.Services.AddScoped<GeoSearchService>();
builder.Services.AddScoped<RecommendationService>();
builder.Services.AddScoped<SearchAnalyticsService>();
builder.Services.AddScoped<SearchAbTestService>();
builder.Services.AddScoped<ThaiSearchService>();
builder.Services.AddScoped<RelevanceScoringService>();
builder.Services.AddScoped<IProductIndexerService, ProductIndexerService>();

// Background sync channel
builder.Services.AddSingleton(Channel.CreateUnbounded<SearchSyncMessage>());
builder.Services.AddHostedService<SearchSyncWorker>();

// Elasticsearch Index initialization
builder.Services.AddScoped<ElasticsearchIndexService>();

var app = builder.Build();

// Initialize Elasticsearch index on startup
using (var scope = app.Services.CreateScope())
{
    var indexService = scope.ServiceProvider
        .GetRequiredService<ElasticsearchIndexService>();
    await indexService.CreateProductsIndexAsync();
}

app.MapControllers();
app.Run();
```

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part 85

### ความสามารถหลักที่ implement ใน Steps 841-850

| Step | ฟีเจอร์ | เทคโนโลยี |
|------|---------|-----------|
| 841 | Full-text search | PostgreSQL tsvector, GIN Index |
| 842 | Elasticsearch integration | Elastic.Clients.Elasticsearch |
| 843 | Search indexing pipeline | EF Core + Bulk API |
| 844 | Faceted search | Aggregations, Terms, Range |
| 845 | Autocomplete | Edge NGram, Completion Suggester |
| 846 | Relevance scoring | Function Score, Boosting |
| 847 | Thai text search | Thai Analyzer, Tokenizer |
| 848 | Geo-search | Geo Distance Query, Sorting |
| 849 | Recommendations | More Like This, Collaborative Filtering |
| 850 | Search analytics | Event Tracking, A/B Testing |

### Best Practices ที่ควรจำ

1. **GIN Index**: ใช้กับ tsvector เสมอเพื่อประสิทธิภาพ
2. **Bulk API**: ใช้สำหรับ indexing จำนวนมาก ไม่ใช้ single index
3. **Caching**: Cache autocomplete results เพื่อลด load บน Elasticsearch
4. **Background sync**: อย่า block main thread ด้วยการ sync
5. **A/B Testing**: วัดผล CTR เสมอก่อนเปลี่ยน ranking algorithm
6. **Thai analyzer**: ต้อง configure ใน index creation เท่านั้น ไม่สามารถเปลี่ยนหลัง create แล้ว
7. **Relevance tuning**: ทดสอบ boost values ด้วย real data เสมอ

---

**ก่อนหน้า → [Part 84: Capstone Real-time](part84-capstone-realtime.md)**
**ต่อไป → [Part 86: Capstone - Payment & Financial](part86-capstone-payments.md)**
