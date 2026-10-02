# Part 81: Capstone Project - E-Commerce Platform (ShopThai)

## ภาพรวมโปรเจค Capstone: แพลตฟอร์ม E-Commerce ที่ครบถ้วน

ยินดีต้อนรับสู่ Capstone Project ที่รวมทุกสิ่งที่เรียนมาทั้งหมด! ในส่วนนี้เราจะสร้าง **ShopThai** - แพลตฟอร์ม E-Commerce ที่ใช้งานได้จริง โดยใช้ C# และ .NET พร้อมสถาปัตยกรรม Modular Monolith ที่ทันสมัย

---

## Step 801: Project Overview และ Architecture Decision

### ShopThai - แพลตฟอร์ม E-Commerce ไทย

**ShopThai** เป็นระบบ E-Commerce ที่รองรับฟีเจอร์หลักดังนี้:
- ระบบสินค้าและการค้นหา (Product Catalog & Search)
- ตะกร้าสินค้า (Shopping Cart)
- ระบบสั่งซื้อ (Order Management)
- การชำระเงิน (Payment Processing)
- Dashboard สำหรับ Admin
- การแจ้งเตือนทาง Email

### ทำไมต้องเลือก Modular Monolith?

**Modular Monolith** คือสถาปัตยกรรมที่ได้รับความนิยมมากขึ้นเพราะ:
1. **ง่ายต่อการ Deploy** - Deploy ไฟล์เดียวเหมือน Monolith
2. **แยก Module ชัดเจน** - แต่ละ Module มี Boundary ของตัวเอง
3. **เตรียมพร้อมสู่ Microservices** - แยกออกได้ง่ายในอนาคต
4. **Team สามารถทำงานแยกกัน** - แต่ละ Module เป็นอิสระ

```
ShopThai/
├── src/
│   ├── ShopThai.API/                    # Entry point - HTTP API
│   ├── ShopThai.Shared/                 # Shared kernel
│   ├── Modules/
│   │   ├── Catalog/                     # Product catalog module
│   │   │   ├── ShopThai.Catalog.Domain/
│   │   │   ├── ShopThai.Catalog.Application/
│   │   │   └── ShopThai.Catalog.Infrastructure/
│   │   ├── Orders/                      # Order management module
│   │   │   ├── ShopThai.Orders.Domain/
│   │   │   ├── ShopThai.Orders.Application/
│   │   │   └── ShopThai.Orders.Infrastructure/
│   │   ├── Payments/                    # Payment module
│   │   │   ├── ShopThai.Payments.Domain/
│   │   │   ├── ShopThai.Payments.Application/
│   │   │   └── ShopThai.Payments.Infrastructure/
│   │   └── Users/                       # User management module
│   │       ├── ShopThai.Users.Domain/
│   │       ├── ShopThai.Users.Application/
│   │       └── ShopThai.Users.Infrastructure/
└── tests/
    ├── ShopThai.Catalog.Tests/
    ├── ShopThai.Orders.Tests/
    ├── ShopThai.Payments.Tests/
    └── ShopThai.Users.Tests/
```

### สร้าง Solution Structure เริ่มต้น

```bash
# สร้าง solution
dotnet new sln -n ShopThai

# สร้าง Shared Kernel
dotnet new classlib -n ShopThai.Shared -o src/ShopThai.Shared

# สร้าง API Project
dotnet new webapi -n ShopThai.API -o src/ShopThai.API

# สร้าง Module - Catalog
dotnet new classlib -n ShopThai.Catalog.Domain -o src/Modules/Catalog/ShopThai.Catalog.Domain
dotnet new classlib -n ShopThai.Catalog.Application -o src/Modules/Catalog/ShopThai.Catalog.Application
dotnet new classlib -n ShopThai.Catalog.Infrastructure -o src/Modules/Catalog/ShopThai.Catalog.Infrastructure

# สร้าง Module - Orders
dotnet new classlib -n ShopThai.Orders.Domain -o src/Modules/Orders/ShopThai.Orders.Domain
dotnet new classlib -n ShopThai.Orders.Application -o src/Modules/Orders/ShopThai.Orders.Application
dotnet new classlib -n ShopThai.Orders.Infrastructure -o src/Modules/Orders/ShopThai.Orders.Infrastructure

# สร้าง Module - Payments
dotnet new classlib -n ShopThai.Payments.Domain -o src/Modules/Payments/ShopThai.Payments.Domain
dotnet new classlib -n ShopThai.Payments.Application -o src/Modules/Payments/ShopThai.Payments.Application
dotnet new classlib -n ShopThai.Payments.Infrastructure -o src/Modules/Payments/ShopThai.Payments.Infrastructure

# สร้าง Module - Users
dotnet new classlib -n ShopThai.Users.Domain -o src/Modules/Users/ShopThai.Users.Domain
dotnet new classlib -n ShopThai.Users.Application -o src/Modules/Users/ShopThai.Users.Application
dotnet new classlib -n ShopThai.Users.Infrastructure -o src/Modules/Users/ShopThai.Users.Infrastructure
```

---

## Step 802: Domain Modeling - สร้าง Domain ที่แข็งแกร่ง

### Shared Kernel - กำหนด Base Classes

**Shared Kernel** คือส่วนที่ทุก Module แชร์กัน ได้แก่ Base Entity, Value Object, Domain Event

```csharp
// src/ShopThai.Shared/Domain/Entity.cs
namespace ShopThai.Shared.Domain;

public abstract class Entity<TId>
{
    public TId Id { get; protected set; }
    private readonly List<IDomainEvent> _domainEvents = new();
    
    protected Entity(TId id)
    {
        Id = id;
    }
    
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();
    
    protected void RaiseDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }
    
    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
    
    public override bool Equals(object? obj)
    {
        if (obj is not Entity<TId> other) return false;
        if (ReferenceEquals(this, other)) return true;
        return Id!.Equals(other.Id);
    }
    
    public override int GetHashCode() => Id!.GetHashCode();
}
```

```csharp
// src/ShopThai.Shared/Domain/AggregateRoot.cs
namespace ShopThai.Shared.Domain;

public abstract class AggregateRoot<TId> : Entity<TId>
{
    protected AggregateRoot(TId id) : base(id) { }
}
```

```csharp
// src/ShopThai.Shared/Domain/ValueObject.cs
namespace ShopThai.Shared.Domain;

public abstract class ValueObject
{
    protected abstract IEnumerable<object?> GetAtomicValues();
    
    public override bool Equals(object? obj)
    {
        if (obj is null || obj.GetType() != GetType()) return false;
        var other = (ValueObject)obj;
        return GetAtomicValues().SequenceEqual(other.GetAtomicValues());
    }
    
    public override int GetHashCode()
    {
        return GetAtomicValues()
            .Aggregate(default(int), HashCode.Combine);
    }
    
    public static bool operator ==(ValueObject? left, ValueObject? right)
        => Equals(left, right);
    
    public static bool operator !=(ValueObject? left, ValueObject? right)
        => !Equals(left, right);
}
```

```csharp
// src/ShopThai.Shared/Domain/IDomainEvent.cs
using MediatR;

namespace ShopThai.Shared.Domain;

public interface IDomainEvent : INotification
{
    Guid EventId { get; }
    DateTime OccurredOn { get; }
}
```

```csharp
// src/ShopThai.Shared/Domain/DomainEvent.cs
namespace ShopThai.Shared.Domain;

public abstract record DomainEvent : IDomainEvent
{
    public Guid EventId { get; init; } = Guid.NewGuid();
    public DateTime OccurredOn { get; init; } = DateTime.UtcNow;
}
```

### Catalog Module - Domain Models

**Catalog Module** จัดการสินค้าทั้งหมด

```csharp
// src/Modules/Catalog/ShopThai.Catalog.Domain/Entities/Product.cs
using ShopThai.Shared.Domain;
using ShopThai.Catalog.Domain.ValueObjects;
using ShopThai.Catalog.Domain.Events;

namespace ShopThai.Catalog.Domain.Entities;

public class Product : AggregateRoot<Guid>
{
    public string Name { get; private set; }
    public string Description { get; private set; }
    public Money Price { get; private set; }
    public int StockQuantity { get; private set; }
    public string ImageUrl { get; private set; }
    public Guid CategoryId { get; private set; }
    public bool IsActive { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? UpdatedAt { get; private set; }
    
    private Product() : base(Guid.Empty) { } // EF Core
    
    private Product(
        Guid id,
        string name,
        string description,
        Money price,
        int stockQuantity,
        string imageUrl,
        Guid categoryId) : base(id)
    {
        Name = name;
        Description = description;
        Price = price;
        StockQuantity = stockQuantity;
        ImageUrl = imageUrl;
        CategoryId = categoryId;
        IsActive = true;
        CreatedAt = DateTime.UtcNow;
    }
    
    public static Product Create(
        string name,
        string description,
        Money price,
        int stockQuantity,
        string imageUrl,
        Guid categoryId)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("ชื่อสินค้าต้องไม่ว่าง", nameof(name));
        
        if (stockQuantity < 0)
            throw new ArgumentException("จำนวนสินค้าต้องไม่ติดลบ", nameof(stockQuantity));
        
        var product = new Product(
            Guid.NewGuid(),
            name,
            description,
            price,
            stockQuantity,
            imageUrl,
            categoryId);
        
        product.RaiseDomainEvent(new ProductCreatedEvent(product.Id, product.Name));
        
        return product;
    }
    
    public void UpdatePrice(Money newPrice)
    {
        var oldPrice = Price;
        Price = newPrice;
        UpdatedAt = DateTime.UtcNow;
        
        RaiseDomainEvent(new ProductPriceChangedEvent(Id, oldPrice, newPrice));
    }
    
    public void AddStock(int quantity)
    {
        if (quantity <= 0)
            throw new ArgumentException("จำนวนที่เพิ่มต้องมากกว่า 0");
        
        StockQuantity += quantity;
        UpdatedAt = DateTime.UtcNow;
    }
    
    public void DeductStock(int quantity)
    {
        if (quantity <= 0)
            throw new ArgumentException("จำนวนที่ตัดต้องมากกว่า 0");
        
        if (StockQuantity < quantity)
            throw new InvalidOperationException($"สินค้าไม่เพียงพอ: มีอยู่ {StockQuantity} ชิ้น");
        
        StockQuantity -= quantity;
        UpdatedAt = DateTime.UtcNow;
        
        if (StockQuantity == 0)
            RaiseDomainEvent(new ProductOutOfStockEvent(Id, Name));
    }
    
    public void Deactivate()
    {
        IsActive = false;
        UpdatedAt = DateTime.UtcNow;
    }
}
```

```csharp
// src/Modules/Catalog/ShopThai.Catalog.Domain/ValueObjects/Money.cs
using ShopThai.Shared.Domain;

namespace ShopThai.Catalog.Domain.ValueObjects;

public class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }
    
    private Money() { } // EF Core
    
    public Money(decimal amount, string currency = "THB")
    {
        if (amount < 0)
            throw new ArgumentException("จำนวนเงินต้องไม่ติดลบ");
        
        if (string.IsNullOrWhiteSpace(currency))
            throw new ArgumentException("สกุลเงินต้องไม่ว่าง");
        
        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }
    
    public static Money FromTHB(decimal amount) => new(amount, "THB");
    public static Money Zero => new(0, "THB");
    
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("ไม่สามารถบวกเงินต่างสกุลได้");
        
        return new Money(Amount + other.Amount, Currency);
    }
    
    public Money Multiply(int factor) => new(Amount * factor, Currency);
    
    public override string ToString() => $"{Amount:N2} {Currency}";
    
    protected override IEnumerable<object?> GetAtomicValues()
    {
        yield return Amount;
        yield return Currency;
    }
}
```

```csharp
// src/Modules/Catalog/ShopThai.Catalog.Domain/Events/ProductCreatedEvent.cs
using ShopThai.Shared.Domain;

namespace ShopThai.Catalog.Domain.Events;

public record ProductCreatedEvent(
    Guid ProductId,
    string ProductName) : DomainEvent;

public record ProductPriceChangedEvent(
    Guid ProductId,
    Money OldPrice,
    Money NewPrice) : DomainEvent;

public record ProductOutOfStockEvent(
    Guid ProductId,
    string ProductName) : DomainEvent;
```

### Orders Module - Domain Models

```csharp
// src/Modules/Orders/ShopThai.Orders.Domain/Entities/Order.cs
using ShopThai.Shared.Domain;
using ShopThai.Orders.Domain.ValueObjects;
using ShopThai.Orders.Domain.Events;

namespace ShopThai.Orders.Domain.Entities;

public class Order : AggregateRoot<Guid>
{
    private readonly List<OrderItem> _items = new();
    
    public Guid CustomerId { get; private set; }
    public string OrderNumber { get; private set; }
    public OrderStatus Status { get; private set; }
    public Address ShippingAddress { get; private set; }
    public Money TotalAmount { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? UpdatedAt { get; private set; }
    public string? Note { get; private set; }
    
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    
    private Order() : base(Guid.Empty) { }
    
    private Order(Guid id, Guid customerId, Address shippingAddress) : base(id)
    {
        CustomerId = customerId;
        ShippingAddress = shippingAddress;
        OrderNumber = GenerateOrderNumber();
        Status = OrderStatus.Pending;
        TotalAmount = Money.Zero;
        CreatedAt = DateTime.UtcNow;
    }
    
    public static Order Create(Guid customerId, Address shippingAddress)
    {
        return new Order(Guid.NewGuid(), customerId, shippingAddress);
    }
    
    public void AddItem(Guid productId, string productName, Money unitPrice, int quantity)
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException("ไม่สามารถเพิ่มสินค้าในออร์เดอร์ที่ดำเนินการแล้ว");
        
        var existingItem = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existingItem != null)
        {
            existingItem.IncreaseQuantity(quantity);
        }
        else
        {
            _items.Add(OrderItem.Create(productId, productName, unitPrice, quantity));
        }
        
        RecalculateTotal();
    }
    
    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException("สามารถยืนยันได้เฉพาะออร์เดอร์ที่รอดำเนินการ");
        
        if (!_items.Any())
            throw new InvalidOperationException("ออร์เดอร์ต้องมีสินค้าอย่างน้อย 1 รายการ");
        
        Status = OrderStatus.Confirmed;
        UpdatedAt = DateTime.UtcNow;
        
        RaiseDomainEvent(new OrderConfirmedEvent(Id, CustomerId, TotalAmount));
    }
    
    public void MarkAsPaid(string paymentTransactionId)
    {
        if (Status != OrderStatus.Confirmed)
            throw new InvalidOperationException("สามารถชำระเงินได้เฉพาะออร์เดอร์ที่ยืนยันแล้ว");
        
        Status = OrderStatus.Paid;
        UpdatedAt = DateTime.UtcNow;
        
        RaiseDomainEvent(new OrderPaidEvent(Id, paymentTransactionId));
    }
    
    public void Ship(string trackingNumber)
    {
        if (Status != OrderStatus.Paid)
            throw new InvalidOperationException("สามารถจัดส่งได้เฉพาะออร์เดอร์ที่ชำระเงินแล้ว");
        
        Status = OrderStatus.Shipped;
        UpdatedAt = DateTime.UtcNow;
        
        RaiseDomainEvent(new OrderShippedEvent(Id, CustomerId, trackingNumber));
    }
    
    public void Cancel(string reason)
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Delivered or OrderStatus.Cancelled)
            throw new InvalidOperationException("ไม่สามารถยกเลิกออร์เดอร์นี้ได้");
        
        Status = OrderStatus.Cancelled;
        Note = reason;
        UpdatedAt = DateTime.UtcNow;
        
        RaiseDomainEvent(new OrderCancelledEvent(Id, CustomerId, reason));
    }
    
    private void RecalculateTotal()
    {
        TotalAmount = _items.Aggregate(
            Money.Zero,
            (total, item) => total.Add(item.Subtotal));
    }
    
    private static string GenerateOrderNumber()
    {
        var timestamp = DateTime.UtcNow.ToString("yyyyMMddHHmmss");
        var random = new Random().Next(1000, 9999);
        return $"ST{timestamp}{random}";
    }
}

public enum OrderStatus
{
    Pending,
    Confirmed,
    Paid,
    Shipped,
    Delivered,
    Cancelled
}
```

```csharp
// src/Modules/Orders/ShopThai.Orders.Domain/Entities/OrderItem.cs
using ShopThai.Shared.Domain;
using ShopThai.Orders.Domain.ValueObjects;

namespace ShopThai.Orders.Domain.Entities;

public class OrderItem : Entity<Guid>
{
    public Guid ProductId { get; private set; }
    public string ProductName { get; private set; }
    public Money UnitPrice { get; private set; }
    public int Quantity { get; private set; }
    public Money Subtotal => UnitPrice.Multiply(Quantity);
    
    private OrderItem() : base(Guid.Empty) { }
    
    private OrderItem(Guid id, Guid productId, string productName, Money unitPrice, int quantity)
        : base(id)
    {
        ProductId = productId;
        ProductName = productName;
        UnitPrice = unitPrice;
        Quantity = quantity;
    }
    
    internal static OrderItem Create(Guid productId, string productName, Money unitPrice, int quantity)
    {
        if (quantity <= 0)
            throw new ArgumentException("จำนวนต้องมากกว่า 0");
        
        return new OrderItem(Guid.NewGuid(), productId, productName, unitPrice, quantity);
    }
    
    internal void IncreaseQuantity(int additionalQuantity)
    {
        if (additionalQuantity <= 0)
            throw new ArgumentException("จำนวนที่เพิ่มต้องมากกว่า 0");
        
        Quantity += additionalQuantity;
    }
}
```

```csharp
// src/Modules/Orders/ShopThai.Orders.Domain/ValueObjects/Address.cs
using ShopThai.Shared.Domain;

namespace ShopThai.Orders.Domain.ValueObjects;

public class Address : ValueObject
{
    public string Street { get; }
    public string City { get; }
    public string Province { get; }
    public string PostalCode { get; }
    public string Country { get; }
    
    private Address() { }
    
    public Address(string street, string city, string province, string postalCode, string country = "Thailand")
    {
        Street = street ?? throw new ArgumentNullException(nameof(street));
        City = city ?? throw new ArgumentNullException(nameof(city));
        Province = province ?? throw new ArgumentNullException(nameof(province));
        PostalCode = postalCode ?? throw new ArgumentNullException(nameof(postalCode));
        Country = country;
    }
    
    public override string ToString() => $"{Street}, {City}, {Province} {PostalCode}, {Country}";
    
    protected override IEnumerable<object?> GetAtomicValues()
    {
        yield return Street;
        yield return City;
        yield return Province;
        yield return PostalCode;
        yield return Country;
    }
}
```

```csharp
// src/Modules/Orders/ShopThai.Orders.Domain/Events/OrderEvents.cs
using ShopThai.Shared.Domain;

namespace ShopThai.Orders.Domain.Events;

public record OrderConfirmedEvent(
    Guid OrderId,
    Guid CustomerId,
    Money TotalAmount) : DomainEvent;

public record OrderPaidEvent(
    Guid OrderId,
    string PaymentTransactionId) : DomainEvent;

public record OrderShippedEvent(
    Guid OrderId,
    Guid CustomerId,
    string TrackingNumber) : DomainEvent;

public record OrderCancelledEvent(
    Guid OrderId,
    Guid CustomerId,
    string Reason) : DomainEvent;
```

---

## Step 803: User Authentication และ JWT

### Users Domain

```csharp
// src/Modules/Users/ShopThai.Users.Domain/Entities/User.cs
using ShopThai.Shared.Domain;
using ShopThai.Users.Domain.ValueObjects;

namespace ShopThai.Users.Domain.Entities;

public class User : AggregateRoot<Guid>
{
    public Email Email { get; private set; }
    public string FirstName { get; private set; }
    public string LastName { get; private set; }
    public string PasswordHash { get; private set; }
    public UserRole Role { get; private set; }
    public bool IsActive { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? LastLoginAt { get; private set; }
    
    public string FullName => $"{FirstName} {LastName}";
    
    private User() : base(Guid.Empty) { }
    
    private User(Guid id, Email email, string firstName, string lastName, string passwordHash, UserRole role)
        : base(id)
    {
        Email = email;
        FirstName = firstName;
        LastName = lastName;
        PasswordHash = passwordHash;
        Role = role;
        IsActive = true;
        CreatedAt = DateTime.UtcNow;
    }
    
    public static User Create(Email email, string firstName, string lastName, string passwordHash)
    {
        return new User(Guid.NewGuid(), email, firstName, lastName, passwordHash, UserRole.Customer);
    }
    
    public static User CreateAdmin(Email email, string firstName, string lastName, string passwordHash)
    {
        return new User(Guid.NewGuid(), email, firstName, lastName, passwordHash, UserRole.Admin);
    }
    
    public void RecordLogin()
    {
        LastLoginAt = DateTime.UtcNow;
    }
    
    public void UpdateProfile(string firstName, string lastName)
    {
        FirstName = firstName;
        LastName = lastName;
    }
    
    public void ChangePassword(string newPasswordHash)
    {
        PasswordHash = newPasswordHash;
    }
    
    public void Deactivate()
    {
        IsActive = false;
    }
}

public enum UserRole
{
    Customer,
    Admin,
    SuperAdmin
}
```

```csharp
// src/Modules/Users/ShopThai.Users.Domain/ValueObjects/Email.cs
using ShopThai.Shared.Domain;
using System.Text.RegularExpressions;

namespace ShopThai.Users.Domain.ValueObjects;

public class Email : ValueObject
{
    private static readonly Regex EmailRegex = new(
        @"^[^@\s]+@[^@\s]+\.[^@\s]+$",
        RegexOptions.Compiled | RegexOptions.IgnoreCase);
    
    public string Value { get; }
    
    private Email() { }
    
    public Email(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Email ต้องไม่ว่าง");
        
        if (!EmailRegex.IsMatch(value))
            throw new ArgumentException($"รูปแบบ Email ไม่ถูกต้อง: {value}");
        
        Value = value.ToLowerInvariant();
    }
    
    public static implicit operator string(Email email) => email.Value;
    public static explicit operator Email(string email) => new(email);
    
    public override string ToString() => Value;
    
    protected override IEnumerable<object?> GetAtomicValues()
    {
        yield return Value;
    }
}
```

### JWT Authentication Service

```csharp
// src/Modules/Users/ShopThai.Users.Application/Services/JwtService.cs
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Security.Cryptography;
using System.Text;
using Microsoft.Extensions.Options;
using Microsoft.IdentityModel.Tokens;
using ShopThai.Users.Domain.Entities;

namespace ShopThai.Users.Application.Services;

public class JwtSettings
{
    public string Secret { get; set; } = string.Empty;
    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public int ExpirationMinutes { get; set; } = 60;
    public int RefreshTokenExpirationDays { get; set; } = 30;
}

public interface IJwtService
{
    string GenerateAccessToken(User user);
    string GenerateRefreshToken();
    ClaimsPrincipal? ValidateToken(string token);
}

public class JwtService : IJwtService
{
    private readonly JwtSettings _settings;
    
    public JwtService(IOptions<JwtSettings> settings)
    {
        _settings = settings.Value;
    }
    
    public string GenerateAccessToken(User user)
    {
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_settings.Secret));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var claims = new[]
        {
            new Claim(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new Claim(JwtRegisteredClaimNames.Email, user.Email.Value),
            new Claim(JwtRegisteredClaimNames.GivenName, user.FirstName),
            new Claim(JwtRegisteredClaimNames.FamilyName, user.LastName),
            new Claim(ClaimTypes.Role, user.Role.ToString()),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
        };
        
        var token = new JwtSecurityToken(
            issuer: _settings.Issuer,
            audience: _settings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_settings.ExpirationMinutes),
            signingCredentials: credentials);
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
    
    public string GenerateRefreshToken()
    {
        var randomBytes = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes);
    }
    
    public ClaimsPrincipal? ValidateToken(string token)
    {
        try
        {
            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_settings.Secret));
            var tokenHandler = new JwtSecurityTokenHandler();
            
            var validationParameters = new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = key,
                ValidateIssuer = true,
                ValidIssuer = _settings.Issuer,
                ValidateAudience = true,
                ValidAudience = _settings.Audience,
                ValidateLifetime = true,
                ClockSkew = TimeSpan.Zero
            };
            
            return tokenHandler.ValidateToken(token, validationParameters, out _);
        }
        catch
        {
            return null;
        }
    }
}
```

### Authentication Commands/Queries (CQRS)

```csharp
// src/Modules/Users/ShopThai.Users.Application/Commands/LoginCommand.cs
using MediatR;

namespace ShopThai.Users.Application.Commands;

public record LoginCommand(string Email, string Password) : IRequest<LoginResult>;

public record LoginResult(
    bool Success,
    string? AccessToken,
    string? RefreshToken,
    string? ErrorMessage);
```

```csharp
// src/Modules/Users/ShopThai.Users.Application/Commands/LoginCommandHandler.cs
using MediatR;
using ShopThai.Users.Domain.ValueObjects;
using ShopThai.Users.Application.Repositories;
using ShopThai.Users.Application.Services;

namespace ShopThai.Users.Application.Commands;

public class LoginCommandHandler : IRequestHandler<LoginCommand, LoginResult>
{
    private readonly IUserRepository _userRepository;
    private readonly IJwtService _jwtService;
    private readonly IPasswordHasher _passwordHasher;
    
    public LoginCommandHandler(
        IUserRepository userRepository,
        IJwtService jwtService,
        IPasswordHasher passwordHasher)
    {
        _userRepository = userRepository;
        _jwtService = jwtService;
        _passwordHasher = passwordHasher;
    }
    
    public async Task<LoginResult> Handle(LoginCommand request, CancellationToken cancellationToken)
    {
        var email = new Email(request.Email);
        var user = await _userRepository.GetByEmailAsync(email, cancellationToken);
        
        if (user is null || !user.IsActive)
            return new LoginResult(false, null, null, "Email หรือรหัสผ่านไม่ถูกต้อง");
        
        if (!_passwordHasher.Verify(request.Password, user.PasswordHash))
            return new LoginResult(false, null, null, "Email หรือรหัสผ่านไม่ถูกต้อง");
        
        user.RecordLogin();
        await _userRepository.UpdateAsync(user, cancellationToken);
        
        var accessToken = _jwtService.GenerateAccessToken(user);
        var refreshToken = _jwtService.GenerateRefreshToken();
        
        // เก็บ refresh token ใน database
        await _userRepository.SaveRefreshTokenAsync(user.Id, refreshToken, cancellationToken);
        
        return new LoginResult(true, accessToken, refreshToken, null);
    }
}
```

```csharp
// src/Modules/Users/ShopThai.Users.Application/Commands/RegisterCommand.cs
using MediatR;

namespace ShopThai.Users.Application.Commands;

public record RegisterCommand(
    string Email,
    string FirstName,
    string LastName,
    string Password,
    string ConfirmPassword) : IRequest<RegisterResult>;

public record RegisterResult(bool Success, Guid? UserId, string? ErrorMessage);
```

```csharp
// src/Modules/Users/ShopThai.Users.Application/Commands/RegisterCommandHandler.cs
using MediatR;
using ShopThai.Users.Domain.Entities;
using ShopThai.Users.Domain.ValueObjects;
using ShopThai.Users.Application.Repositories;
using ShopThai.Users.Application.Services;

namespace ShopThai.Users.Application.Commands;

public class RegisterCommandHandler : IRequestHandler<RegisterCommand, RegisterResult>
{
    private readonly IUserRepository _userRepository;
    private readonly IPasswordHasher _passwordHasher;
    
    public RegisterCommandHandler(IUserRepository userRepository, IPasswordHasher passwordHasher)
    {
        _userRepository = userRepository;
        _passwordHasher = passwordHasher;
    }
    
    public async Task<RegisterResult> Handle(RegisterCommand request, CancellationToken cancellationToken)
    {
        if (request.Password != request.ConfirmPassword)
            return new RegisterResult(false, null, "รหัสผ่านไม่ตรงกัน");
        
        if (request.Password.Length < 8)
            return new RegisterResult(false, null, "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร");
        
        var email = new Email(request.Email);
        var existingUser = await _userRepository.GetByEmailAsync(email, cancellationToken);
        
        if (existingUser is not null)
            return new RegisterResult(false, null, "Email นี้ถูกใช้งานแล้ว");
        
        var passwordHash = _passwordHasher.Hash(request.Password);
        var user = User.Create(email, request.FirstName, request.LastName, passwordHash);
        
        await _userRepository.AddAsync(user, cancellationToken);
        
        return new RegisterResult(true, user.Id, null);
    }
}
```

---

## Step 804: Product Catalog พร้อม Search

### CQRS สำหรับ Catalog

```csharp
// src/Modules/Catalog/ShopThai.Catalog.Application/Queries/SearchProductsQuery.cs
using MediatR;

namespace ShopThai.Catalog.Application.Queries;

public record SearchProductsQuery(
    string? SearchTerm,
    Guid? CategoryId,
    decimal? MinPrice,
    decimal? MaxPrice,
    string? SortBy,
    bool SortDescending = false,
    int Page = 1,
    int PageSize = 20) : IRequest<PagedResult<ProductDto>>;

public record ProductDto(
    Guid Id,
    string Name,
    string Description,
    decimal Price,
    string Currency,
    int StockQuantity,
    string ImageUrl,
    Guid CategoryId,
    string CategoryName,
    bool IsActive);

public record PagedResult<T>(
    IReadOnlyList<T> Items,
    int TotalCount,
    int Page,
    int PageSize)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasNextPage => Page < TotalPages;
    public bool HasPreviousPage => Page > 1;
}
```

```csharp
// src/Modules/Catalog/ShopThai.Catalog.Application/Queries/SearchProductsQueryHandler.cs
using MediatR;
using Microsoft.EntityFrameworkCore;
using ShopThai.Catalog.Infrastructure.Data;

namespace ShopThai.Catalog.Application.Queries;

public class SearchProductsQueryHandler 
    : IRequestHandler<SearchProductsQuery, PagedResult<ProductDto>>
{
    private readonly CatalogDbContext _dbContext;
    
    public SearchProductsQueryHandler(CatalogDbContext dbContext)
    {
        _dbContext = dbContext;
    }
    
    public async Task<PagedResult<ProductDto>> Handle(
        SearchProductsQuery request, 
        CancellationToken cancellationToken)
    {
        var query = _dbContext.Products
            .Include(p => p.Category)
            .Where(p => p.IsActive)
            .AsQueryable();
        
        // ค้นหาตามคำค้น
        if (!string.IsNullOrWhiteSpace(request.SearchTerm))
        {
            var searchTerm = request.SearchTerm.ToLower();
            query = query.Where(p => 
                p.Name.ToLower().Contains(searchTerm) ||
                p.Description.ToLower().Contains(searchTerm));
        }
        
        // กรองตาม Category
        if (request.CategoryId.HasValue)
            query = query.Where(p => p.CategoryId == request.CategoryId);
        
        // กรองตามราคา
        if (request.MinPrice.HasValue)
            query = query.Where(p => p.Price.Amount >= request.MinPrice.Value);
        
        if (request.MaxPrice.HasValue)
            query = query.Where(p => p.Price.Amount <= request.MaxPrice.Value);
        
        // เรียงลำดับ
        query = (request.SortBy?.ToLower(), request.SortDescending) switch
        {
            ("price", false) => query.OrderBy(p => p.Price.Amount),
            ("price", true) => query.OrderByDescending(p => p.Price.Amount),
            ("name", false) => query.OrderBy(p => p.Name),
            ("name", true) => query.OrderByDescending(p => p.Name),
            _ => query.OrderByDescending(p => p.CreatedAt)
        };
        
        var totalCount = await query.CountAsync(cancellationToken);
        
        var items = await query
            .Skip((request.Page - 1) * request.PageSize)
            .Take(request.PageSize)
            .Select(p => new ProductDto(
                p.Id,
                p.Name,
                p.Description,
                p.Price.Amount,
                p.Price.Currency,
                p.StockQuantity,
                p.ImageUrl,
                p.CategoryId,
                p.Category.Name,
                p.IsActive))
            .ToListAsync(cancellationToken);
        
        return new PagedResult<ProductDto>(items, totalCount, request.Page, request.PageSize);
    }
}
```

### Create Product Command

```csharp
// src/Modules/Catalog/ShopThai.Catalog.Application/Commands/CreateProductCommand.cs
using MediatR;

namespace ShopThai.Catalog.Application.Commands;

public record CreateProductCommand(
    string Name,
    string Description,
    decimal Price,
    int StockQuantity,
    string ImageUrl,
    Guid CategoryId) : IRequest<Guid>;
```

```csharp
// src/Modules/Catalog/ShopThai.Catalog.Application/Commands/CreateProductCommandHandler.cs
using MediatR;
using ShopThai.Catalog.Domain.Entities;
using ShopThai.Catalog.Domain.ValueObjects;
using ShopThai.Catalog.Application.Repositories;

namespace ShopThai.Catalog.Application.Commands;

public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, Guid>
{
    private readonly IProductRepository _productRepository;
    
    public CreateProductCommandHandler(IProductRepository productRepository)
    {
        _productRepository = productRepository;
    }
    
    public async Task<Guid> Handle(CreateProductCommand request, CancellationToken cancellationToken)
    {
        var price = Money.FromTHB(request.Price);
        
        var product = Product.Create(
            request.Name,
            request.Description,
            price,
            request.StockQuantity,
            request.ImageUrl,
            request.CategoryId);
        
        await _productRepository.AddAsync(product, cancellationToken);
        
        return product.Id;
    }
}
```

---

## Step 805: Shopping Cart ด้วย Redis

### Cart Models

```csharp
// src/Modules/Catalog/ShopThai.Catalog.Application/Cart/CartItem.cs
namespace ShopThai.Catalog.Application.Cart;

public class Cart
{
    public Guid UserId { get; set; }
    public List<CartItem> Items { get; set; } = new();
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
    
    public decimal TotalAmount => Items.Sum(i => i.Subtotal);
    public int TotalItems => Items.Sum(i => i.Quantity);
}

public class CartItem
{
    public Guid ProductId { get; set; }
    public string ProductName { get; set; } = string.Empty;
    public decimal UnitPrice { get; set; }
    public string Currency { get; set; } = "THB";
    public int Quantity { get; set; }
    public string ImageUrl { get; set; } = string.Empty;
    
    public decimal Subtotal => UnitPrice * Quantity;
}
```

### Redis Cart Service

```csharp
// src/Modules/Catalog/ShopThai.Catalog.Infrastructure/Services/RedisCartService.cs
using System.Text.Json;
using Microsoft.Extensions.Caching.Distributed;
using ShopThai.Catalog.Application.Cart;
using ShopThai.Catalog.Application.Services;

namespace ShopThai.Catalog.Infrastructure.Services;

public interface ICartService
{
    Task<Cart?> GetCartAsync(Guid userId, CancellationToken cancellationToken = default);
    Task SaveCartAsync(Cart cart, CancellationToken cancellationToken = default);
    Task<Cart> AddItemAsync(Guid userId, CartItem item, CancellationToken cancellationToken = default);
    Task<Cart> RemoveItemAsync(Guid userId, Guid productId, CancellationToken cancellationToken = default);
    Task<Cart> UpdateItemQuantityAsync(Guid userId, Guid productId, int quantity, CancellationToken cancellationToken = default);
    Task ClearCartAsync(Guid userId, CancellationToken cancellationToken = default);
}

public class RedisCartService : ICartService
{
    private readonly IDistributedCache _cache;
    private static readonly TimeSpan CartExpiry = TimeSpan.FromDays(7);
    
    public RedisCartService(IDistributedCache cache)
    {
        _cache = cache;
    }
    
    private static string GetCartKey(Guid userId) => $"cart:{userId}";
    
    public async Task<Cart?> GetCartAsync(Guid userId, CancellationToken cancellationToken = default)
    {
        var key = GetCartKey(userId);
        var json = await _cache.GetStringAsync(key, cancellationToken);
        
        if (json is null) return null;
        
        return JsonSerializer.Deserialize<Cart>(json);
    }
    
    public async Task SaveCartAsync(Cart cart, CancellationToken cancellationToken = default)
    {
        cart.UpdatedAt = DateTime.UtcNow;
        
        var key = GetCartKey(cart.UserId);
        var json = JsonSerializer.Serialize(cart);
        
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = CartExpiry
        };
        
        await _cache.SetStringAsync(key, json, options, cancellationToken);
    }
    
    public async Task<Cart> AddItemAsync(Guid userId, CartItem item, CancellationToken cancellationToken = default)
    {
        var cart = await GetCartAsync(userId, cancellationToken) 
                   ?? new Cart { UserId = userId };
        
        var existingItem = cart.Items.FirstOrDefault(i => i.ProductId == item.ProductId);
        
        if (existingItem is not null)
        {
            existingItem.Quantity += item.Quantity;
        }
        else
        {
            cart.Items.Add(item);
        }
        
        await SaveCartAsync(cart, cancellationToken);
        return cart;
    }
    
    public async Task<Cart> RemoveItemAsync(Guid userId, Guid productId, CancellationToken cancellationToken = default)
    {
        var cart = await GetCartAsync(userId, cancellationToken) 
                   ?? new Cart { UserId = userId };
        
        cart.Items.RemoveAll(i => i.ProductId == productId);
        await SaveCartAsync(cart, cancellationToken);
        return cart;
    }
    
    public async Task<Cart> UpdateItemQuantityAsync(
        Guid userId, Guid productId, int quantity, CancellationToken cancellationToken = default)
    {
        var cart = await GetCartAsync(userId, cancellationToken) 
                   ?? new Cart { UserId = userId };
        
        var item = cart.Items.FirstOrDefault(i => i.ProductId == productId);
        
        if (item is null)
            throw new InvalidOperationException("ไม่พบสินค้านี้ในตะกร้า");
        
        if (quantity <= 0)
        {
            cart.Items.Remove(item);
        }
        else
        {
            item.Quantity = quantity;
        }
        
        await SaveCartAsync(cart, cancellationToken);
        return cart;
    }
    
    public async Task ClearCartAsync(Guid userId, CancellationToken cancellationToken = default)
    {
        var key = GetCartKey(userId);
        await _cache.RemoveAsync(key, cancellationToken);
    }
}
```

---

## Step 806: Order Placement พร้อม Domain Events

### Place Order Command

```csharp
// src/Modules/Orders/ShopThai.Orders.Application/Commands/PlaceOrderCommand.cs
using MediatR;

namespace ShopThai.Orders.Application.Commands;

public record PlaceOrderCommand(
    Guid CustomerId,
    string Street,
    string City,
    string Province,
    string PostalCode,
    List<PlaceOrderItemDto> Items) : IRequest<PlaceOrderResult>;

public record PlaceOrderItemDto(
    Guid ProductId,
    string ProductName,
    decimal UnitPrice,
    int Quantity);

public record PlaceOrderResult(bool Success, Guid? OrderId, string? ErrorMessage);
```

```csharp
// src/Modules/Orders/ShopThai.Orders.Application/Commands/PlaceOrderCommandHandler.cs
using MediatR;
using ShopThai.Orders.Domain.Entities;
using ShopThai.Orders.Domain.ValueObjects;
using ShopThai.Orders.Application.Repositories;
using ShopThai.Shared.Domain;

namespace ShopThai.Orders.Application.Commands;

public class PlaceOrderCommandHandler : IRequestHandler<PlaceOrderCommand, PlaceOrderResult>
{
    private readonly IOrderRepository _orderRepository;
    private readonly IPublisher _publisher;
    
    public PlaceOrderCommandHandler(IOrderRepository orderRepository, IPublisher publisher)
    {
        _orderRepository = orderRepository;
        _publisher = publisher;
    }
    
    public async Task<PlaceOrderResult> Handle(PlaceOrderCommand request, CancellationToken cancellationToken)
    {
        if (!request.Items.Any())
            return new PlaceOrderResult(false, null, "ต้องมีสินค้าอย่างน้อย 1 รายการ");
        
        var shippingAddress = new Address(
            request.Street,
            request.City,
            request.Province,
            request.PostalCode);
        
        var order = Order.Create(request.CustomerId, shippingAddress);
        
        foreach (var item in request.Items)
        {
            var unitPrice = new Money(item.UnitPrice);
            order.AddItem(item.ProductId, item.ProductName, unitPrice, item.Quantity);
        }
        
        order.Confirm();
        
        await _orderRepository.AddAsync(order, cancellationToken);
        
        // Publish domain events
        foreach (var domainEvent in order.DomainEvents)
        {
            await _publisher.Publish(domainEvent, cancellationToken);
        }
        
        order.ClearDomainEvents();
        
        return new PlaceOrderResult(true, order.Id, null);
    }
}
```

### Domain Event Handlers

```csharp
// src/Modules/Orders/ShopThai.Orders.Application/EventHandlers/OrderConfirmedEventHandler.cs
using MediatR;
using Microsoft.Extensions.Logging;
using ShopThai.Orders.Domain.Events;

namespace ShopThai.Orders.Application.EventHandlers;

public class OrderConfirmedEventHandler : INotificationHandler<OrderConfirmedEvent>
{
    private readonly ILogger<OrderConfirmedEventHandler> _logger;
    private readonly IEmailNotificationService _emailService;
    
    public OrderConfirmedEventHandler(
        ILogger<OrderConfirmedEventHandler> logger,
        IEmailNotificationService emailService)
    {
        _logger = logger;
        _emailService = emailService;
    }
    
    public async Task Handle(OrderConfirmedEvent notification, CancellationToken cancellationToken)
    {
        _logger.LogInformation(
            "Order {OrderId} confirmed for customer {CustomerId}. Total: {TotalAmount}",
            notification.OrderId,
            notification.CustomerId,
            notification.TotalAmount);
        
        // ส่ง email แจ้งเตือน
        await _emailService.SendOrderConfirmationAsync(
            notification.OrderId,
            notification.CustomerId,
            cancellationToken);
    }
}
```

---

## Step 807: Payment Integration

### Payment Domain

```csharp
// src/Modules/Payments/ShopThai.Payments.Domain/Entities/Payment.cs
using ShopThai.Shared.Domain;
using ShopThai.Payments.Domain.Events;

namespace ShopThai.Payments.Domain.Entities;

public class Payment : AggregateRoot<Guid>
{
    public Guid OrderId { get; private set; }
    public Guid CustomerId { get; private set; }
    public decimal Amount { get; private set; }
    public string Currency { get; private set; }
    public PaymentStatus Status { get; private set; }
    public string? TransactionId { get; private set; }
    public string? FailureReason { get; private set; }
    public PaymentMethod Method { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? ProcessedAt { get; private set; }
    
    private Payment() : base(Guid.Empty) { }
    
    private Payment(Guid id, Guid orderId, Guid customerId, decimal amount, string currency, PaymentMethod method)
        : base(id)
    {
        OrderId = orderId;
        CustomerId = customerId;
        Amount = amount;
        Currency = currency;
        Method = method;
        Status = PaymentStatus.Pending;
        CreatedAt = DateTime.UtcNow;
    }
    
    public static Payment Create(
        Guid orderId,
        Guid customerId,
        decimal amount,
        string currency,
        PaymentMethod method)
    {
        return new Payment(Guid.NewGuid(), orderId, customerId, amount, currency, method);
    }
    
    public void Succeed(string transactionId)
    {
        Status = PaymentStatus.Succeeded;
        TransactionId = transactionId;
        ProcessedAt = DateTime.UtcNow;
        
        RaiseDomainEvent(new PaymentSucceededEvent(Id, OrderId, CustomerId, Amount, transactionId));
    }
    
    public void Fail(string reason)
    {
        Status = PaymentStatus.Failed;
        FailureReason = reason;
        ProcessedAt = DateTime.UtcNow;
        
        RaiseDomainEvent(new PaymentFailedEvent(Id, OrderId, reason));
    }
    
    public void Refund()
    {
        if (Status != PaymentStatus.Succeeded)
            throw new InvalidOperationException("สามารถคืนเงินได้เฉพาะการชำระเงินที่สำเร็จแล้ว");
        
        Status = PaymentStatus.Refunded;
        ProcessedAt = DateTime.UtcNow;
        
        RaiseDomainEvent(new PaymentRefundedEvent(Id, OrderId, CustomerId, Amount));
    }
}

public enum PaymentStatus
{
    Pending,
    Processing,
    Succeeded,
    Failed,
    Refunded
}

public enum PaymentMethod
{
    CreditCard,
    PromptPay,
    BankTransfer,
    TrueWallet
}
```

```csharp
// src/Modules/Payments/ShopThai.Payments.Domain/Events/PaymentEvents.cs
using ShopThai.Shared.Domain;

namespace ShopThai.Payments.Domain.Events;

public record PaymentSucceededEvent(
    Guid PaymentId,
    Guid OrderId,
    Guid CustomerId,
    decimal Amount,
    string TransactionId) : DomainEvent;

public record PaymentFailedEvent(
    Guid PaymentId,
    Guid OrderId,
    string FailureReason) : DomainEvent;

public record PaymentRefundedEvent(
    Guid PaymentId,
    Guid OrderId,
    Guid CustomerId,
    decimal Amount) : DomainEvent;
```

### Mock Payment Gateway

```csharp
// src/Modules/Payments/ShopThai.Payments.Infrastructure/Services/MockPaymentGateway.cs
using Microsoft.Extensions.Logging;
using ShopThai.Payments.Application.Services;

namespace ShopThai.Payments.Infrastructure.Services;

public interface IPaymentGateway
{
    Task<PaymentGatewayResult> ProcessPaymentAsync(
        Guid paymentId,
        decimal amount,
        string currency,
        PaymentGatewayRequest request,
        CancellationToken cancellationToken = default);
}

public record PaymentGatewayRequest(
    string CardNumber,
    string ExpiryMonth,
    string ExpiryYear,
    string Cvv,
    string CardholderName);

public record PaymentGatewayResult(
    bool Success,
    string? TransactionId,
    string? ErrorMessage);

public class MockPaymentGateway : IPaymentGateway
{
    private readonly ILogger<MockPaymentGateway> _logger;
    
    public MockPaymentGateway(ILogger<MockPaymentGateway> logger)
    {
        _logger = logger;
    }
    
    public async Task<PaymentGatewayResult> ProcessPaymentAsync(
        Guid paymentId,
        decimal amount,
        string currency,
        PaymentGatewayRequest request,
        CancellationToken cancellationToken = default)
    {
        // จำลองการเรียก API จริง
        await Task.Delay(500, cancellationToken);
        
        _logger.LogInformation(
            "Processing payment {PaymentId} for {Amount} {Currency}",
            paymentId, amount, currency);
        
        // บัตรทดสอบ: 4000000000000002 = ปฏิเสธ
        if (request.CardNumber == "4000000000000002")
        {
            return new PaymentGatewayResult(false, null, "บัตรถูกปฏิเสธ");
        }
        
        // บัตรทดสอบอื่นๆ = อนุมัติ
        var transactionId = $"TXN{Guid.NewGuid():N}".ToUpper()[..20];
        
        _logger.LogInformation(
            "Payment {PaymentId} succeeded with transaction {TransactionId}",
            paymentId, transactionId);
        
        return new PaymentGatewayResult(true, transactionId, null);
    }
}
```

### Process Payment Command

```csharp
// src/Modules/Payments/ShopThai.Payments.Application/Commands/ProcessPaymentCommand.cs
using MediatR;

namespace ShopThai.Payments.Application.Commands;

public record ProcessPaymentCommand(
    Guid OrderId,
    Guid CustomerId,
    decimal Amount,
    string CardNumber,
    string ExpiryMonth,
    string ExpiryYear,
    string Cvv,
    string CardholderName) : IRequest<ProcessPaymentResult>;

public record ProcessPaymentResult(
    bool Success,
    Guid? PaymentId,
    string? TransactionId,
    string? ErrorMessage);
```

```csharp
// src/Modules/Payments/ShopThai.Payments.Application/Commands/ProcessPaymentCommandHandler.cs
using MediatR;
using ShopThai.Payments.Domain.Entities;
using ShopThai.Payments.Application.Repositories;
using ShopThai.Payments.Infrastructure.Services;

namespace ShopThai.Payments.Application.Commands;

public class ProcessPaymentCommandHandler : IRequestHandler<ProcessPaymentCommand, ProcessPaymentResult>
{
    private readonly IPaymentRepository _paymentRepository;
    private readonly IPaymentGateway _paymentGateway;
    private readonly IPublisher _publisher;
    
    public ProcessPaymentCommandHandler(
        IPaymentRepository paymentRepository,
        IPaymentGateway paymentGateway,
        IPublisher publisher)
    {
        _paymentRepository = paymentRepository;
        _paymentGateway = paymentGateway;
        _publisher = publisher;
    }
    
    public async Task<ProcessPaymentResult> Handle(ProcessPaymentCommand request, CancellationToken cancellationToken)
    {
        var payment = Payment.Create(
            request.OrderId,
            request.CustomerId,
            request.Amount,
            "THB",
            PaymentMethod.CreditCard);
        
        await _paymentRepository.AddAsync(payment, cancellationToken);
        
        var gatewayRequest = new PaymentGatewayRequest(
            request.CardNumber,
            request.ExpiryMonth,
            request.ExpiryYear,
            request.Cvv,
            request.CardholderName);
        
        var result = await _paymentGateway.ProcessPaymentAsync(
            payment.Id,
            payment.Amount,
            payment.Currency,
            gatewayRequest,
            cancellationToken);
        
        if (result.Success)
        {
            payment.Succeed(result.TransactionId!);
        }
        else
        {
            payment.Fail(result.ErrorMessage!);
        }
        
        await _paymentRepository.UpdateAsync(payment, cancellationToken);
        
        // Publish domain events
        foreach (var domainEvent in payment.DomainEvents)
        {
            await _publisher.Publish(domainEvent, cancellationToken);
        }
        
        payment.ClearDomainEvents();
        
        return result.Success
            ? new ProcessPaymentResult(true, payment.Id, result.TransactionId, null)
            : new ProcessPaymentResult(false, payment.Id, null, result.ErrorMessage);
    }
}
```

### Webhook Handler

```csharp
// src/ShopThai.API/Controllers/PaymentWebhookController.cs
using Microsoft.AspNetCore.Mvc;
using System.Text.Json;

namespace ShopThai.API.Controllers;

[ApiController]
[Route("api/webhooks/payment")]
public class PaymentWebhookController : ControllerBase
{
    private readonly ILogger<PaymentWebhookController> _logger;
    
    public PaymentWebhookController(ILogger<PaymentWebhookController> logger)
    {
        _logger = logger;
    }
    
    [HttpPost]
    public async Task<IActionResult> HandleWebhook()
    {
        using var reader = new StreamReader(Request.Body);
        var payload = await reader.ReadToEndAsync();
        
        _logger.LogInformation("Received payment webhook: {Payload}", payload);
        
        try
        {
            var webhookEvent = JsonSerializer.Deserialize<PaymentWebhookEvent>(payload);
            
            if (webhookEvent is null)
                return BadRequest("Invalid webhook payload");
            
            // ตรวจสอบ signature (ในระบบจริงต้องทำ)
            // if (!ValidateSignature(payload, Request.Headers["X-Webhook-Signature"]))
            //     return Unauthorized();
            
            switch (webhookEvent.EventType)
            {
                case "payment.succeeded":
                    _logger.LogInformation("Payment {PaymentId} succeeded", webhookEvent.PaymentId);
                    break;
                    
                case "payment.failed":
                    _logger.LogWarning("Payment {PaymentId} failed: {Reason}", 
                        webhookEvent.PaymentId, webhookEvent.Reason);
                    break;
                    
                default:
                    _logger.LogWarning("Unknown webhook event type: {Type}", webhookEvent.EventType);
                    break;
            }
            
            return Ok(new { received = true });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing webhook");
            return StatusCode(500, "Error processing webhook");
        }
    }
}

public record PaymentWebhookEvent(
    string EventType,
    Guid PaymentId,
    string? Reason,
    DateTime Timestamp);
```

---

## Step 808: Admin Dashboard พร้อม SignalR

### SignalR Hub

```csharp
// src/ShopThai.API/Hubs/AdminDashboardHub.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.SignalR;

namespace ShopThai.API.Hubs;

[Authorize(Roles = "Admin,SuperAdmin")]
public class AdminDashboardHub : Hub
{
    private readonly ILogger<AdminDashboardHub> _logger;
    
    public AdminDashboardHub(ILogger<AdminDashboardHub> logger)
    {
        _logger = logger;
    }
    
    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier;
        _logger.LogInformation("Admin {UserId} connected to dashboard", userId);
        
        await Groups.AddToGroupAsync(Context.ConnectionId, "AdminDashboard");
        await base.OnConnectedAsync();
    }
    
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, "AdminDashboard");
        await base.OnDisconnectedAsync(exception);
    }
    
    public async Task RequestDashboardStats()
    {
        // ส่งข้อมูลสถิติกลับไปให้ client
        await Clients.Caller.SendAsync("ReceiveDashboardStats", new
        {
            message = "Stats requested - will be sent shortly"
        });
    }
}
```

```csharp
// src/ShopThai.API/Services/DashboardService.cs
using Microsoft.AspNetCore.SignalR;
using ShopThai.API.Hubs;

namespace ShopThai.API.Services;

public class DashboardStats
{
    public int TotalOrders { get; set; }
    public decimal TotalRevenue { get; set; }
    public int NewCustomers { get; set; }
    public int ActiveProducts { get; set; }
    public List<RecentOrder> RecentOrders { get; set; } = new();
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
}

public record RecentOrder(
    string OrderNumber,
    string CustomerName,
    decimal Amount,
    string Status,
    DateTime CreatedAt);

public interface IDashboardService
{
    Task<DashboardStats> GetStatsAsync(CancellationToken cancellationToken = default);
    Task BroadcastStatsAsync(CancellationToken cancellationToken = default);
}

public class DashboardService : IDashboardService
{
    private readonly IHubContext<AdminDashboardHub> _hubContext;
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<DashboardService> _logger;
    
    public DashboardService(
        IHubContext<AdminDashboardHub> hubContext,
        IServiceProvider serviceProvider,
        ILogger<DashboardService> logger)
    {
        _hubContext = hubContext;
        _serviceProvider = serviceProvider;
        _logger = logger;
    }
    
    public async Task<DashboardStats> GetStatsAsync(CancellationToken cancellationToken = default)
    {
        using var scope = _serviceProvider.CreateScope();
        
        // ดึงข้อมูลจาก Database
        // ในระบบจริงจะ query จาก DbContext
        return new DashboardStats
        {
            TotalOrders = 1250,
            TotalRevenue = 2_850_000.00m,
            NewCustomers = 45,
            ActiveProducts = 320,
            RecentOrders = new List<RecentOrder>
            {
                new("ST20241001001", "สมชาย ใจดี", 1500.00m, "Confirmed", DateTime.UtcNow.AddMinutes(-5)),
                new("ST20241001002", "สมหญิง รักดี", 2800.00m, "Paid", DateTime.UtcNow.AddMinutes(-10)),
                new("ST20241001003", "วิชัย มีสุข", 850.00m, "Shipped", DateTime.UtcNow.AddMinutes(-15))
            }
        };
    }
    
    public async Task BroadcastStatsAsync(CancellationToken cancellationToken = default)
    {
        try
        {
            var stats = await GetStatsAsync(cancellationToken);
            
            await _hubContext.Clients
                .Group("AdminDashboard")
                .SendAsync("ReceiveDashboardStats", stats, cancellationToken);
            
            _logger.LogDebug("Dashboard stats broadcasted to admin users");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error broadcasting dashboard stats");
        }
    }
}
```

### Background Service สำหรับ Dashboard

```csharp
// src/ShopThai.API/Workers/DashboardWorker.cs
namespace ShopThai.API.Workers;

public class DashboardWorker : BackgroundService
{
    private readonly IDashboardService _dashboardService;
    private readonly ILogger<DashboardWorker> _logger;
    private static readonly TimeSpan UpdateInterval = TimeSpan.FromSeconds(30);
    
    public DashboardWorker(IDashboardService dashboardService, ILogger<DashboardWorker> logger)
    {
        _dashboardService = dashboardService;
        _logger = logger;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Dashboard worker started");
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await _dashboardService.BroadcastStatsAsync(stoppingToken);
            }
            catch (OperationCanceledException)
            {
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in dashboard worker");
            }
            
            await Task.Delay(UpdateInterval, stoppingToken);
        }
        
        _logger.LogInformation("Dashboard worker stopped");
    }
}
```

---

## Step 809: Email Notifications ด้วย Background Worker และ Queue

### Email Queue Models

```csharp
// src/ShopThai.Shared/Messaging/EmailMessage.cs
namespace ShopThai.Shared.Messaging;

public class EmailMessage
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string To { get; set; } = string.Empty;
    public string Subject { get; set; } = string.Empty;
    public string Body { get; set; } = string.Empty;
    public bool IsHtml { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public int RetryCount { get; set; } = 0;
    public EmailTemplate Template { get; set; }
    public Dictionary<string, string> TemplateData { get; set; } = new();
}

public enum EmailTemplate
{
    OrderConfirmation,
    OrderShipped,
    OrderCancelled,
    PaymentSucceeded,
    PaymentFailed,
    Welcome,
    PasswordReset
}
```

### Email Service Interface

```csharp
// src/ShopThai.Shared/Services/IEmailService.cs
namespace ShopThai.Shared.Services;

public interface IEmailService
{
    Task SendAsync(EmailMessage message, CancellationToken cancellationToken = default);
    Task EnqueueAsync(EmailMessage message, CancellationToken cancellationToken = default);
}
```

### Email Worker ด้วย Channel

```csharp
// src/ShopThai.API/Workers/EmailWorker.cs
using System.Threading.Channels;
using ShopThai.Shared.Messaging;
using ShopThai.Shared.Services;

namespace ShopThai.API.Workers;

public class EmailWorker : BackgroundService
{
    private readonly Channel<EmailMessage> _emailChannel;
    private readonly ILogger<EmailWorker> _logger;
    private readonly IServiceProvider _serviceProvider;
    
    public EmailWorker(
        Channel<EmailMessage> emailChannel,
        ILogger<EmailWorker> logger,
        IServiceProvider serviceProvider)
    {
        _emailChannel = emailChannel;
        _logger = logger;
        _serviceProvider = serviceProvider;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Email worker started");
        
        await foreach (var emailMessage in _emailChannel.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                await ProcessEmailAsync(emailMessage, stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing email {EmailId}", emailMessage.Id);
                
                // Retry logic
                if (emailMessage.RetryCount < 3)
                {
                    emailMessage.RetryCount++;
                    await _emailChannel.Writer.WriteAsync(emailMessage, stoppingToken);
                }
            }
        }
    }
    
    private async Task ProcessEmailAsync(EmailMessage message, CancellationToken cancellationToken)
    {
        using var scope = _serviceProvider.CreateScope();
        var emailSender = scope.ServiceProvider.GetRequiredService<IEmailSender>();
        
        // Render template
        var (subject, body) = RenderTemplate(message);
        
        _logger.LogInformation(
            "Sending {Template} email to {To}",
            message.Template, message.To);
        
        await emailSender.SendAsync(message.To, subject, body, cancellationToken);
        
        _logger.LogInformation("Email {EmailId} sent successfully", message.Id);
    }
    
    private static (string Subject, string Body) RenderTemplate(EmailMessage message)
    {
        return message.Template switch
        {
            EmailTemplate.OrderConfirmation => (
                "ยืนยันการสั่งซื้อของคุณ",
                $"""
                <h2>ขอบคุณสำหรับการสั่งซื้อ!</h2>
                <p>หมายเลขออร์เดอร์: {message.TemplateData.GetValueOrDefault("orderNumber")}</p>
                <p>ยอดรวม: {message.TemplateData.GetValueOrDefault("totalAmount")} บาท</p>
                """),
            
            EmailTemplate.OrderShipped => (
                "สินค้าของคุณถูกจัดส่งแล้ว!",
                $"""
                <h2>สินค้าถูกจัดส่งแล้ว</h2>
                <p>เลขติดตามพัสดุ: {message.TemplateData.GetValueOrDefault("trackingNumber")}</p>
                """),
            
            EmailTemplate.Welcome => (
                "ยินดีต้อนรับสู่ ShopThai!",
                $"""
                <h2>ยินดีต้อนรับ {message.TemplateData.GetValueOrDefault("firstName")}!</h2>
                <p>ขอบคุณที่สมัครสมาชิกกับเรา</p>
                """),
            
            _ => ("แจ้งเตือนจาก ShopThai", message.Body)
        };
    }
}

public interface IEmailSender
{
    Task SendAsync(string to, string subject, string body, CancellationToken cancellationToken = default);
}

// Mock implementation สำหรับ Development
public class MockEmailSender : IEmailSender
{
    private readonly ILogger<MockEmailSender> _logger;
    
    public MockEmailSender(ILogger<MockEmailSender> logger)
    {
        _logger = logger;
    }
    
    public Task SendAsync(string to, string subject, string body, CancellationToken cancellationToken = default)
    {
        _logger.LogInformation(
            "[MOCK EMAIL] To: {To} | Subject: {Subject}",
            to, subject);
        
        return Task.CompletedTask;
    }
}
```

### MassTransit Integration สำหรับ Domain Events

```csharp
// src/ShopThai.API/Messaging/OrderConfirmedConsumer.cs
using MassTransit;
using System.Threading.Channels;
using ShopThai.Orders.Domain.Events;
using ShopThai.Shared.Messaging;

namespace ShopThai.API.Messaging;

public class OrderConfirmedConsumer : IConsumer<OrderConfirmedEvent>
{
    private readonly Channel<EmailMessage> _emailChannel;
    private readonly ILogger<OrderConfirmedConsumer> _logger;
    
    public OrderConfirmedConsumer(
        Channel<EmailMessage> emailChannel,
        ILogger<OrderConfirmedConsumer> logger)
    {
        _emailChannel = emailChannel;
        _logger = logger;
    }
    
    public async Task Consume(ConsumeContext<OrderConfirmedEvent> context)
    {
        var orderConfirmed = context.Message;
        
        _logger.LogInformation(
            "Consuming OrderConfirmedEvent for order {OrderId}",
            orderConfirmed.OrderId);
        
        var emailMessage = new EmailMessage
        {
            To = "customer@example.com", // ในระบบจริงดึงจาก customer service
            Template = EmailTemplate.OrderConfirmation,
            TemplateData = new Dictionary<string, string>
            {
                ["orderNumber"] = orderConfirmed.OrderId.ToString(),
                ["totalAmount"] = orderConfirmed.TotalAmount.Amount.ToString("N2")
            }
        };
        
        await _emailChannel.Writer.WriteAsync(emailMessage, context.CancellationToken);
    }
}
```

---

## Step 810: Full Solution Structure และ Project Setup

### EF Core DbContexts

```csharp
// src/Modules/Catalog/ShopThai.Catalog.Infrastructure/Data/CatalogDbContext.cs
using Microsoft.EntityFrameworkCore;
using ShopThai.Catalog.Domain.Entities;

namespace ShopThai.Catalog.Infrastructure.Data;

public class CatalogDbContext : DbContext
{
    public CatalogDbContext(DbContextOptions<CatalogDbContext> options) : base(options) { }
    
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasDefaultSchema("catalog");
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(CatalogDbContext).Assembly);
    }
}
```

```csharp
// src/Modules/Catalog/ShopThai.Catalog.Infrastructure/Data/Configurations/ProductConfiguration.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using ShopThai.Catalog.Domain.Entities;

namespace ShopThai.Catalog.Infrastructure.Data.Configurations;

public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");
        
        builder.HasKey(p => p.Id);
        
        builder.Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(200);
        
        builder.Property(p => p.Description)
            .HasMaxLength(2000);
        
        builder.OwnsOne(p => p.Price, price =>
        {
            price.Property(m => m.Amount)
                .HasColumnName("Price")
                .HasColumnType("decimal(18,2)")
                .IsRequired();
            
            price.Property(m => m.Currency)
                .HasColumnName("Currency")
                .HasMaxLength(3)
                .IsRequired();
        });
        
        builder.Property(p => p.StockQuantity).IsRequired();
        builder.Property(p => p.ImageUrl).HasMaxLength(500);
        builder.Property(p => p.IsActive).IsRequired();
        builder.Property(p => p.CreatedAt).IsRequired();
        
        builder.HasIndex(p => p.Name);
        builder.HasIndex(p => p.CategoryId);
        builder.HasIndex(p => p.IsActive);
    }
}
```

```csharp
// src/Modules/Orders/ShopThai.Orders.Infrastructure/Data/OrdersDbContext.cs
using Microsoft.EntityFrameworkCore;
using ShopThai.Orders.Domain.Entities;

namespace ShopThai.Orders.Infrastructure.Data;

public class OrdersDbContext : DbContext
{
    public OrdersDbContext(DbContextOptions<OrdersDbContext> options) : base(options) { }
    
    public DbSet<Order> Orders => Set<Order>();
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasDefaultSchema("orders");
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(OrdersDbContext).Assembly);
    }
}
```

```csharp
// src/Modules/Orders/ShopThai.Orders.Infrastructure/Data/Configurations/OrderConfiguration.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using ShopThai.Orders.Domain.Entities;

namespace ShopThai.Orders.Infrastructure.Data.Configurations;

public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");
        
        builder.HasKey(o => o.Id);
        
        builder.Property(o => o.OrderNumber)
            .IsRequired()
            .HasMaxLength(30);
        
        builder.HasIndex(o => o.OrderNumber).IsUnique();
        
        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(20);
        
        builder.OwnsOne(o => o.ShippingAddress, address =>
        {
            address.Property(a => a.Street).HasColumnName("Street").HasMaxLength(200);
            address.Property(a => a.City).HasColumnName("City").HasMaxLength(100);
            address.Property(a => a.Province).HasColumnName("Province").HasMaxLength(100);
            address.Property(a => a.PostalCode).HasColumnName("PostalCode").HasMaxLength(10);
            address.Property(a => a.Country).HasColumnName("Country").HasMaxLength(50);
        });
        
        builder.OwnsOne(o => o.TotalAmount, money =>
        {
            money.Property(m => m.Amount).HasColumnName("TotalAmount").HasColumnType("decimal(18,2)");
            money.Property(m => m.Currency).HasColumnName("TotalAmountCurrency").HasMaxLength(3);
        });
        
        // Configure Items as owned collection
        builder.OwnsMany(o => o.Items, item =>
        {
            item.ToTable("OrderItems");
            item.HasKey(i => i.Id);
            item.Property(i => i.ProductName).HasMaxLength(200);
            item.OwnsOne(i => i.UnitPrice, price =>
            {
                price.Property(m => m.Amount).HasColumnName("UnitPrice").HasColumnType("decimal(18,2)");
                price.Property(m => m.Currency).HasColumnName("Currency").HasMaxLength(3);
            });
        });
    }
}
```

### Dependency Injection Registration

```csharp
// src/Modules/Catalog/ShopThai.Catalog.Infrastructure/DependencyInjection.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using ShopThai.Catalog.Application.Repositories;
using ShopThai.Catalog.Infrastructure.Data;
using ShopThai.Catalog.Infrastructure.Repositories;
using ShopThai.Catalog.Infrastructure.Services;

namespace ShopThai.Catalog.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddCatalogInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<CatalogDbContext>(options =>
            options.UseNpgsql(configuration.GetConnectionString("CatalogDb")));
        
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<ICategoryRepository, CategoryRepository>();
        services.AddScoped<ICartService, RedisCartService>();
        
        return services;
    }
}
```

```csharp
// src/Modules/Catalog/ShopThai.Catalog.Application/DependencyInjection.cs
using Microsoft.Extensions.DependencyInjection;
using System.Reflection;

namespace ShopThai.Catalog.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddCatalogApplication(this IServiceCollection services)
    {
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));
        
        return services;
    }
}
```

### API Controllers

```csharp
// src/ShopThai.API/Controllers/ProductsController.cs
using MediatR;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using ShopThai.Catalog.Application.Commands;
using ShopThai.Catalog.Application.Queries;

namespace ShopThai.API.Controllers;

[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;
    
    public ProductsController(IMediator mediator)
    {
        _mediator = mediator;
    }
    
    [HttpGet]
    public async Task<IActionResult> Search([FromQuery] SearchProductsQuery query)
    {
        var result = await _mediator.Send(query);
        return Ok(result);
    }
    
    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetById(Guid id)
    {
        var result = await _mediator.Send(new GetProductByIdQuery(id));
        
        if (result is null) return NotFound();
        return Ok(result);
    }
    
    [HttpPost]
    [Authorize(Roles = "Admin,SuperAdmin")]
    public async Task<IActionResult> Create([FromBody] CreateProductCommand command)
    {
        var productId = await _mediator.Send(command);
        return CreatedAtAction(nameof(GetById), new { id = productId }, new { id = productId });
    }
    
    [HttpPut("{id:guid}/price")]
    [Authorize(Roles = "Admin,SuperAdmin")]
    public async Task<IActionResult> UpdatePrice(Guid id, [FromBody] UpdateProductPriceCommand command)
    {
        var updatedCommand = command with { ProductId = id };
        await _mediator.Send(updatedCommand);
        return NoContent();
    }
    
    [HttpDelete("{id:guid}")]
    [Authorize(Roles = "Admin,SuperAdmin")]
    public async Task<IActionResult> Deactivate(Guid id)
    {
        await _mediator.Send(new DeactivateProductCommand(id));
        return NoContent();
    }
}
```

```csharp
// src/ShopThai.API/Controllers/CartController.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;
using ShopThai.Catalog.Application.Cart;
using ShopThai.Catalog.Infrastructure.Services;

namespace ShopThai.API.Controllers;

[ApiController]
[Route("api/cart")]
[Authorize]
public class CartController : ControllerBase
{
    private readonly ICartService _cartService;
    
    public CartController(ICartService cartService)
    {
        _cartService = cartService;
    }
    
    private Guid UserId => Guid.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);
    
    [HttpGet]
    public async Task<IActionResult> GetCart()
    {
        var cart = await _cartService.GetCartAsync(UserId);
        return Ok(cart ?? new Cart { UserId = UserId });
    }
    
    [HttpPost("items")]
    public async Task<IActionResult> AddItem([FromBody] AddToCartRequest request)
    {
        var item = new CartItem
        {
            ProductId = request.ProductId,
            ProductName = request.ProductName,
            UnitPrice = request.UnitPrice,
            Quantity = request.Quantity,
            ImageUrl = request.ImageUrl
        };
        
        var cart = await _cartService.AddItemAsync(UserId, item);
        return Ok(cart);
    }
    
    [HttpPut("items/{productId:guid}")]
    public async Task<IActionResult> UpdateItemQuantity(Guid productId, [FromBody] UpdateCartItemRequest request)
    {
        var cart = await _cartService.UpdateItemQuantityAsync(UserId, productId, request.Quantity);
        return Ok(cart);
    }
    
    [HttpDelete("items/{productId:guid}")]
    public async Task<IActionResult> RemoveItem(Guid productId)
    {
        var cart = await _cartService.RemoveItemAsync(UserId, productId);
        return Ok(cart);
    }
    
    [HttpDelete]
    public async Task<IActionResult> ClearCart()
    {
        await _cartService.ClearCartAsync(UserId);
        return NoContent();
    }
}

public record AddToCartRequest(
    Guid ProductId,
    string ProductName,
    decimal UnitPrice,
    int Quantity,
    string ImageUrl);

public record UpdateCartItemRequest(int Quantity);
```

```csharp
// src/ShopThai.API/Controllers/OrdersController.cs
using MediatR;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;
using ShopThai.Orders.Application.Commands;
using ShopThai.Orders.Application.Queries;

namespace ShopThai.API.Controllers;

[ApiController]
[Route("api/orders")]
[Authorize]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;
    
    public OrdersController(IMediator mediator)
    {
        _mediator = mediator;
    }
    
    private Guid CustomerId => Guid.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);
    
    [HttpGet]
    public async Task<IActionResult> GetMyOrders([FromQuery] int page = 1, [FromQuery] int pageSize = 10)
    {
        var query = new GetCustomerOrdersQuery(CustomerId, page, pageSize);
        var result = await _mediator.Send(query);
        return Ok(result);
    }
    
    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetById(Guid id)
    {
        var result = await _mediator.Send(new GetOrderByIdQuery(id, CustomerId));
        if (result is null) return NotFound();
        return Ok(result);
    }
    
    [HttpPost]
    public async Task<IActionResult> PlaceOrder([FromBody] PlaceOrderRequest request)
    {
        var command = new PlaceOrderCommand(
            CustomerId,
            request.Street,
            request.City,
            request.Province,
            request.PostalCode,
            request.Items.Select(i => new PlaceOrderItemDto(
                i.ProductId,
                i.ProductName,
                i.UnitPrice,
                i.Quantity)).ToList());
        
        var result = await _mediator.Send(command);
        
        if (!result.Success)
            return BadRequest(new { error = result.ErrorMessage });
        
        return CreatedAtAction(nameof(GetById), new { id = result.OrderId }, new { id = result.OrderId });
    }
    
    [HttpPost("{id:guid}/cancel")]
    public async Task<IActionResult> Cancel(Guid id, [FromBody] CancelOrderRequest request)
    {
        await _mediator.Send(new CancelOrderCommand(id, CustomerId, request.Reason));
        return NoContent();
    }
}

public record PlaceOrderRequest(
    string Street,
    string City,
    string Province,
    string PostalCode,
    List<PlaceOrderItemRequest> Items);

public record PlaceOrderItemRequest(
    Guid ProductId,
    string ProductName,
    decimal UnitPrice,
    int Quantity);

public record CancelOrderRequest(string Reason);
```

### Program.cs - การตั้งค่าทั้งหมด

```csharp
// src/ShopThai.API/Program.cs
using System.Text;
using System.Threading.Channels;
using MassTransit;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using ShopThai.API.Hubs;
using ShopThai.API.Messaging;
using ShopThai.API.Services;
using ShopThai.API.Workers;
using ShopThai.Catalog.Application;
using ShopThai.Catalog.Infrastructure;
using ShopThai.Orders.Application;
using ShopThai.Orders.Infrastructure;
using ShopThai.Payments.Application;
using ShopThai.Payments.Infrastructure;
using ShopThai.Shared.Messaging;
using ShopThai.Users.Application;
using ShopThai.Users.Infrastructure;

var builder = WebApplication.CreateBuilder(args);

// Add API
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new() { Title = "ShopThai API", Version = "v1" });
    c.AddSecurityDefinition("Bearer", new()
    {
        Type = Microsoft.OpenApi.Models.SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT"
    });
    c.AddSecurityRequirement(new()
    {
        {
            new() { Reference = new() { Type = Microsoft.OpenApi.Models.ReferenceType.SecurityScheme, Id = "Bearer" } },
            Array.Empty<string>()
        }
    });
});

// JWT Authentication
var jwtSettings = builder.Configuration.GetSection("JwtSettings");
builder.Services.Configure<JwtSettings>(jwtSettings);
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwtSettings["Secret"]!)),
            ValidateIssuer = true,
            ValidIssuer = jwtSettings["Issuer"],
            ValidateAudience = true,
            ValidAudience = jwtSettings["Audience"],
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
        
        // SignalR JWT Support
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                var accessToken = context.Request.Query["access_token"];
                var path = context.HttpContext.Request.Path;
                
                if (!string.IsNullOrEmpty(accessToken) && 
                    path.StartsWithSegments("/hubs"))
                {
                    context.Token = accessToken;
                }
                
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();

// Redis
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
});

// SignalR
builder.Services.AddSignalR();

// Module registrations
builder.Services.AddCatalogApplication();
builder.Services.AddCatalogInfrastructure(builder.Configuration);
builder.Services.AddOrdersApplication();
builder.Services.AddOrdersInfrastructure(builder.Configuration);
builder.Services.AddPaymentsApplication();
builder.Services.AddPaymentsInfrastructure(builder.Configuration);
builder.Services.AddUsersApplication();
builder.Services.AddUsersInfrastructure(builder.Configuration);

// Email Channel + Worker
var emailChannel = Channel.CreateBounded<EmailMessage>(new BoundedChannelOptions(1000)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleWriter = false,
    SingleReader = true
});
builder.Services.AddSingleton(emailChannel);
builder.Services.AddSingleton<IEmailSender, MockEmailSender>();
builder.Services.AddHostedService<EmailWorker>();

// Dashboard Worker
builder.Services.AddScoped<IDashboardService, DashboardService>();
builder.Services.AddHostedService<DashboardWorker>();

// MassTransit
builder.Services.AddMassTransit(config =>
{
    config.AddConsumer<OrderConfirmedConsumer>();
    
    config.UsingInMemory((context, cfg) =>
    {
        cfg.ConfigureEndpoints(context);
    });
    
    // Production: ใช้ RabbitMQ หรือ Azure Service Bus
    // config.UsingRabbitMq((context, cfg) =>
    // {
    //     cfg.Host(builder.Configuration.GetConnectionString("RabbitMQ"));
    //     cfg.ConfigureEndpoints(context);
    // });
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
app.MapHub<AdminDashboardHub>("/hubs/admin-dashboard");

// Health check
app.MapGet("/health", () => Results.Ok(new { status = "healthy", timestamp = DateTime.UtcNow }));

app.Run();
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ShopThai API
  shopthai-api:
    build:
      context: .
      dockerfile: src/ShopThai.API/Dockerfile
    container_name: shopthai-api
    ports:
      - "5000:8080"
      - "5001:8081"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://+:8080;https://+:8081
      - ConnectionStrings__CatalogDb=Host=postgres;Database=shopthai_catalog;Username=postgres;Password=postgres123
      - ConnectionStrings__OrdersDb=Host=postgres;Database=shopthai_orders;Username=postgres;Password=postgres123
      - ConnectionStrings__PaymentsDb=Host=postgres;Database=shopthai_payments;Username=postgres;Password=postgres123
      - ConnectionStrings__UsersDb=Host=postgres;Database=shopthai_users;Username=postgres;Password=postgres123
      - ConnectionStrings__Redis=redis:6379
      - JwtSettings__Secret=ShopThaiSuperSecretKeyThatIsAtLeast32CharactersLong!
      - JwtSettings__Issuer=ShopThai
      - JwtSettings__Audience=ShopThaiClient
      - JwtSettings__ExpirationMinutes=60
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - shopthai-network
    restart: unless-stopped

  # PostgreSQL Database
  postgres:
    image: postgres:16-alpine
    container_name: shopthai-postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
      POSTGRES_MULTIPLE_DATABASES: shopthai_catalog,shopthai_orders,shopthai_payments,shopthai_users
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init.sh:/docker-entrypoint-initdb.d/init.sh
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - shopthai-network

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: shopthai-redis
    ports:
      - "6379:6379"
    command: redis-server --requirepass redis123 --appendonly yes
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "redis123", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - shopthai-network

  # RabbitMQ (สำหรับ Production)
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: shopthai-rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - shopthai-network

  # pgAdmin สำหรับ Database Management
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: shopthai-pgadmin
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@shopthai.com
      PGADMIN_DEFAULT_PASSWORD: admin123
    depends_on:
      - postgres
    networks:
      - shopthai-network
    profiles:
      - tools

  # Redis Commander
  redis-commander:
    image: rediscommander/redis-commander:latest
    container_name: shopthai-redis-commander
    ports:
      - "8081:8081"
    environment:
      REDIS_HOST: redis
      REDIS_PASSWORD: redis123
    depends_on:
      - redis
    networks:
      - shopthai-network
    profiles:
      - tools

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:

networks:
  shopthai-network:
    driver: bridge
```

### appsettings.json

```json
{
  "ConnectionStrings": {
    "CatalogDb": "Host=localhost;Database=shopthai_catalog;Username=postgres;Password=postgres123",
    "OrdersDb": "Host=localhost;Database=shopthai_orders;Username=postgres;Password=postgres123",
    "PaymentsDb": "Host=localhost;Database=shopthai_payments;Username=postgres;Password=postgres123",
    "UsersDb": "Host=localhost;Database=shopthai_users;Username=postgres;Password=postgres123",
    "Redis": "localhost:6379,password=redis123"
  },
  "JwtSettings": {
    "Secret": "ShopThaiSuperSecretKeyThatIsAtLeast32CharactersLong!",
    "Issuer": "ShopThai",
    "Audience": "ShopThaiClient",
    "ExpirationMinutes": 60,
    "RefreshTokenExpirationDays": 30
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

### Dockerfile

```dockerfile
# src/ShopThai.API/Dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

# Copy project files
COPY ["src/ShopThai.API/ShopThai.API.csproj", "src/ShopThai.API/"]
COPY ["src/ShopThai.Shared/ShopThai.Shared.csproj", "src/ShopThai.Shared/"]
COPY ["src/Modules/Catalog/ShopThai.Catalog.Application/ShopThai.Catalog.Application.csproj", "src/Modules/Catalog/ShopThai.Catalog.Application/"]
COPY ["src/Modules/Catalog/ShopThai.Catalog.Domain/ShopThai.Catalog.Domain.csproj", "src/Modules/Catalog/ShopThai.Catalog.Domain/"]
COPY ["src/Modules/Catalog/ShopThai.Catalog.Infrastructure/ShopThai.Catalog.Infrastructure.csproj", "src/Modules/Catalog/ShopThai.Catalog.Infrastructure/"]
COPY ["src/Modules/Orders/ShopThai.Orders.Application/ShopThai.Orders.Application.csproj", "src/Modules/Orders/ShopThai.Orders.Application/"]
COPY ["src/Modules/Orders/ShopThai.Orders.Domain/ShopThai.Orders.Domain.csproj", "src/Modules/Orders/ShopThai.Orders.Domain/"]
COPY ["src/Modules/Orders/ShopThai.Orders.Infrastructure/ShopThai.Orders.Infrastructure.csproj", "src/Modules/Orders/ShopThai.Orders.Infrastructure/"]
COPY ["src/Modules/Payments/ShopThai.Payments.Application/ShopThai.Payments.Application.csproj", "src/Modules/Payments/ShopThai.Payments.Application/"]
COPY ["src/Modules/Payments/ShopThai.Payments.Domain/ShopThai.Payments.Domain.csproj", "src/Modules/Payments/ShopThai.Payments.Domain/"]
COPY ["src/Modules/Payments/ShopThai.Payments.Infrastructure/ShopThai.Payments.Infrastructure.csproj", "src/Modules/Payments/ShopThai.Payments.Infrastructure/"]
COPY ["src/Modules/Users/ShopThai.Users.Application/ShopThai.Users.Application.csproj", "src/Modules/Users/ShopThai.Users.Application/"]
COPY ["src/Modules/Users/ShopThai.Users.Domain/ShopThai.Users.Domain.csproj", "src/Modules/Users/ShopThai.Users.Domain/"]
COPY ["src/Modules/Users/ShopThai.Users.Infrastructure/ShopThai.Users.Infrastructure.csproj", "src/Modules/Users/ShopThai.Users.Infrastructure/"]

RUN dotnet restore "src/ShopThai.API/ShopThai.API.csproj"

COPY . .

WORKDIR "/src/src/ShopThai.API"
RUN dotnet build "ShopThai.API.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
RUN dotnet publish "ShopThai.API.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "ShopThai.API.dll"]
```

### Global Packages ที่ต้องใช้

```xml
<!-- Directory.Build.props -->
<Project>
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>
</Project>
```

```xml
<!-- Directory.Packages.props -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>

  <ItemGroup>
    <!-- Shared -->
    <PackageVersion Include="MediatR" Version="12.2.0" />
    <PackageVersion Include="MassTransit" Version="8.2.3" />
    <PackageVersion Include="MassTransit.RabbitMQ" Version="8.2.3" />

    <!-- EF Core -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Tools" Version="8.0.0" />

    <!-- Authentication -->
    <PackageVersion Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="8.0.0" />
    <PackageVersion Include="System.IdentityModel.Tokens.Jwt" Version="7.2.0" />

    <!-- Redis -->
    <PackageVersion Include="Microsoft.Extensions.Caching.StackExchangeRedis" Version="8.0.0" />

    <!-- SignalR -->
    <PackageVersion Include="Microsoft.AspNetCore.SignalR" Version="1.1.0" />

    <!-- Logging -->
    <PackageVersion Include="Serilog.AspNetCore" Version="8.0.0" />
    <PackageVersion Include="Serilog.Sinks.Console" Version="5.0.1" />
    <PackageVersion Include="Serilog.Sinks.File" Version="5.0.0" />

    <!-- Validation -->
    <PackageVersion Include="FluentValidation.AspNetCore" Version="11.3.0" />

    <!-- Documentation -->
    <PackageVersion Include="Swashbuckle.AspNetCore" Version="6.5.0" />
  </ItemGroup>
</Project>
```

---

## สรุปโครงสร้างและสิ่งที่เรียนรู้

### สิ่งที่สร้างใน Part 81 นี้

ในส่วนนี้เราได้สร้างแพลตฟอร์ม **ShopThai** ที่ครบถ้วน ประกอบด้วย:

| Component | เทคโนโลยี | วัตถุประสงค์ |
|-----------|-----------|-------------|
| Modular Monolith | .NET 8 | สถาปัตยกรรมหลัก |
| Domain Modeling | DDD + Value Objects | Catalog, Orders, Payments, Users |
| Authentication | JWT + Refresh Token | ความปลอดภัย |
| Product Search | EF Core + LINQ | ค้นหาสินค้า |
| Shopping Cart | Redis | ประสิทธิภาพสูง |
| Order Management | CQRS + Domain Events | จัดการออร์เดอร์ |
| Payment Gateway | Mock + Webhook | การชำระเงิน |
| Admin Dashboard | SignalR | Real-time stats |
| Email Notifications | Channel + Background Worker | แจ้งเตือน |
| Containerization | Docker Compose | Deploy ง่าย |

### หลักการที่ใช้

1. **Domain-Driven Design (DDD)** - Entities, Value Objects, Aggregates, Domain Events
2. **CQRS** - แยก Commands และ Queries ชัดเจน
3. **Clean Architecture** - Domain → Application → Infrastructure
4. **Event-Driven** - Domain Events ด้วย MediatR และ MassTransit
5. **Modular Design** - แต่ละ Module เป็นอิสระ

### การ Run Project

```bash
# Clone และ setup
git clone <repo-url>
cd ShopThai

# Development
dotnet run --project src/ShopThai.API

# Docker
docker compose up -d

# Database migrations
dotnet ef database update --project src/Modules/Catalog/ShopThai.Catalog.Infrastructure --startup-project src/ShopThai.API
dotnet ef database update --project src/Modules/Orders/ShopThai.Orders.Infrastructure --startup-project src/ShopThai.API
dotnet ef database update --project src/Modules/Payments/ShopThai.Payments.Infrastructure --startup-project src/ShopThai.API
dotnet ef database update --project src/Modules/Users/ShopThai.Users.Infrastructure --startup-project src/ShopThai.API
```

### Endpoints หลัก

```
POST   /api/auth/register          - สมัครสมาชิก
POST   /api/auth/login             - เข้าสู่ระบบ
POST   /api/auth/refresh           - ต่ออายุ Token

GET    /api/products               - ค้นหาสินค้า
GET    /api/products/{id}          - ดูสินค้า
POST   /api/products               - สร้างสินค้า (Admin)

GET    /api/cart                   - ดูตะกร้า
POST   /api/cart/items             - เพิ่มสินค้า
PUT    /api/cart/items/{id}        - แก้ไขจำนวน
DELETE /api/cart/items/{id}        - ลบสินค้า

POST   /api/orders                 - สั่งซื้อ
GET    /api/orders                 - ดูออร์เดอร์ของฉัน
POST   /api/orders/{id}/cancel     - ยกเลิก

POST   /api/payments/process       - ชำระเงิน
POST   /api/webhooks/payment       - Webhook

WS     /hubs/admin-dashboard       - SignalR Dashboard
```

---

**ก่อนหน้า → [Part 80: Architecture Review](../part71-80/part80-architecture-review.md)**

**ต่อไป → [Part 82: Capstone - Testing & Quality](part82-capstone-testing.md)**
