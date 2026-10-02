# Part 69: GraphQL with Hot Chocolate
## ขั้นตอนที่ 681-690: สร้าง GraphQL API ด้วย Hot Chocolate ใน .NET

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจความแตกต่างระหว่าง GraphQL กับ REST
- ติดตั้งและตั้งค่า Hot Chocolate สำหรับ .NET
- สร้าง Query Types, Mutation Types และ Subscription Types
- แก้ปัญหา N+1 ด้วย DataLoader
- ใช้ Filtering, Sorting และ Pagination
- จัดการ Authentication/Authorization ใน GraphQL
- สร้าง E-Commerce API แบบครบวงจร

---

## ขั้นตอนที่ 681: GraphQL Overview vs REST และการติดตั้ง Hot Chocolate

### GraphQL คืออะไร และต่างจาก REST อย่างไร?

**REST API** เป็นแนวทางดั้งเดิมที่ออกแบบ Endpoint แยกตาม Resource แต่มีข้อจำกัดหลายอย่าง:

```
REST Architecture:
GET  /products          → ดึง Products ทั้งหมด (อาจได้ข้อมูลมากเกินไป = Over-fetching)
GET  /products/1        → ดึง Product ตาม ID
GET  /products/1/orders → ดึง Orders ของ Product นั้น
GET  /users/1           → ดึงข้อมูล User (ต้องเรียกแยก = Under-fetching)

ปัญหา:
- Over-fetching: ได้ข้อมูลมากกว่าที่ต้องการ
- Under-fetching: ต้องเรียก API หลายครั้งเพื่อข้อมูลครบ
- Version Management: v1, v2, v3 ทำให้ซับซ้อน
```

**GraphQL** แก้ปัญหาเหล่านี้ด้วยการให้ Client ระบุเองว่าต้องการข้อมูลอะไร:

```graphql
# GraphQL Query - ดึงเฉพาะที่ต้องการ
query {
  product(id: 1) {
    name
    price
    orders {
      id
      total
      user {
        name
        email
      }
    }
  }
}
```

### เปรียบเทียบ REST vs GraphQL

```
┌─────────────────────┬──────────────────────┬──────────────────────┐
│ Feature             │ REST                 │ GraphQL              │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Endpoint            │ หลาย Endpoint         │ Endpoint เดียว       │
│ Data Fetching       │ Fixed Structure      │ Client-driven        │
│ Over/Under-fetching │ มีปัญหา               │ ไม่มีปัญหา            │
│ Versioning          │ ต้องทำ v1, v2        │ ไม่จำเป็น            │
│ Type System         │ ไม่มีมาตรฐาน          │ Strongly Typed       │
│ Real-time           │ WebSocket/SSE แยก    │ Subscription built-in│
│ Learning Curve      │ ง่าย                 │ สูงกว่าเล็กน้อย       │
└─────────────────────┴──────────────────────┴──────────────────────┘
```

### ติดตั้ง Hot Chocolate

**Hot Chocolate** คือ GraphQL server library สำหรับ .NET ที่ได้รับความนิยมสูงสุด

```bash
# สร้างโปรเจกต์ใหม่
dotnet new webapi -n ECommerceGraphQL
cd ECommerceGraphQL

# ติดตั้ง Hot Chocolate packages
dotnet add package HotChocolate.AspNetCore
dotnet add package HotChocolate.Data
dotnet add package HotChocolate.Data.EntityFramework
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

### โครงสร้างโปรเจกต์

```
ECommerceGraphQL/
├── Data/
│   ├── AppDbContext.cs
│   └── SeedData.cs
├── Models/
│   ├── Product.cs
│   ├── Order.cs
│   ├── OrderItem.cs
│   └── User.cs
├── GraphQL/
│   ├── Types/
│   │   ├── ProductType.cs
│   │   ├── OrderType.cs
│   │   └── UserType.cs
│   ├── Queries/
│   │   └── Query.cs
│   ├── Mutations/
│   │   └── Mutation.cs
│   ├── Subscriptions/
│   │   └── Subscription.cs
│   └── DataLoaders/
│       ├── ProductByIdDataLoader.cs
│       └── UserByIdDataLoader.cs
└── Program.cs
```

### Models

```csharp
// Models/User.cs
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = default!;
    public string Email { get; set; } = default!;
    public string PasswordHash { get; set; } = default!;
    public string Role { get; set; } = "Customer";
    public List<Order> Orders { get; set; } = new();
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}

// Models/Product.cs
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = default!;
    public string Description { get; set; } = default!;
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public string Category { get; set; } = default!;
    public string ImageUrl { get; set; } = default!;
    public bool IsActive { get; set; } = true;
    public List<OrderItem> OrderItems { get; set; } = new();
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}

// Models/Order.cs
public class Order
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public User User { get; set; } = default!;
    public List<OrderItem> Items { get; set; } = new();
    public decimal Total { get; set; }
    public string Status { get; set; } = "Pending";
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}

// Models/OrderItem.cs
public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public Order Order { get; set; } = default!;
    public int ProductId { get; set; }
    public Product Product { get; set; } = default!;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}
```

### DbContext และการตั้งค่า

```csharp
// Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<User> Users => Set<User>();
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>()
            .Property(p => p.Price)
            .HasPrecision(18, 2);

        modelBuilder.Entity<Order>()
            .Property(o => o.Total)
            .HasPrecision(18, 2);

        modelBuilder.Entity<OrderItem>()
            .Property(oi => oi.UnitPrice)
            .HasPrecision(18, 2);
    }
}
```

### Program.cs - ตั้งค่า Hot Chocolate

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// เพิ่ม DbContext
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// ตั้งค่า GraphQL ด้วย Hot Chocolate
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddSubscriptionType<Subscription>()
    .AddType<ProductType>()
    .AddType<OrderType>()
    .AddType<UserType>()
    .AddFiltering()
    .AddSorting()
    .AddProjections()
    .AddInMemorySubscriptions(); // สำหรับ Development

var app = builder.Build();

app.UseWebSockets(); // จำเป็นสำหรับ Subscriptions
app.MapGraphQL();   // Default: /graphql

app.Run();
```

---

## ขั้นตอนที่ 682: Query Types และ Resolvers

### Query Type คืออะไร?

Query Type เป็นจุดเริ่มต้นของการอ่านข้อมูลใน GraphQL เปรียบได้กับ GET endpoints ใน REST

```csharp
// GraphQL/Queries/Query.cs
using HotChocolate.Data;

public class Query
{
    // Resolver แบบพื้นฐาน
    [UseDbContext(typeof(AppDbContext))]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    public IQueryable<Product> GetProducts([ScopedService] AppDbContext context)
        => context.Products.Where(p => p.IsActive);

    // Resolver แบบมี ID
    [UseDbContext(typeof(AppDbContext))]
    [UseProjection]
    public Task<Product?> GetProductAsync(
        int id,
        [ScopedService] AppDbContext context,
        CancellationToken cancellationToken)
        => context.Products.FindAsync(new object[] { id }, cancellationToken).AsTask();

    // Resolver ด้วย DataLoader (ดีกว่า)
    public Task<Product?> GetProductByIdAsync(
        int id,
        ProductByIdDataLoader dataLoader,
        CancellationToken cancellationToken)
        => dataLoader.LoadAsync(id, cancellationToken);

    [UseDbContext(typeof(AppDbContext))]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    public IQueryable<Order> GetOrders([ScopedService] AppDbContext context)
        => context.Orders;

    [UseDbContext(typeof(AppDbContext))]
    [UseProjection]
    public IQueryable<User> GetUsers([ScopedService] AppDbContext context)
        => context.Users;
}
```

### Object Types การกำหนดโครงสร้าง Type

```csharp
// GraphQL/Types/ProductType.cs
public class ProductType : ObjectType<Product>
{
    protected override void Configure(IObjectTypeDescriptor<Product> descriptor)
    {
        descriptor.Description("สินค้าใน E-Commerce");

        descriptor
            .Field(p => p.Id)
            .Description("รหัสสินค้า");

        descriptor
            .Field(p => p.Name)
            .Description("ชื่อสินค้า");

        descriptor
            .Field(p => p.Price)
            .Description("ราคาสินค้า (บาท)");

        // ซ่อน Field ที่ไม่ต้องการแสดง
        descriptor
            .Field(p => p.OrderItems)
            .Ignore();

        // เพิ่ม Computed Field
        descriptor
            .Field("isInStock")
            .Description("สินค้ามีในสต็อกหรือไม่")
            .Resolve(ctx => ctx.Parent<Product>().Stock > 0);

        // Field ที่ต้องการ Authorization
        descriptor
            .Field(p => p.CreatedAt)
            .Authorize("Admin");
    }
}
```

### การใช้งาน Schema-first กับ Annotation-based

```csharp
// GraphQL/Types/UserType.cs - แบบ Annotation-based
[GraphQLDescription("ผู้ใช้งานระบบ")]
public class UserType : ObjectType<User>
{
    protected override void Configure(IObjectTypeDescriptor<User> descriptor)
    {
        descriptor
            .Field(u => u.PasswordHash)
            .Ignore(); // ไม่แสดง Password Hash

        descriptor
            .Field(u => u.Orders)
            .UseDbContext<AppDbContext>()
            .ResolveWith<UserResolvers>(r => r.GetOrdersAsync(default!, default!, default!))
            .UseFiltering()
            .UseSorting();
    }
}

// Resolver Class แยก
public class UserResolvers
{
    public async Task<IEnumerable<Order>> GetOrdersAsync(
        [Parent] User user,
        [ScopedService] AppDbContext context,
        CancellationToken cancellationToken)
    {
        return await context.Orders
            .Where(o => o.UserId == user.Id)
            .ToListAsync(cancellationToken);
    }
}
```

### การเรียกใช้ GraphQL Query

```graphql
# ตัวอย่าง Query จาก Client
query GetProducts {
  products(
    where: { category: { eq: "Electronics" } }
    order: { price: ASC }
  ) {
    id
    name
    price
    category
    isInStock
  }
}

query GetProductDetail {
  productById(id: 1) {
    id
    name
    price
    description
    stock
  }
}

# Query ด้วย Fragment
fragment ProductFields on Product {
  id
  name
  price
}

query GetMultipleProducts {
  product1: productById(id: 1) {
    ...ProductFields
  }
  product2: productById(id: 2) {
    ...ProductFields
  }
}
```

---

## ขั้นตอนที่ 683: Mutations

### Mutation Types สำหรับการเปลี่ยนแปลงข้อมูล

Mutation ใน GraphQL เปรียบได้กับ POST, PUT, DELETE ใน REST

```csharp
// GraphQL/Mutations/Mutation.cs
public class Mutation
{
    [UseDbContext(typeof(AppDbContext))]
    public async Task<AddProductPayload> AddProductAsync(
        AddProductInput input,
        [ScopedService] AppDbContext context,
        CancellationToken cancellationToken)
    {
        var product = new Product
        {
            Name = input.Name,
            Description = input.Description,
            Price = input.Price,
            Stock = input.Stock,
            Category = input.Category,
            ImageUrl = input.ImageUrl ?? string.Empty
        };

        context.Products.Add(product);
        await context.SaveChangesAsync(cancellationToken);

        return new AddProductPayload(product);
    }

    [UseDbContext(typeof(AppDbContext))]
    public async Task<UpdateProductPayload> UpdateProductAsync(
        UpdateProductInput input,
        [ScopedService] AppDbContext context,
        CancellationToken cancellationToken)
    {
        var product = await context.Products.FindAsync(
            new object[] { input.Id }, cancellationToken);

        if (product is null)
            return new UpdateProductPayload(
                new UserError("สินค้าไม่พบ", "PRODUCT_NOT_FOUND"));

        if (input.Name is not null) product.Name = input.Name;
        if (input.Price is not null) product.Price = input.Price.Value;
        if (input.Stock is not null) product.Stock = input.Stock.Value;

        await context.SaveChangesAsync(cancellationToken);

        return new UpdateProductPayload(product);
    }

    [UseDbContext(typeof(AppDbContext))]
    public async Task<DeleteProductPayload> DeleteProductAsync(
        int id,
        [ScopedService] AppDbContext context,
        CancellationToken cancellationToken)
    {
        var product = await context.Products.FindAsync(
            new object[] { id }, cancellationToken);

        if (product is null)
            return new DeleteProductPayload(false, "สินค้าไม่พบ");

        product.IsActive = false; // Soft delete
        await context.SaveChangesAsync(cancellationToken);

        return new DeleteProductPayload(true, "ลบสินค้าเรียบร้อยแล้ว");
    }

    [UseDbContext(typeof(AppDbContext))]
    public async Task<CreateOrderPayload> CreateOrderAsync(
        CreateOrderInput input,
        [ScopedService] AppDbContext context,
        [Service] ITopicEventSender eventSender,
        CancellationToken cancellationToken)
    {
        // คำนวณราคา
        var productIds = input.Items.Select(i => i.ProductId).ToList();
        var products = await context.Products
            .Where(p => productIds.Contains(p.Id))
            .ToDictionaryAsync(p => p.Id, cancellationToken);

        var order = new Order
        {
            UserId = input.UserId,
            Items = input.Items.Select(item => new OrderItem
            {
                ProductId = item.ProductId,
                Quantity = item.Quantity,
                UnitPrice = products[item.ProductId].Price
            }).ToList()
        };

        order.Total = order.Items.Sum(i => i.Quantity * i.UnitPrice);

        context.Orders.Add(order);
        await context.SaveChangesAsync(cancellationToken);

        // ส่ง Event สำหรับ Subscription
        await eventSender.SendAsync(
            nameof(Subscription.OnOrderCreated),
            order,
            cancellationToken);

        return new CreateOrderPayload(order);
    }
}
```

### Input Types และ Payload Types

```csharp
// Input Types
public record AddProductInput(
    string Name,
    string Description,
    decimal Price,
    int Stock,
    string Category,
    string? ImageUrl);

public record UpdateProductInput(
    int Id,
    string? Name,
    decimal? Price,
    int? Stock);

public record CreateOrderInput(
    int UserId,
    List<OrderItemInput> Items);

public record OrderItemInput(
    int ProductId,
    int Quantity);

// Payload Types (Response)
public class AddProductPayload
{
    public AddProductPayload(Product product)
        => Product = product;

    public Product? Product { get; }
}

public class UpdateProductPayload
{
    public UpdateProductPayload(Product product)
        => Product = product;

    public UpdateProductPayload(UserError error)
        => Errors = new[] { error };

    public Product? Product { get; }
    public IReadOnlyList<UserError>? Errors { get; }
}

public class DeleteProductPayload
{
    public DeleteProductPayload(bool success, string message)
    {
        Success = success;
        Message = message;
    }

    public bool Success { get; }
    public string Message { get; }
}

public class CreateOrderPayload
{
    public CreateOrderPayload(Order order) => Order = order;
    public Order? Order { get; }
}
```

### การเรียก Mutation จาก Client

```graphql
# เพิ่มสินค้าใหม่
mutation AddProduct {
  addProduct(input: {
    name: "iPhone 15 Pro"
    description: "สมาร์ทโฟนรุ่นล่าสุดจาก Apple"
    price: 39900
    stock: 50
    category: "Electronics"
    imageUrl: "https://example.com/iphone15.jpg"
  }) {
    product {
      id
      name
      price
    }
  }
}

# อัพเดทสินค้า
mutation UpdateProduct {
  updateProduct(input: {
    id: 1
    price: 37900
    stock: 45
  }) {
    product {
      id
      name
      price
      stock
    }
    errors {
      message
      code
    }
  }
}

# สร้าง Order
mutation CreateOrder {
  createOrder(input: {
    userId: 1
    items: [
      { productId: 1, quantity: 2 }
      { productId: 3, quantity: 1 }
    ]
  }) {
    order {
      id
      total
      status
    }
  }
}
```

---

## ขั้นตอนที่ 684: Subscriptions (Real-time ด้วย WebSocket)

### Subscription คืออะไร?

Subscription ทำให้ Client รับข้อมูล Real-time ผ่าน WebSocket เมื่อมี Event เกิดขึ้น

```csharp
// GraphQL/Subscriptions/Subscription.cs
public class Subscription
{
    // Subscribe เมื่อมี Order ใหม่
    [Subscribe]
    [Topic]
    public Order OnOrderCreated([EventMessage] Order order)
        => order;

    // Subscribe เมื่อ Order status เปลี่ยน
    [Subscribe]
    [Topic("{orderId}")]
    public Order OnOrderStatusChanged(
        int orderId,
        [EventMessage] Order order)
        => order;

    // Subscribe การเปลี่ยนแปลงราคาสินค้า
    [Subscribe]
    [Topic("{category}")]
    public Product OnProductPriceChanged(
        string category,
        [EventMessage] Product product)
        => product;
}
```

### การส่ง Subscription Events จาก Mutation

```csharp
// ใน Mutation - ส่ง Event เมื่อ Order status เปลี่ยน
[UseDbContext(typeof(AppDbContext))]
public async Task<UpdateOrderStatusPayload> UpdateOrderStatusAsync(
    int orderId,
    string newStatus,
    [ScopedService] AppDbContext context,
    [Service] ITopicEventSender eventSender,
    CancellationToken cancellationToken)
{
    var order = await context.Orders
        .Include(o => o.Items)
        .FirstOrDefaultAsync(o => o.Id == orderId, cancellationToken);

    if (order is null)
        return new UpdateOrderStatusPayload(null, "Order ไม่พบ");

    order.Status = newStatus;
    await context.SaveChangesAsync(cancellationToken);

    // ส่ง Event ไปยัง Subscriber ทุกคนที่ดู Order นี้
    await eventSender.SendAsync(
        $"OnOrderStatusChanged_{orderId}",
        order,
        cancellationToken);

    return new UpdateOrderStatusPayload(order, null);
}
```

### การตั้งค่า In-Memory และ Redis Subscriptions

```csharp
// Program.cs - เลือก Subscription Provider
builder.Services
    .AddGraphQLServer()
    // ...
    
    // สำหรับ Development (Single Server)
    .AddInMemorySubscriptions()
    
    // หรือสำหรับ Production (Multiple Servers)
    // .AddRedisSubscriptions(r => r.Configuration = "localhost:6379")
    ;

// เพิ่ม WebSocket support
app.UseWebSockets();
app.MapGraphQL();
```

### การใช้งาน Subscription จาก Client

```graphql
# Subscribe รอ Order ใหม่
subscription WaitForNewOrder {
  onOrderCreated {
    id
    total
    status
    user {
      name
    }
    items {
      product {
        name
      }
      quantity
    }
  }
}

# Subscribe ติดตามสถานะ Order เฉพาะ
subscription TrackOrder {
  onOrderStatusChanged(orderId: 42) {
    id
    status
    updatedAt
  }
}
```

### Custom Subscription Service

```csharp
// Services/OrderNotificationService.cs
public class OrderNotificationService
{
    private readonly ITopicEventSender _eventSender;

    public OrderNotificationService(ITopicEventSender eventSender)
        => _eventSender = eventSender;

    public async Task NotifyOrderStatusChangedAsync(
        Order order,
        CancellationToken ct = default)
    {
        // แจ้ง Subscriber ที่ดู Order ID นี้
        await _eventSender.SendAsync(
            $"OnOrderStatusChanged_{order.Id}",
            order, ct);

        // แจ้ง Global Subscriber ด้วย
        await _eventSender.SendAsync(
            nameof(Subscription.OnOrderCreated),
            order, ct);
    }
}
```

---

## ขั้นตอนที่ 685: Filtering, Sorting และ Pagination

### การตั้งค่า Filtering และ Sorting

Hot Chocolate มี built-in support สำหรับ Filtering และ Sorting ผ่าน Attributes

```csharp
// ลงทะเบียนใน Program.cs
builder.Services
    .AddGraphQLServer()
    .AddFiltering()
    .AddSorting()
    .AddProjections();

// Query ที่รองรับ Filtering, Sorting, Pagination
public class Query
{
    [UseDbContext(typeof(AppDbContext))]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    [UsePaging(IncludeTotalCount = true, DefaultPageSize = 10, MaxPageSize = 50)]
    public IQueryable<Product> GetProducts([ScopedService] AppDbContext context)
        => context.Products.Where(p => p.IsActive);
}
```

### Custom Filter Types

```csharp
// GraphQL/Filters/ProductFilterType.cs
public class ProductFilterType : FilterInputType<Product>
{
    protected override void Configure(IFilterInputTypeDescriptor<Product> descriptor)
    {
        descriptor.BindFieldsExplicitly();

        descriptor.Field(p => p.Name)
            .Name("name");

        descriptor.Field(p => p.Price)
            .Name("price");

        descriptor.Field(p => p.Category)
            .Name("category");

        descriptor.Field(p => p.Stock)
            .Name("stock");

        // Custom Filter: ราคาอยู่ในช่วง
        descriptor.Field("priceRange")
            .Type<PriceRangeFilterInputType>();
    }
}

// ใช้ Custom Filter ใน Query
public IQueryable<Product> GetProducts(
    [ScopedService] AppDbContext context,
    [GraphQLType(typeof(ProductFilterType))] ProductFilter? where)
{
    var query = context.Products.Where(p => p.IsActive);

    if (where?.PriceMin.HasValue == true)
        query = query.Where(p => p.Price >= where.PriceMin.Value);

    if (where?.PriceMax.HasValue == true)
        query = query.Where(p => p.Price <= where.PriceMax.Value);

    return query;
}
```

### Cursor-based Pagination

```csharp
// Cursor Pagination (แนะนำ)
[UsePaging(IncludeTotalCount = true)]
public IQueryable<Product> GetProducts([ScopedService] AppDbContext context)
    => context.Products.OrderBy(p => p.Id);

// Offset Pagination (สำหรับกรณีที่ต้องการ)
[UseOffsetPaging(IncludeTotalCount = true)]
public IQueryable<Product> GetProductsOffset([ScopedService] AppDbContext context)
    => context.Products;
```

### การใช้ Filtering และ Sorting จาก Client

```graphql
# Filter Products
query FilterProducts {
  products(
    where: {
      and: [
        { category: { eq: "Electronics" } }
        { price: { lte: 50000 } }
        { stock: { gt: 0 } }
      ]
    }
    order: [
      { price: ASC }
      { name: ASC }
    ]
    first: 10
  ) {
    totalCount
    pageInfo {
      hasNextPage
      endCursor
    }
    nodes {
      id
      name
      price
      category
    }
  }
}

# Pagination ไปหน้าถัดไป
query NextPage {
  products(
    first: 10
    after: "cursor_value_from_previous_query"
  ) {
    pageInfo {
      hasNextPage
      endCursor
    }
    nodes {
      id
      name
    }
  }
}
```

### Sorting แบบ Custom

```csharp
// Custom Sort Type
public class ProductSortType : SortInputType<Product>
{
    protected override void Configure(ISortInputTypeDescriptor<Product> descriptor)
    {
        descriptor.BindFieldsExplicitly();
        descriptor.Field(p => p.Name).Name("name");
        descriptor.Field(p => p.Price).Name("price");
        descriptor.Field(p => p.CreatedAt).Name("createdAt");
        // Custom: เรียงตามความนิยม (จาก OrderItems count)
    }
}
```

---

## ขั้นตอนที่ 686: DataLoader สำหรับแก้ปัญหา N+1

### N+1 Problem คืออะไร?

```
ปัญหา N+1:
Query ดึง Orders 10 รายการ → 1 Query
สำหรับแต่ละ Order ดึง User → 10 Queries เพิ่มเติม!
รวม = 11 Queries (1 + N)

ยิ่ง N มาก ยิ่งช้า และกด Database หนัก
```

### DataLoader การแก้ปัญหา N+1

```csharp
// GraphQL/DataLoaders/UserByIdDataLoader.cs
public class UserByIdDataLoader : BatchDataLoader<int, User>
{
    private readonly IDbContextFactory<AppDbContext> _dbContextFactory;

    public UserByIdDataLoader(
        IDbContextFactory<AppDbContext> dbContextFactory,
        IBatchScheduler batchScheduler,
        DataLoaderOptions? options = null)
        : base(batchScheduler, options)
    {
        _dbContextFactory = dbContextFactory;
    }

    protected override async Task<IReadOnlyDictionary<int, User>> LoadBatchAsync(
        IReadOnlyList<int> keys,
        CancellationToken cancellationToken)
    {
        // Query ครั้งเดียวสำหรับทุก User ID ที่ต้องการ
        await using var dbContext = await _dbContextFactory.CreateDbContextAsync(cancellationToken);

        return await dbContext.Users
            .Where(u => keys.Contains(u.Id))
            .ToDictionaryAsync(u => u.Id, cancellationToken);
    }
}
```

```csharp
// GraphQL/DataLoaders/ProductByIdDataLoader.cs
public class ProductByIdDataLoader : BatchDataLoader<int, Product>
{
    private readonly IDbContextFactory<AppDbContext> _dbContextFactory;

    public ProductByIdDataLoader(
        IDbContextFactory<AppDbContext> dbContextFactory,
        IBatchScheduler batchScheduler,
        DataLoaderOptions? options = null)
        : base(batchScheduler, options)
    {
        _dbContextFactory = dbContextFactory;
    }

    protected override async Task<IReadOnlyDictionary<int, Product>> LoadBatchAsync(
        IReadOnlyList<int> keys,
        CancellationToken cancellationToken)
    {
        await using var dbContext = await _dbContextFactory.CreateDbContextAsync(cancellationToken);

        return await dbContext.Products
            .Where(p => keys.Contains(p.Id))
            .ToDictionaryAsync(p => p.Id, cancellationToken);
    }
}
```

### Group DataLoader - โหลดข้อมูลแบบ 1-to-Many

```csharp
// GraphQL/DataLoaders/OrdersByUserIdDataLoader.cs
public class OrdersByUserIdDataLoader : GroupedDataLoader<int, Order>
{
    private readonly IDbContextFactory<AppDbContext> _dbContextFactory;

    public OrdersByUserIdDataLoader(
        IDbContextFactory<AppDbContext> dbContextFactory,
        IBatchScheduler batchScheduler,
        DataLoaderOptions? options = null)
        : base(batchScheduler, options)
    {
        _dbContextFactory = dbContextFactory;
    }

    protected override async Task<ILookup<int, Order>> LoadGroupedBatchAsync(
        IReadOnlyList<int> keys,
        CancellationToken cancellationToken)
    {
        await using var dbContext = await _dbContextFactory.CreateDbContextAsync(cancellationToken);

        var orders = await dbContext.Orders
            .Where(o => keys.Contains(o.UserId))
            .Include(o => o.Items)
            .ToListAsync(cancellationToken);

        return orders.ToLookup(o => o.UserId);
    }
}
```

### ใช้ DataLoader ใน Resolver

```csharp
// GraphQL/Types/OrderType.cs
public class OrderType : ObjectType<Order>
{
    protected override void Configure(IObjectTypeDescriptor<Order> descriptor)
    {
        // ใช้ DataLoader แทนการ Load ตรง
        descriptor
            .Field(o => o.User)
            .ResolveWith<OrderResolvers>(r => r.GetUserAsync(default!, default!, default!));

        descriptor
            .Field(o => o.Items)
            .ResolveWith<OrderResolvers>(r => r.GetItemsAsync(default!, default!, default!));
    }
}

public class OrderResolvers
{
    // ใช้ UserByIdDataLoader - Batch Loading
    public Task<User> GetUserAsync(
        [Parent] Order order,
        UserByIdDataLoader dataLoader,
        CancellationToken cancellationToken)
        => dataLoader.LoadAsync(order.UserId, cancellationToken)!;

    public async Task<IEnumerable<OrderItem>> GetItemsAsync(
        [Parent] Order order,
        [ScopedService] AppDbContext context,
        CancellationToken cancellationToken)
    {
        return await context.OrderItems
            .Where(oi => oi.OrderId == order.Id)
            .Include(oi => oi.Product)
            .ToListAsync(cancellationToken);
    }
}
```

### ลงทะเบียน DataLoader และ DbContextFactory

```csharp
// Program.cs
// ใช้ Factory สำหรับ DataLoader
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// ลงทะเบียน DataLoaders
builder.Services
    .AddGraphQLServer()
    .AddDataLoader<UserByIdDataLoader>()
    .AddDataLoader<ProductByIdDataLoader>()
    .AddDataLoader<OrdersByUserIdDataLoader>();
```

---

## ขั้นตอนที่ 687: Authentication และ Authorization ใน GraphQL

### ตั้งค่า JWT Authentication

```csharp
// Program.cs - ตั้งค่า Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("Admin", policy =>
        policy.RequireRole("Admin"));
    options.AddPolicy("Customer", policy =>
        policy.RequireRole("Customer", "Admin"));
});

// เพิ่ม Authorization ใน Hot Chocolate
builder.Services
    .AddGraphQLServer()
    .AddAuthorization()
    // ...
    ;

// Middleware
app.UseAuthentication();
app.UseAuthorization();
```

### ใช้ @authorize Directive

```csharp
// GraphQL/Queries/Query.cs
public class Query
{
    // ต้อง Login ก่อน
    [Authorize]
    [UseDbContext(typeof(AppDbContext))]
    public IQueryable<Order> GetMyOrders(
        [ScopedService] AppDbContext context,
        [GlobalState("currentUserId")] int userId)
        => context.Orders.Where(o => o.UserId == userId);

    // ต้องเป็น Admin เท่านั้น
    [Authorize(Policy = "Admin")]
    [UseDbContext(typeof(AppDbContext))]
    public IQueryable<User> GetAllUsers([ScopedService] AppDbContext context)
        => context.Users;

    // Public - ไม่ต้อง Login
    [UseDbContext(typeof(AppDbContext))]
    [UseFiltering]
    [UseSorting]
    public IQueryable<Product> GetProducts([ScopedService] AppDbContext context)
        => context.Products.Where(p => p.IsActive);
}
```

### Authorization ระดับ Field

```csharp
// GraphQL/Types/UserType.cs
public class UserType : ObjectType<User>
{
    protected override void Configure(IObjectTypeDescriptor<User> descriptor)
    {
        // ซ่อน Email จาก User อื่น
        descriptor
            .Field(u => u.Email)
            .Authorize(policy: "Admin")
            .Description("Email ของผู้ใช้ (เฉพาะ Admin เท่านั้น)");

        // ซ่อน PasswordHash เสมอ
        descriptor
            .Field(u => u.PasswordHash)
            .Ignore();
    }
}
```

### Login Mutation และ JWT Token

```csharp
// GraphQL/Mutations/AuthMutation.cs
public class AuthMutation
{
    [UseDbContext(typeof(AppDbContext))]
    public async Task<LoginPayload> LoginAsync(
        LoginInput input,
        [ScopedService] AppDbContext context,
        [Service] ITokenService tokenService,
        CancellationToken cancellationToken)
    {
        var user = await context.Users
            .FirstOrDefaultAsync(u => u.Email == input.Email, cancellationToken);

        if (user is null || !BCrypt.Net.BCrypt.Verify(input.Password, user.PasswordHash))
            return new LoginPayload(new UserError("อีเมลหรือรหัสผ่านไม่ถูกต้อง", "INVALID_CREDENTIALS"));

        var token = tokenService.GenerateToken(user);
        return new LoginPayload(token, user);
    }
}

// Services/TokenService.cs
public class TokenService : ITokenService
{
    private readonly IConfiguration _config;

    public TokenService(IConfiguration config) => _config = config;

    public string GenerateToken(User user)
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new Claim(ClaimTypes.Email, user.Email),
            new Claim(ClaimTypes.Role, user.Role),
            new Claim("userId", user.Id.ToString())
        };

        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_config["Jwt:Key"]!));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _config["Jwt:Issuer"],
            audience: _config["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddHours(24),
            signingCredentials: credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

### Global State - ดึงข้อมูล Current User

```csharp
// Middleware สำหรับดึง Current User
public class CurrentUserMiddleware
{
    private readonly RequestDelegate _next;

    public CurrentUserMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        var userId = context.User.FindFirst("userId")?.Value;

        if (int.TryParse(userId, out var id))
        {
            context.Items["currentUserId"] = id;
        }

        await _next(context);
    }
}

// ใช้งานใน Program.cs
app.UseMiddleware<CurrentUserMiddleware>();
```

---

## ขั้นตอนที่ 688: Error Handling และ Custom Scalars

### Error Handling ใน GraphQL

```csharp
// แบบ Functional Error (แนะนำ)
public class UpdateProductPayload
{
    public UpdateProductPayload(Product product)
        => Product = product;

    public UpdateProductPayload(IReadOnlyList<UserError> errors)
        => Errors = errors;

    public Product? Product { get; }
    public IReadOnlyList<UserError>? Errors { get; }
    public bool IsSuccess => Product is not null;
}

// ใช้งาน
public async Task<UpdateProductPayload> UpdateProductAsync(
    UpdateProductInput input,
    [ScopedService] AppDbContext context)
{
    var errors = new List<UserError>();

    if (input.Price <= 0)
        errors.Add(new UserError("ราคาต้องมากกว่า 0", "INVALID_PRICE"));

    if (input.Stock < 0)
        errors.Add(new UserError("สต็อกต้องไม่ติดลบ", "INVALID_STOCK"));

    if (errors.Any())
        return new UpdateProductPayload(errors);

    var product = await context.Products.FindAsync(input.Id);

    if (product is null)
        return new UpdateProductPayload(new[]
        {
            new UserError($"ไม่พบสินค้า ID: {input.Id}", "PRODUCT_NOT_FOUND")
        });

    // อัพเดท...
    return new UpdateProductPayload(product);
}
```

### Custom Exception Filter

```csharp
// GraphQL/Errors/AppErrorFilter.cs
public class AppErrorFilter : IErrorFilter
{
    private readonly ILogger<AppErrorFilter> _logger;

    public AppErrorFilter(ILogger<AppErrorFilter> logger)
        => _logger = logger;

    public IError OnError(IError error)
    {
        if (error.Exception is DbUpdateException dbEx)
        {
            _logger.LogError(dbEx, "Database error occurred");
            return error
                .WithMessage("เกิดข้อผิดพลาดกับฐานข้อมูล")
                .WithCode("DATABASE_ERROR")
                .RemoveExtensions();
        }

        if (error.Exception is UnauthorizedAccessException)
        {
            return error
                .WithMessage("ไม่มีสิทธิ์เข้าถึง")
                .WithCode("UNAUTHORIZED");
        }

        // Production: ซ่อน Stack Trace
        if (error.Exception is not null)
        {
            _logger.LogError(error.Exception, "Unhandled GraphQL error");
            return error
                .WithMessage("เกิดข้อผิดพลาดภายใน")
                .WithCode("INTERNAL_ERROR")
                .RemoveExtensions();
        }

        return error;
    }
}

// ลงทะเบียนใน Program.cs
builder.Services
    .AddGraphQLServer()
    .AddErrorFilter<AppErrorFilter>();
```

### Custom Scalars

```csharp
// GraphQL/Scalars/ThaiDateScalarType.cs
public class ThaiDateScalarType : ScalarType<DateTime, StringValueNode>
{
    private readonly string _format = "dd/MM/yyyy HH:mm";

    public ThaiDateScalarType() : base("ThaiDate")
    {
        Description = "วันที่และเวลาในรูปแบบไทย (dd/MM/yyyy HH:mm)";
    }

    public override IValueNode ParseResult(object? resultValue)
    {
        if (resultValue is DateTime dt)
            return new StringValueNode(dt.ToString(_format));

        throw new SerializationException(
            "ไม่สามารถแปลง DateTime ได้", this);
    }

    protected override DateTime ParseLiteral(StringValueNode valueSyntax)
    {
        if (DateTime.TryParseExact(
            valueSyntax.Value, _format,
            CultureInfo.InvariantCulture,
            DateTimeStyles.None,
            out var dt))
            return dt;

        throw new SerializationException(
            $"รูปแบบวันที่ไม่ถูกต้อง ต้องเป็น {_format}", this);
    }

    protected override StringValueNode ParseValue(DateTime runtimeValue)
        => new(runtimeValue.ToString(_format));
}

// Thai Baht Scalar
public class ThaiCurrencyScalarType : ScalarType<decimal, StringValueNode>
{
    public ThaiCurrencyScalarType() : base("THB")
    {
        Description = "จำนวนเงินบาทไทย";
    }

    public override IValueNode ParseResult(object? resultValue)
    {
        if (resultValue is decimal value)
            return new StringValueNode(value.ToString("N2"));

        throw new SerializationException("ไม่สามารถแปลงค่าเงินได้", this);
    }

    protected override decimal ParseLiteral(StringValueNode valueSyntax)
    {
        if (decimal.TryParse(valueSyntax.Value,
            NumberStyles.Any, CultureInfo.InvariantCulture, out var d))
            return d;

        throw new SerializationException("รูปแบบจำนวนเงินไม่ถูกต้อง", this);
    }

    protected override StringValueNode ParseValue(decimal runtimeValue)
        => new(runtimeValue.ToString("N2"));
}

// ลงทะเบียน
builder.Services
    .AddGraphQLServer()
    .AddType<ThaiDateScalarType>()
    .AddType<ThaiCurrencyScalarType>();
```

### Validation Middleware

```csharp
// GraphQL/Middleware/ValidationMiddleware.cs
public class InputValidationMiddleware
{
    private readonly FieldDelegate _next;

    public InputValidationMiddleware(FieldDelegate next) => _next = next;

    public async Task InvokeAsync(IMiddlewareContext context)
    {
        // ตรวจสอบ Input ก่อน Resolver ทำงาน
        foreach (var argument in context.Field.Arguments)
        {
            var value = context.ArgumentValue<object?>(argument.Name);

            if (value is IValidatable validatable)
            {
                var validationResult = validatable.Validate();
                if (!validationResult.IsValid)
                {
                    context.ReportError(
                        ErrorBuilder.New()
                            .SetMessage(validationResult.ErrorMessage)
                            .SetCode("VALIDATION_ERROR")
                            .Build());
                    return;
                }
            }
        }

        await _next(context);
    }
}
```

---

## ขั้นตอนที่ 689: Schema Stitching และ Federation

### Schema Stitching คืออะไร?

Schema Stitching คือการรวม GraphQL Schema หลายอันเข้าด้วยกัน เหมาะสำหรับ Microservices

```
Microservices Architecture:
┌────────────────────┐   ┌────────────────────┐
│  Product Service   │   │   Order Service     │
│  GraphQL Schema    │   │   GraphQL Schema    │
│  - products        │   │  - orders           │
│  - categories      │   │  - orderItems       │
└─────────┬──────────┘   └──────────┬──────────┘
          │                          │
          └──────────┬───────────────┘
                     │
          ┌──────────▼──────────┐
          │   Gateway Service   │
          │  Stitched Schema    │
          │  - products         │
          │  - orders           │
          │  - users            │
          └─────────────────────┘
```

### ตั้งค่า Gateway ด้วย Hot Chocolate

```csharp
// Gateway/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddHttpClient("products", c => c.BaseAddress =
        new Uri("http://product-service/graphql"))
    .AddHttpClient("orders", c => c.BaseAddress =
        new Uri("http://order-service/graphql"))
    .AddHttpClient("users", c => c.BaseAddress =
        new Uri("http://user-service/graphql"));

builder.Services
    .AddGraphQLServer()
    .AddRemoteSchema("products")
    .AddRemoteSchema("orders")
    .AddRemoteSchema("users")
    .AddTypeExtensionsFromFile("./stitching.graphql");
```

### Extension Schema สำหรับ Stitching

```graphql
# stitching.graphql
# เพิ่ม Field จาก Remote Schema อื่นเข้ามา
extend type Order {
  product: Product!
    @delegate(schema: "products", path: "product(id: $fields:productId)")
}

extend type Product {
  orders: [Order!]!
    @delegate(schema: "orders", path: "ordersByProduct(productId: $fields:id)")
}
```

### Apollo Federation

```csharp
// Product Service - Federation Provider
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddApolloFederation();

// ใน Product Type
[Key("id")] // Federation Key
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = default!;
    public decimal Price { get; set; }
}

public class ProductType : ObjectType<Product>
{
    protected override void Configure(IObjectTypeDescriptor<Product> descriptor)
    {
        descriptor.Key("id");

        // Reference Resolver สำหรับ Federation
        descriptor
            .ResolveReferenceWith<ProductResolvers>(
                r => r.GetProductByIdAsync(default!, default!, default!));
    }
}

public class ProductResolvers
{
    public Task<Product?> GetProductByIdAsync(
        [Parent] Product productRef,
        ProductByIdDataLoader dataLoader,
        CancellationToken cancellationToken)
        => dataLoader.LoadAsync(productRef.Id, cancellationToken);
}
```

### Order Service ที่อ้างอิง Product

```csharp
// Order Service - Federation Consumer
[ExtendServiceType] // Extend type จาก Service อื่น
public class ProductExtension : ObjectTypeExtension<Product>
{
    protected override void Configure(IObjectTypeDescriptor<Product> descriptor)
    {
        descriptor.Key("id");
        descriptor.Field("id").IsExternal();

        // เพิ่ม Field ใหม่บน Product จาก Order Service
        descriptor
            .Field("orderCount")
            .ResolveWith<OrderServiceResolvers>(
                r => r.GetOrderCountForProductAsync(default!, default!, default!));
    }
}

public class OrderServiceResolvers
{
    [UseDbContext(typeof(OrderDbContext))]
    public async Task<int> GetOrderCountForProductAsync(
        [Parent] Product product,
        [ScopedService] OrderDbContext context,
        CancellationToken cancellationToken)
    {
        return await context.OrderItems
            .CountAsync(oi => oi.ProductId == product.Id, cancellationToken);
    }
}
```

---

## ขั้นตอนที่ 690: Full E-Commerce GraphQL API Example

### ภาพรวม E-Commerce GraphQL API

```
E-Commerce GraphQL API:
┌──────────────────────────────────────────────────────────┐
│                    GraphQL Schema                         │
│                                                          │
│  Queries:              Mutations:        Subscriptions:  │
│  - products            - addProduct      - onOrderCreated│
│  - product(id)         - updateProduct   - onPriceUpdate │
│  - orders (auth)       - deleteProduct   - onStockChange │
│  - myOrders (auth)     - createOrder                     │
│  - users (admin)       - updateOrderStatus               │
│  - me (auth)           - login                           │
│                        - register                        │
└──────────────────────────────────────────────────────────┘
```

### Complete Query Type

```csharp
// GraphQL/Queries/Query.cs
[QueryType]
public class Query
{
    // === Products ===
    [UseDbContext(typeof(AppDbContext))]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    [UsePaging(IncludeTotalCount = true, DefaultPageSize = 20)]
    public IQueryable<Product> GetProducts([ScopedService] AppDbContext ctx)
        => ctx.Products.Where(p => p.IsActive);

    public Task<Product?> GetProduct(
        int id,
        ProductByIdDataLoader loader,
        CancellationToken ct)
        => loader.LoadAsync(id, ct);

    [UseDbContext(typeof(AppDbContext))]
    public async Task<IEnumerable<string>> GetCategories(
        [ScopedService] AppDbContext ctx)
        => await ctx.Products
            .Select(p => p.Category)
            .Distinct()
            .OrderBy(c => c)
            .ToListAsync();

    // === Orders ===
    [Authorize]
    [UseDbContext(typeof(AppDbContext))]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    [UsePaging]
    public IQueryable<Order> GetMyOrders(
        [ScopedService] AppDbContext ctx,
        ClaimsPrincipal claimsPrincipal)
    {
        var userId = int.Parse(
            claimsPrincipal.FindFirst(ClaimTypes.NameIdentifier)!.Value);
        return ctx.Orders.Where(o => o.UserId == userId);
    }

    [Authorize(Policy = "Admin")]
    [UseDbContext(typeof(AppDbContext))]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    [UsePaging]
    public IQueryable<Order> GetOrders([ScopedService] AppDbContext ctx)
        => ctx.Orders;

    // === Users ===
    [Authorize]
    [UseDbContext(typeof(AppDbContext))]
    [UseProjection]
    public async Task<User?> GetMeAsync(
        [ScopedService] AppDbContext ctx,
        ClaimsPrincipal claimsPrincipal,
        CancellationToken ct)
    {
        var userId = int.Parse(
            claimsPrincipal.FindFirst(ClaimTypes.NameIdentifier)!.Value);
        return await ctx.Users.FindAsync(new object[] { userId }, ct);
    }

    [Authorize(Policy = "Admin")]
    [UseDbContext(typeof(AppDbContext))]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    [UsePaging]
    public IQueryable<User> GetUsers([ScopedService] AppDbContext ctx)
        => ctx.Users;
}
```

### Complete Mutation Type

```csharp
// GraphQL/Mutations/Mutation.cs
[MutationType]
public class Mutation
{
    // === Auth ===
    [UseDbContext(typeof(AppDbContext))]
    public async Task<LoginPayload> LoginAsync(
        LoginInput input,
        [ScopedService] AppDbContext ctx,
        [Service] ITokenService tokenService,
        CancellationToken ct)
    {
        var user = await ctx.Users
            .FirstOrDefaultAsync(u => u.Email == input.Email, ct);

        if (user is null || !BCrypt.Net.BCrypt.Verify(input.Password, user.PasswordHash))
            return new LoginPayload(new UserError("ข้อมูลไม่ถูกต้อง", "INVALID_CREDENTIALS"));

        return new LoginPayload(tokenService.GenerateToken(user), user);
    }

    [UseDbContext(typeof(AppDbContext))]
    public async Task<RegisterPayload> RegisterAsync(
        RegisterInput input,
        [ScopedService] AppDbContext ctx,
        [Service] ITokenService tokenService,
        CancellationToken ct)
    {
        if (await ctx.Users.AnyAsync(u => u.Email == input.Email, ct))
            return new RegisterPayload(new UserError("อีเมลนี้มีผู้ใช้งานแล้ว", "EMAIL_EXISTS"));

        var user = new User
        {
            Name = input.Name,
            Email = input.Email,
            PasswordHash = BCrypt.Net.BCrypt.HashPassword(input.Password),
            Role = "Customer"
        };

        ctx.Users.Add(user);
        await ctx.SaveChangesAsync(ct);

        return new RegisterPayload(tokenService.GenerateToken(user), user);
    }

    // === Products ===
    [Authorize(Policy = "Admin")]
    [UseDbContext(typeof(AppDbContext))]
    public async Task<ProductPayload> AddProductAsync(
        AddProductInput input,
        [ScopedService] AppDbContext ctx,
        [Service] ITopicEventSender sender,
        CancellationToken ct)
    {
        var product = new Product
        {
            Name = input.Name,
            Description = input.Description,
            Price = input.Price,
            Stock = input.Stock,
            Category = input.Category,
            ImageUrl = input.ImageUrl ?? string.Empty
        };

        ctx.Products.Add(product);
        await ctx.SaveChangesAsync(ct);

        await sender.SendAsync(
            $"OnProductAdded_{product.Category}", product, ct);

        return new ProductPayload(product);
    }

    [Authorize(Policy = "Admin")]
    [UseDbContext(typeof(AppDbContext))]
    public async Task<ProductPayload> UpdateProductPriceAsync(
        int productId,
        decimal newPrice,
        [ScopedService] AppDbContext ctx,
        [Service] ITopicEventSender sender,
        CancellationToken ct)
    {
        var product = await ctx.Products.FindAsync(
            new object[] { productId }, ct);

        if (product is null)
            return new ProductPayload(
                new UserError($"ไม่พบสินค้า ID: {productId}", "NOT_FOUND"));

        product.Price = newPrice;
        await ctx.SaveChangesAsync(ct);

        // แจ้ง Subscriber
        await sender.SendAsync(
            $"OnPriceChanged_{product.Category}", product, ct);

        return new ProductPayload(product);
    }

    // === Orders ===
    [Authorize]
    [UseDbContext(typeof(AppDbContext))]
    public async Task<OrderPayload> CreateOrderAsync(
        CreateOrderInput input,
        [ScopedService] AppDbContext ctx,
        [Service] ITopicEventSender sender,
        ClaimsPrincipal claimsPrincipal,
        CancellationToken ct)
    {
        var userId = int.Parse(
            claimsPrincipal.FindFirst(ClaimTypes.NameIdentifier)!.Value);

        // ตรวจสอบสต็อก
        var errors = new List<UserError>();
        var productIds = input.Items.Select(i => i.ProductId).ToList();
        var products = await ctx.Products
            .Where(p => productIds.Contains(p.Id))
            .ToDictionaryAsync(p => p.Id, ct);

        foreach (var item in input.Items)
        {
            if (!products.TryGetValue(item.ProductId, out var product))
            {
                errors.Add(new UserError(
                    $"ไม่พบสินค้า ID: {item.ProductId}", "PRODUCT_NOT_FOUND"));
                continue;
            }

            if (product.Stock < item.Quantity)
                errors.Add(new UserError(
                    $"สินค้า '{product.Name}' สต็อกไม่พอ (มี {product.Stock} ชิ้น)",
                    "INSUFFICIENT_STOCK"));
        }

        if (errors.Any())
            return new OrderPayload(errors);

        var order = new Order
        {
            UserId = userId,
            Items = input.Items.Select(item => new OrderItem
            {
                ProductId = item.ProductId,
                Quantity = item.Quantity,
                UnitPrice = products[item.ProductId].Price
            }).ToList()
        };

        order.Total = order.Items.Sum(i => i.Quantity * i.UnitPrice);

        // ลดสต็อก
        foreach (var item in input.Items)
            products[item.ProductId].Stock -= item.Quantity;

        ctx.Orders.Add(order);
        await ctx.SaveChangesAsync(ct);

        await sender.SendAsync(nameof(Subscription.OnOrderCreated), order, ct);

        return new OrderPayload(order);
    }
}
```

### Complete Subscription Type

```csharp
// GraphQL/Subscriptions/Subscription.cs
[SubscriptionType]
public class Subscription
{
    [Subscribe]
    [Topic]
    public Order OnOrderCreated([EventMessage] Order order)
        => order;

    [Subscribe]
    [Topic("{orderId}")]
    public Order OnOrderStatusChanged(
        int orderId,
        [EventMessage] Order order)
        => order;

    [Subscribe]
    [Topic("{category}")]
    public Product OnPriceChanged(
        string category,
        [EventMessage] Product product)
        => product;

    [Subscribe]
    [Topic("{category}")]
    public Product OnProductAdded(
        string category,
        [EventMessage] Product product)
        => product;
}
```

### ตัวอย่างการใช้งานครบวงจร

```graphql
# 1. Register
mutation Register {
  register(input: {
    name: "สมชาย ใจดี"
    email: "somchai@example.com"
    password: "SecurePass123!"
  }) {
    token
    user {
      id
      name
      email
    }
  }
}

# 2. Login
mutation Login {
  login(input: {
    email: "somchai@example.com"
    password: "SecurePass123!"
  }) {
    token
    user {
      id
      name
      role
    }
  }
}

# 3. ดูสินค้า (Header: Authorization: Bearer <token>)
query BrowseProducts {
  products(
    where: {
      and: [
        { category: { eq: "Electronics" } }
        { price: { lte: 30000 } }
      ]
    }
    order: { price: ASC }
    first: 5
  ) {
    totalCount
    nodes {
      id
      name
      price
      stock
      isInStock
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}

# 4. สร้าง Order
mutation PlaceOrder {
  createOrder(input: {
    items: [
      { productId: 1, quantity: 2 }
      { productId: 5, quantity: 1 }
    ]
  }) {
    order {
      id
      total
      status
      items {
        product { name }
        quantity
        unitPrice
      }
    }
    errors {
      message
      code
    }
  }
}

# 5. Subscribe รับแจ้งเตือน Order ใหม่ (Admin)
subscription AdminOrderWatch {
  onOrderCreated {
    id
    total
    user {
      name
      email
    }
    items {
      product { name }
      quantity
    }
  }
}
```

### การทดสอบด้วย Integration Tests

```csharp
// Tests/GraphQLIntegrationTests.cs
public class ProductQueryTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public ProductQueryTests(WebApplicationFactory<Program> factory)
        => _factory = factory;

    [Fact]
    public async Task GetProducts_ShouldReturnActiveProducts()
    {
        var client = _factory.CreateClient();

        var query = new
        {
            query = """
                query {
                  products {
                    nodes {
                      id
                      name
                      price
                    }
                  }
                }
                """
        };

        var response = await client.PostAsJsonAsync("/graphql", query);
        response.EnsureSuccessStatusCode();

        var result = await response.Content
            .ReadFromJsonAsync<GraphQLResponse<ProductsData>>();

        Assert.NotNull(result?.Data?.Products?.Nodes);
        Assert.All(result.Data.Products.Nodes, p => Assert.True(p.Price > 0));
    }

    [Fact]
    public async Task CreateOrder_WithoutAuth_ShouldFail()
    {
        var client = _factory.CreateClient();

        var mutation = new
        {
            query = """
                mutation {
                  createOrder(input: { items: [{ productId: 1, quantity: 1 }] }) {
                    order { id }
                  }
                }
                """
        };

        var response = await client.PostAsJsonAsync("/graphql", mutation);
        var result = await response.Content
            .ReadFromJsonAsync<JsonElement>();

        // ต้องมี Error เพราะไม่ได้ Login
        Assert.True(result.GetProperty("errors").GetArrayLength() > 0);
    }
}
```

### appsettings.json

```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=ECommerceGraphQL;Trusted_Connection=true;"
  },
  "Jwt": {
    "Key": "your-super-secret-key-minimum-32-characters",
    "Issuer": "ECommerceAPI",
    "Audience": "ECommerceClients"
  },
  "GraphQL": {
    "MaxAllowedComplexity": 250,
    "MaxAllowedDepth": 10
  }
}
```

### ตั้งค่า Complexity และ Depth Limiting

```csharp
// Program.cs - ป้องกัน GraphQL DoS Attack
builder.Services
    .AddGraphQLServer()
    .AddMaxExecutionDepthRule(maxAllowedExecutionDepth: 10)
    .ModifyRequestOptions(opt =>
    {
        opt.IncludeExceptionDetails =
            builder.Environment.IsDevelopment();
    });
```

---

## 🔑 สรุป Key Concepts

```
GraphQL Hot Chocolate Summary:
┌────────────────────────────────────────────────────────────┐
│  Component        │ หน้าที่                                  │
├────────────────────────────────────────────────────────────┤
│  QueryType        │ อ่านข้อมูล (GET equivalent)              │
│  MutationType     │ เปลี่ยนแปลงข้อมูล (POST/PUT/DELETE)      │
│  SubscriptionType │ Real-time via WebSocket                 │
│  DataLoader       │ แก้ปัญหา N+1 ด้วย Batch Loading         │
│  [UseFiltering]   │ เพิ่ม where parameter อัตโนมัติ          │
│  [UseSorting]     │ เพิ่ม order parameter อัตโนมัติ           │
│  [UsePaging]      │ Cursor-based Pagination                 │
│  [Authorize]      │ JWT Authentication & Authorization      │
│  ErrorFilter      │ จัดการ Exception แบบ Global              │
│  Custom Scalars   │ Type พิเศษ เช่น ThaiDate, THB           │
│  Schema Stitching │ รวม Multiple GraphQL Schemas            │
└────────────────────────────────────────────────────────────┘
```

## 📦 Packages ที่ใช้ทั้งหมด

```xml
<!-- .csproj -->
<PackageReference Include="HotChocolate.AspNetCore" Version="14.*" />
<PackageReference Include="HotChocolate.Data" Version="14.*" />
<PackageReference Include="HotChocolate.Data.EntityFramework" Version="14.*" />
<PackageReference Include="HotChocolate.AspNetCore.Authorization" Version="14.*" />
<PackageReference Include="HotChocolate.Stitching" Version="14.*" />
<PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="8.*" />
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.*" />
<PackageReference Include="BCrypt.Net-Next" Version="4.*" />
```

---

**ก่อนหน้า → [Part 68: gRPC Services](part68-grpc.md)**
**ต่อไป → [Part 70: Caching Strategies](part70-caching.md)**
