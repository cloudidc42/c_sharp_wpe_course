# Part 41: Entity Framework Core - พื้นฐาน
## ขั้นตอนที่ 401-410: ORM และ Code First

---

## 🎯 เป้าหมายของ Part นี้
- EF Core คืออะไร
- Code First: Entity → Database
- DbContext และ DbSet
- Fluent API vs Data Annotations
- Migrations
- CRUD operations
- LINQ to EF Core
- โปรแกรม Blog System

---

## ขั้นตอนที่ 401: ติดตั้ง EF Core

```xml
<!-- .csproj - ใช้ SQLite สำหรับ development -->
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0"/>
<PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="8.0.0"/>
<PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="8.0.0">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
</PackageReference>
<PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.0.0">
    <PrivateAssets>all</PrivateAssets>
</PackageReference>
```

```bash
# EF Core CLI commands
dotnet tool install --global dotnet-ef    # install tool once
dotnet ef migrations add InitialCreate    # create migration
dotnet ef database update                 # apply migration
dotnet ef migrations remove              # remove last migration
dotnet ef database drop                  # drop database
dotnet ef dbcontext info                 # show DbContext info
```

---

## ขั้นตอนที่ 402: Entities (Models)

```csharp
// Entities/Author.cs
public class Author
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public string? Bio { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    // Navigation
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}

// Entities/Post.cs
public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public string Content { get; set; } = "";
    public string Slug { get; set; } = "";
    public bool IsPublished { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? PublishedAt { get; set; }
    
    // Foreign keys
    public int AuthorId { get; set; }
    public int? CategoryId { get; set; }
    
    // Navigation
    public Author Author { get; set; } = null!;
    public Category? Category { get; set; }
    public ICollection<PostTag> PostTags { get; set; } = new List<PostTag>();
    public ICollection<Comment> Comments { get; set; } = new List<Comment>();
}

// Entities/Category.cs
public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string? Description { get; set; }
    
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}

// Entities/Tag.cs - Many-to-Many with Post
public class Tag
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    
    public ICollection<PostTag> PostTags { get; set; } = new List<PostTag>();
}

// Join table for Post <-> Tag many-to-many
public class PostTag
{
    public int PostId { get; set; }
    public int TagId { get; set; }
    
    public Post Post { get; set; } = null!;
    public Tag Tag { get; set; } = null!;
}

// Entities/Comment.cs
public class Comment
{
    public int Id { get; set; }
    public string AuthorName { get; set; } = "";
    public string Content { get; set; } = "";
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public bool IsApproved { get; set; }
    
    public int PostId { get; set; }
    public Post Post { get; set; } = null!;
}
```

---

## ขั้นตอนที่ 403: DbContext

```csharp
// Data/BlogDbContext.cs
using Microsoft.EntityFrameworkCore;

public class BlogDbContext : DbContext
{
    public DbSet<Author> Authors { get; set; }
    public DbSet<Post> Posts { get; set; }
    public DbSet<Category> Categories { get; set; }
    public DbSet<Tag> Tags { get; set; }
    public DbSet<PostTag> PostTags { get; set; }
    public DbSet<Comment> Comments { get; set; }
    
    public BlogDbContext(DbContextOptions<BlogDbContext> options) : base(options) { }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // Apply all configurations from assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(BlogDbContext).Assembly);
        
        // Or configure manually:
        ConfigureAuthor(modelBuilder);
        ConfigurePost(modelBuilder);
        ConfigurePostTag(modelBuilder);
        
        // Seed data
        SeedData(modelBuilder);
    }
    
    private static void ConfigureAuthor(ModelBuilder mb)
    {
        mb.Entity<Author>(e =>
        {
            e.HasKey(a => a.Id);
            e.Property(a => a.Name).IsRequired().HasMaxLength(200);
            e.Property(a => a.Email).IsRequired().HasMaxLength(200);
            e.HasIndex(a => a.Email).IsUnique();
        });
    }
    
    private static void ConfigurePost(ModelBuilder mb)
    {
        mb.Entity<Post>(e =>
        {
            e.HasKey(p => p.Id);
            e.Property(p => p.Title).IsRequired().HasMaxLength(500);
            e.Property(p => p.Slug).IsRequired().HasMaxLength(500);
            e.HasIndex(p => p.Slug).IsUnique();
            
            // Relationship: Post → Author (many-to-one)
            e.HasOne(p => p.Author)
             .WithMany(a => a.Posts)
             .HasForeignKey(p => p.AuthorId)
             .OnDelete(DeleteBehavior.Cascade);
            
            // Relationship: Post → Category (optional many-to-one)
            e.HasOne(p => p.Category)
             .WithMany(c => c.Posts)
             .HasForeignKey(p => p.CategoryId)
             .OnDelete(DeleteBehavior.SetNull)
             .IsRequired(false);
        });
    }
    
    private static void ConfigurePostTag(ModelBuilder mb)
    {
        // Composite key for join table
        mb.Entity<PostTag>(e =>
        {
            e.HasKey(pt => new { pt.PostId, pt.TagId });
            
            e.HasOne(pt => pt.Post)
             .WithMany(p => p.PostTags)
             .HasForeignKey(pt => pt.PostId);
            
            e.HasOne(pt => pt.Tag)
             .WithMany(t => t.PostTags)
             .HasForeignKey(pt => pt.TagId);
        });
    }
    
    private static void SeedData(ModelBuilder mb)
    {
        mb.Entity<Category>().HasData(
            new Category { Id = 1, Name = "เทคโนโลยี", Description = "บทความด้านไอที" },
            new Category { Id = 2, Name = "การเขียนโปรแกรม", Description = "Tutorial และ Guide" },
            new Category { Id = 3, Name = "ทั่วไป" }
        );
        
        mb.Entity<Author>().HasData(
            new Author { Id = 1, Name = "สมชาย นักพัฒนา", Email = "somchai@blog.com", 
                         CreatedAt = DateTime.UtcNow }
        );
    }
    
    // Auto-set timestamps
    public override int SaveChanges()
    {
        UpdateTimestamps();
        return base.SaveChanges();
    }
    
    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        UpdateTimestamps();
        return base.SaveChangesAsync(ct);
    }
    
    private void UpdateTimestamps()
    {
        foreach (var entry in ChangeTracker.Entries())
        {
            if (entry.State == EntityState.Added && entry.Entity is IHasCreatedAt created)
                created.CreatedAt = DateTime.UtcNow;
        }
    }
}

// Interface for auto-timestamp
public interface IHasCreatedAt
{
    DateTime CreatedAt { get; set; }
}
```

---

## ขั้นตอนที่ 404: DbContext Factory (Desktop App)

```csharp
// Data/BlogDbContextFactory.cs - for design-time migrations
using Microsoft.EntityFrameworkCore.Design;

public class BlogDbContextFactory : IDesignTimeDbContextFactory<BlogDbContext>
{
    public BlogDbContext CreateDbContext(string[] args)
    {
        var opt = new DbContextOptionsBuilder<BlogDbContext>()
            .UseSqlite("Data Source=blog.db")
            .Options;
        return new BlogDbContext(opt);
    }
}

// Program.cs (Console App / Desktop)
var dbPath = Path.Combine(
    Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
    "BlogApp", "blog.db");

Directory.CreateDirectory(Path.GetDirectoryName(dbPath)!);

var options = new DbContextOptionsBuilder<BlogDbContext>()
    .UseSqlite($"Data Source={dbPath}")
    .EnableSensitiveDataLogging()  // Dev only
    .LogTo(Console.WriteLine, LogLevel.Information)  // Dev only
    .Options;

using var context = new BlogDbContext(options);
await context.Database.MigrateAsync(); // Auto-apply migrations
```

---

## ขั้นตอนที่ 405: CRUD Operations

```csharp
// CREATE
var post = new Post
{
    Title = "เริ่มต้น EF Core",
    Content = "EF Core เป็น ORM ยอดนิยมสำหรับ .NET...",
    Slug = "getting-started-ef-core",
    AuthorId = 1,
    IsPublished = true,
    PublishedAt = DateTime.UtcNow
};

await context.Posts.AddAsync(post);
await context.SaveChangesAsync();
Console.WriteLine($"Created post ID: {post.Id}"); // Auto-set after save

// CREATE RANGE
var tags = new[] { "C#", "EF Core", ".NET 8" }.Select(name => new Tag { Name = name });
await context.Tags.AddRangeAsync(tags);
await context.SaveChangesAsync();

// READ - Single
var foundPost = await context.Posts.FindAsync(1); // By PK (fastest)
var bySlug = await context.Posts.FirstOrDefaultAsync(p => p.Slug == "getting-started-ef-core");
var singlePost = await context.Posts.SingleOrDefaultAsync(p => p.Id == 99); // Throws if >1

// READ - With includes (Eager Loading)
var postWithAuthor = await context.Posts
    .Include(p => p.Author)
    .Include(p => p.Category)
    .Include(p => p.PostTags)
        .ThenInclude(pt => pt.Tag)
    .Include(p => p.Comments.Where(c => c.IsApproved))
    .Where(p => p.IsPublished)
    .OrderByDescending(p => p.PublishedAt)
    .FirstOrDefaultAsync();

// UPDATE
var postToUpdate = await context.Posts.FindAsync(1);
if (postToUpdate != null)
{
    postToUpdate.Title = "Updated Title";
    postToUpdate.Content = "Updated content...";
    await context.SaveChangesAsync(); // EF tracks changes automatically
}

// UPDATE without loading (ExecuteUpdate - EF 7+)
await context.Posts
    .Where(p => p.AuthorId == 1)
    .ExecuteUpdateAsync(s => s
        .SetProperty(p => p.IsPublished, true)
        .SetProperty(p => p.PublishedAt, DateTime.UtcNow));

// DELETE
var postToDelete = await context.Posts.FindAsync(1);
if (postToDelete != null)
{
    context.Posts.Remove(postToDelete);
    await context.SaveChangesAsync();
}

// DELETE without loading (ExecuteDelete - EF 7+)
await context.Comments
    .Where(c => !c.IsApproved && c.CreatedAt < DateTime.UtcNow.AddDays(-30))
    .ExecuteDeleteAsync();
```

---

## ขั้นตอนที่ 406: LINQ Queries

```csharp
// Filtering
var published = await context.Posts
    .Where(p => p.IsPublished && p.CategoryId == 2)
    .ToListAsync();

// Projection (Select specific columns - more efficient)
var titles = await context.Posts
    .Where(p => p.IsPublished)
    .Select(p => new { p.Id, p.Title, p.Slug, AuthorName = p.Author.Name })
    .ToListAsync();

// Ordering
var latest = await context.Posts
    .OrderByDescending(p => p.PublishedAt)
    .ThenBy(p => p.Title)
    .Take(10)
    .ToListAsync();

// Pagination
int page = 1, pageSize = 10;
var paged = await context.Posts
    .Where(p => p.IsPublished)
    .OrderByDescending(p => p.PublishedAt)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();

// Count
int totalPosts = await context.Posts.CountAsync(p => p.IsPublished);
bool hasAny = await context.Posts.AnyAsync(p => p.AuthorId == 1);

// Aggregations
decimal? avgComments = await context.Posts
    .AverageAsync(p => p.Comments.Count);

var categoryStats = await context.Posts
    .GroupBy(p => p.Category!.Name)
    .Select(g => new 
    { 
        Category = g.Key, 
        Count = g.Count(),
        Latest = g.Max(p => p.PublishedAt)
    })
    .ToListAsync();

// Full-text search (SQLite LIKE)
string search = "EF Core";
var searchResults = await context.Posts
    .Where(p => EF.Functions.Like(p.Title, $"%{search}%") || 
                EF.Functions.Like(p.Content, $"%{search}%"))
    .ToListAsync();

// Raw SQL (when LINQ is not enough)
var posts = await context.Posts
    .FromSqlRaw("SELECT * FROM Posts WHERE Id > {0}", 5)
    .ToListAsync();
```

---

## ขั้นตอนที่ 407: Repository Pattern กับ EF Core

```csharp
// Data/IRepository.cs
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<List<T>> GetAllAsync();
    Task<T> AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
}

// Data/PostRepository.cs
public class PostRepository : IRepository<Post>
{
    private readonly BlogDbContext _ctx;
    
    public PostRepository(BlogDbContext ctx)
    {
        _ctx = ctx;
    }
    
    public async Task<Post?> GetByIdAsync(int id)
        => await _ctx.Posts
            .Include(p => p.Author)
            .Include(p => p.Category)
            .Include(p => p.PostTags).ThenInclude(pt => pt.Tag)
            .FirstOrDefaultAsync(p => p.Id == id);
    
    public async Task<List<Post>> GetAllAsync()
        => await _ctx.Posts
            .Include(p => p.Author)
            .Include(p => p.Category)
            .OrderByDescending(p => p.CreatedAt)
            .ToListAsync();
    
    public async Task<List<Post>> GetPublishedAsync(int page = 1, int size = 10)
        => await _ctx.Posts
            .Where(p => p.IsPublished)
            .Include(p => p.Author)
            .OrderByDescending(p => p.PublishedAt)
            .Skip((page - 1) * size)
            .Take(size)
            .ToListAsync();
    
    public async Task<Post> AddAsync(Post post)
    {
        await _ctx.Posts.AddAsync(post);
        await _ctx.SaveChangesAsync();
        return post;
    }
    
    public async Task UpdateAsync(Post post)
    {
        _ctx.Posts.Update(post);
        await _ctx.SaveChangesAsync();
    }
    
    public async Task DeleteAsync(int id)
    {
        var post = await _ctx.Posts.FindAsync(id);
        if (post != null)
        {
            _ctx.Posts.Remove(post);
            await _ctx.SaveChangesAsync();
        }
    }
}
```

---

## ขั้นตอนที่ 408-410: Blog System Console Demo

```csharp
// Program.cs - Blog System Demo
var options = new DbContextOptionsBuilder<BlogDbContext>()
    .UseSqlite("Data Source=blog.db")
    .Options;

await using var context = new BlogDbContext(options);
await context.Database.EnsureCreatedAsync();

var repo = new PostRepository(context);

// สร้างโพสต์
var newPost = await repo.AddAsync(new Post
{
    Title = "WPF กับ MVVM Pattern",
    Content = "การใช้ MVVM ใน WPF ทำให้โค้ดสะอาดและ testable...",
    Slug = "wpf-mvvm-pattern",
    AuthorId = 1,
    CategoryId = 2,
    IsPublished = true,
    PublishedAt = DateTime.UtcNow
});

Console.WriteLine($"สร้างโพสต์ ID: {newPost.Id}");

// ดึงโพสต์ทั้งหมด
var posts = await context.Posts
    .Include(p => p.Author)
    .Include(p => p.Category)
    .OrderByDescending(p => p.CreatedAt)
    .ToListAsync();

Console.WriteLine($"\nโพสต์ทั้งหมด ({posts.Count} รายการ):");
foreach (var post in posts)
{
    Console.WriteLine($"  [{post.Id}] {post.Title}");
    Console.WriteLine($"       โดย: {post.Author.Name} | หมวด: {post.Category?.Name ?? "ไม่มีหมวด"}");
    Console.WriteLine($"       สถานะ: {(post.IsPublished ? "เผยแพร่แล้ว" : "ฉบับร่าง")}");
}

// สถิติ
var stats = await context.Posts
    .GroupBy(p => p.Category!.Name)
    .Select(g => new { Category = g.Key ?? "ไม่มีหมวด", Count = g.Count() })
    .ToListAsync();

Console.WriteLine("\nสถิติตามหมวดหมู่:");
foreach (var stat in stats)
    Console.WriteLine($"  {stat.Category}: {stat.Count} โพสต์");
```

---

## 📝 สรุป Part 41

| Concept | สิ่งสำคัญ |
|---------|---------|
| Code First | Entity class → auto-generate DB |
| DbContext | Unit of Work + gateway to DB |
| DbSet<T> | แต่ละ table |
| Migrations | Version control สำหรับ DB schema |
| Fluent API | Configure relationships ใน OnModelCreating |
| Include | Eager loading related entities |
| ExecuteUpdate/Delete | Bulk operations ไม่ต้อง load |
| Repository Pattern | Abstract data access layer |

---

**ก่อนหน้า → [Part 40: WPF Final Project](../part31-40/part40-wpf-final.md)**  
**ต่อไป → [Part 42: EF Core Migrations & Advanced Queries](part42-ef-migrations.md)**
