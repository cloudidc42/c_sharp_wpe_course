# Part 42: EF Core Migrations & Advanced Queries
## ขั้นตอนที่ 411-420: Migrations, Relationships, Performance

---

## 🎯 เป้าหมายของ Part นี้
- Migrations workflow
- Schema changes
- Complex relationships (TPH, TPT)
- Owned entities
- Value objects
- Query optimization (AsNoTracking, Projection)
- Raw SQL และ Stored Procedures
- Connection Resiliency

---

## ขั้นตอนที่ 411: Migrations Workflow

```bash
# 1. สร้าง migration ครั้งแรก
dotnet ef migrations add InitialCreate

# ไฟล์ที่ถูกสร้าง:
# Migrations/
#   20241001_InitialCreate.cs          ← Up() / Down()
#   20241001_InitialCreate.Designer.cs ← snapshot metadata
#   BlogDbContextModelSnapshot.cs      ← current model state

# 2. Apply migration
dotnet ef database update

# 3. เพิ่ม column ใหม่ในภายหลัง
# แก้ไข Post.cs: เพิ่ม ViewCount
dotnet ef migrations add AddViewCount
dotnet ef database update

# 4. Rollback
dotnet ef database update InitialCreate  # rollback to specific

# 5. Script SQL (สำหรับ production)
dotnet ef migrations script --idempotent -o migration.sql
```

---

## ขั้นตอนที่ 412: Migration File ตัวอย่าง

```csharp
// Migrations/20241001_InitialCreate.cs
public partial class InitialCreate : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Create Authors table
        migrationBuilder.CreateTable(
            name: "Authors",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false)
                    .Annotation("Sqlite:Autoincrement", true),
                Name = table.Column<string>(maxLength: 200, nullable: false),
                Email = table.Column<string>(maxLength: 200, nullable: false),
                Bio = table.Column<string>(nullable: true),
                CreatedAt = table.Column<DateTime>(nullable: false)
            },
            constraints: t => t.PrimaryKey("PK_Authors", x => x.Id));
        
        migrationBuilder.CreateIndex(
            name: "IX_Authors_Email",
            table: "Authors",
            column: "Email",
            unique: true);
        
        // Add Posts table with FK
        migrationBuilder.CreateTable(
            name: "Posts",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false).Annotation("Sqlite:Autoincrement", true),
                Title = table.Column<string>(maxLength: 500, nullable: false),
                AuthorId = table.Column<int>(nullable: false),
                // ...
            },
            constraints: t =>
            {
                t.PrimaryKey("PK_Posts", x => x.Id);
                t.ForeignKey("FK_Posts_Authors_AuthorId", x => x.AuthorId,
                    principalTable: "Authors", principalColumn: "Id",
                    onDelete: ReferentialAction.Cascade);
            });
        
        // Seed data
        migrationBuilder.InsertData("Categories", new[] { "Id", "Name" }, 
            new object[] { 1, "เทคโนโลยี" });
    }
    
    protected override void Down(MigrationBuilder migrationBuilder)
    {
        // Reverse order!
        migrationBuilder.DropTable("Posts");
        migrationBuilder.DropTable("Authors");
    }
}

// Custom migration (manual SQL)
public partial class AddFullTextIndex : Migration
{
    protected override void Up(MigrationBuilder mb)
    {
        mb.Sql("CREATE INDEX IF NOT EXISTS IX_Posts_Title ON Posts(Title)");
        mb.Sql("UPDATE Posts SET Slug = lower(replace(Title, ' ', '-')) WHERE Slug = ''");
    }
    
    protected override void Down(MigrationBuilder mb)
    {
        mb.Sql("DROP INDEX IF EXISTS IX_Posts_Title");
    }
}
```

---

## ขั้นตอนที่ 413: Owned Entities & Value Objects

```csharp
// Value object - Address (no primary key, owned by entity)
public record Address
{
    public string Street { get; init; } = "";
    public string City { get; init; } = "";
    public string Province { get; init; } = "";
    public string PostalCode { get; init; } = "";
    public string Country { get; init; } = "Thailand";
    
    public string Full => $"{Street}, {City}, {Province} {PostalCode}";
}

// Entity that owns the value object
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public Address BillingAddress { get; set; } = new();  // Owned
    public Address? ShippingAddress { get; set; }        // Optional owned
}

// Configure in OnModelCreating
modelBuilder.Entity<Customer>(e =>
{
    // Map owned entity - columns stored inline in Customer table
    e.OwnsOne(c => c.BillingAddress, addr =>
    {
        addr.Property(a => a.Street).HasColumnName("BillingStreet");
        addr.Property(a => a.City).HasColumnName("BillingCity");
        addr.Property(a => a.PostalCode).HasColumnName("BillingPostal");
    });
    
    e.OwnsOne(c => c.ShippingAddress, addr =>
    {
        addr.Property(a => a.Street).HasColumnName("ShipStreet").IsRequired(false);
        addr.Property(a => a.City).HasColumnName("ShipCity").IsRequired(false);
    });
});
```

---

## ขั้นตอนที่ 414: Inheritance (TPH - Table Per Hierarchy)

```csharp
// Base class
public abstract class ContentBase
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public int AuthorId { get; set; }
    public Author Author { get; set; } = null!;
    
    // Discriminator stored as column
    public string ContentType => GetType().Name;
}

// Derived types
public class Article : ContentBase
{
    public string Body { get; set; } = "";
    public string Slug { get; set; } = "";
    public bool IsPublished { get; set; }
}

public class Video : ContentBase
{
    public string VideoUrl { get; set; } = "";
    public int DurationSeconds { get; set; }
    public string Thumbnail { get; set; } = "";
}

public class Podcast : ContentBase
{
    public string AudioUrl { get; set; } = "";
    public int EpisodeNumber { get; set; }
    public int DurationMinutes { get; set; }
}

// Configure TPH (all in one table)
modelBuilder.Entity<ContentBase>(e =>
{
    e.HasDiscriminator<string>("ContentType")
     .HasValue<Article>("Article")
     .HasValue<Video>("Video")
     .HasValue<Podcast>("Podcast");
    
    // Nullable columns for subclass fields
    e.Property<string?>("Body").IsRequired(false);
    e.Property<string?>("VideoUrl").IsRequired(false);
});

// Query subtype
var articles = await context.Set<Article>()
    .Where(a => a.IsPublished)
    .ToListAsync();

// Query all content
var allContent = await context.Set<ContentBase>()
    .OfType<Article>() // filter by type
    .ToListAsync();
```

---

## ขั้นตอนที่ 415: Query Performance

```csharp
// 1. AsNoTracking - read-only queries (faster, less memory)
var posts = await context.Posts
    .AsNoTracking()
    .Where(p => p.IsPublished)
    .Include(p => p.Author)
    .ToListAsync();

// 2. AsNoTrackingWithIdentityResolution - no tracking but dedup navigation
var postsWithNav = await context.Posts
    .AsNoTrackingWithIdentityResolution()
    .Include(p => p.Author)
    .ToListAsync();

// 3. Projection - select only needed columns (avoids loading BLOBs)
var postSummaries = await context.Posts
    .AsNoTracking()
    .Where(p => p.IsPublished)
    .Select(p => new PostSummaryDto
    {
        Id = p.Id,
        Title = p.Title,
        Slug = p.Slug,
        AuthorName = p.Author.Name,
        CategoryName = p.Category != null ? p.Category.Name : "ไม่มีหมวด",
        CommentCount = p.Comments.Count(c => c.IsApproved),
        CreatedAt = p.CreatedAt
    })
    .OrderByDescending(p => p.CreatedAt)
    .ToListAsync();

// 4. Split Query - avoid cartesian explosion
var postsWithAll = await context.Posts
    .AsSplitQuery()  // Issues separate SQL queries per Include
    .Include(p => p.PostTags).ThenInclude(pt => pt.Tag)
    .Include(p => p.Comments)
    .ToListAsync();

// 5. Compiled query - reuse query plan
private static readonly Func<BlogDbContext, string, Task<Post?>> GetPostBySlug =
    EF.CompileAsyncQuery((BlogDbContext ctx, string slug) =>
        ctx.Posts.Include(p => p.Author).FirstOrDefault(p => p.Slug == slug));

// Usage:
var post = await GetPostBySlug(context, "my-post-slug");

// 6. Raw SQL for complex queries
var stats = await context.Database
    .SqlQueryRaw<CategoryStatDto>(@"
        SELECT c.Name AS CategoryName, COUNT(p.Id) AS PostCount, 
               AVG(CAST(p.ViewCount AS REAL)) AS AvgViews
        FROM Categories c
        LEFT JOIN Posts p ON p.CategoryId = c.Id AND p.IsPublished = 1
        GROUP BY c.Id, c.Name
        HAVING COUNT(p.Id) > 0
        ORDER BY PostCount DESC")
    .ToListAsync();
```

---

## ขั้นตอนที่ 416: Concurrency & Transactions

```csharp
// Optimistic Concurrency with RowVersion
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
    
    [Timestamp]  // or Fluent API: .IsRowVersion()
    public byte[] RowVersion { get; set; } = null!;
}

// Handle concurrency exception
try
{
    product.Price = 999;
    await context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    foreach (var entry in ex.Entries)
    {
        var dbValues = await entry.GetDatabaseValuesAsync();
        if (dbValues == null)
        {
            // Record was deleted
            Console.WriteLine("ข้อมูลถูกลบโดยผู้ใช้อื่น");
        }
        else
        {
            // Show conflict
            var dbProduct = (Product)dbValues.ToObject();
            Console.WriteLine($"ราคาปัจจุบันในฐานข้อมูล: {dbProduct.Price}");
            // Refresh and retry...
            entry.OriginalValues.SetValues(dbValues);
        }
    }
}

// Explicit transaction
using var transaction = await context.Database.BeginTransactionAsync();
try
{
    var order = new Order { CustomerId = 1 };
    await context.Orders.AddAsync(order);
    await context.SaveChangesAsync();
    
    // Deduct stock
    await context.Products
        .Where(p => p.Id == 5)
        .ExecuteUpdateAsync(s => s.SetProperty(p => p.Stock, p => p.Stock - 1));
    
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

---

## ขั้นตอนที่ 417-420: Complete Repository with Specification Pattern

```csharp
// Specification Pattern - encapsulate query logic
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> Criteria { get; }
    public List<Expression<Func<T, object>>> Includes { get; } = new();
    public Expression<Func<T, object>>? OrderBy { get; protected set; }
    public Expression<Func<T, object>>? OrderByDesc { get; protected set; }
    public int? Take { get; protected set; }
    public int? Skip { get; protected set; }
    
    protected void AddInclude(Expression<Func<T, object>> include) => Includes.Add(include);
    protected void AddOrderBy(Expression<Func<T, object>> orderBy) => OrderBy = orderBy;
    protected void AddOrderByDescending(Expression<Func<T, object>> orderByDesc) => OrderByDesc = orderByDesc;
    protected void ApplyPaging(int skip, int take) { Skip = skip; Take = take; }
}

// Concrete specification
public class PublishedPostsSpec : Specification<Post>
{
    public override Expression<Func<Post, bool>> Criteria => p => p.IsPublished;
    
    public PublishedPostsSpec(int page = 1, int pageSize = 10)
    {
        AddInclude(p => p.Author);
        AddInclude(p => p.Category!);
        AddOrderByDescending(p => p.PublishedAt!);
        ApplyPaging((page - 1) * pageSize, pageSize);
    }
}

public class PostsByCategorySpec : Specification<Post>
{
    public override Expression<Func<Post, bool>> Criteria => 
        p => p.IsPublished && p.CategoryId == _categoryId;
    
    private readonly int _categoryId;
    
    public PostsByCategorySpec(int categoryId)
    {
        _categoryId = categoryId;
        AddInclude(p => p.Author);
        AddOrderByDescending(p => p.PublishedAt!);
    }
}

// Generic repository using specification
public class SpecificationRepository<T> where T : class
{
    protected readonly BlogDbContext _ctx;
    
    public SpecificationRepository(BlogDbContext ctx) => _ctx = ctx;
    
    public async Task<List<T>> ListAsync(Specification<T> spec)
    {
        return await ApplySpec(_ctx.Set<T>().AsNoTracking(), spec).ToListAsync();
    }
    
    public async Task<T?> FirstOrDefaultAsync(Specification<T> spec)
    {
        return await ApplySpec(_ctx.Set<T>().AsNoTracking(), spec).FirstOrDefaultAsync();
    }
    
    public async Task<int> CountAsync(Specification<T> spec)
    {
        return await _ctx.Set<T>().Where(spec.Criteria).CountAsync();
    }
    
    private IQueryable<T> ApplySpec(IQueryable<T> query, Specification<T> spec)
    {
        query = query.Where(spec.Criteria);
        query = spec.Includes.Aggregate(query, (q, i) => q.Include(i));
        
        if (spec.OrderBy != null) query = query.OrderBy(spec.OrderBy);
        else if (spec.OrderByDesc != null) query = query.OrderByDescending(spec.OrderByDesc);
        
        if (spec.Skip.HasValue) query = query.Skip(spec.Skip.Value);
        if (spec.Take.HasValue) query = query.Take(spec.Take.Value);
        
        return query;
    }
}

// Usage
var repo = new SpecificationRepository<Post>(context);
var publishedPosts = await repo.ListAsync(new PublishedPostsSpec(page: 1, pageSize: 10));
var categoryPosts = await repo.ListAsync(new PostsByCategorySpec(categoryId: 2));
```

---

## 📝 สรุป Part 42

| Concept | สิ่งสำคัญ |
|---------|---------|
| Migrations | Version control for DB schema |
| AsNoTracking | Read-only = faster queries |
| Projection (Select) | Load only needed data |
| SplitQuery | Avoid N+1 and cartesian explosion |
| Compiled Query | Reuse query plan |
| Optimistic Concurrency | RowVersion for conflict detection |
| Transaction | Atomic multi-step operations |
| Specification Pattern | Encapsulate query logic |

---

**ก่อนหน้า → [Part 41: EF Core Basics](part41-ef-core-basics.md)**  
**ต่อไป → [Part 43: EF Core Advanced - Unit of Work & DI](part43-ef-unit-of-work.md)**
