# Part 82: Capstone — Testing & Quality (การทดสอบและคุณภาพ)

## Steps 811–820 — การทดสอบระบบ ShopThai E-Commerce อย่างครบวงจร

---

## ภาพรวม

ใน Part นี้เราจะนำโปรเจกต์ **ShopThai** จาก Part 81 มาเพิ่มชั้นการทดสอบแบบครบวงจร ตั้งแต่ Unit Tests ระดับ Domain ไปจนถึง Performance Tests, Architecture Tests และ CI Pipeline พร้อม Coverage Gate

โครงสร้างที่จะเพิ่มเข้าไปในโซลูชัน:

```
ShopThai.sln
├── src/
│   ├── ShopThai.Domain/
│   ├── ShopThai.Application/
│   ├── ShopThai.Infrastructure/
│   └── ShopThai.API/
└── tests/
    ├── ShopThai.Domain.Tests/          ← Step 812
    ├── ShopThai.Application.Tests/     ← Step 813
    ├── ShopThai.Integration.Tests/     ← Step 814, 815
    ├── ShopThai.Performance.Tests/     ← Step 816
    ├── ShopThai.Architecture.Tests/    ← Step 817
    └── ShopThai.Contract.Tests/        ← Step 818
```

---

## Step 811: Test Strategy สำหรับโปรเจกต์ ShopThai

### กลยุทธ์การทดสอบ (Testing Strategy)

ก่อนจะเขียน test ใดๆ เราต้องวางแผนกลยุทธ์ให้ชัดเจนก่อน โดยพิจารณาจาก:

1. **ความเสี่ยง (Risk)** — ส่วนไหนของระบบที่ถ้าพัง จะส่งผลเสียหายมากที่สุด?
2. **ต้นทุน (Cost)** — test แต่ละประเภทใช้เวลาและทรัพยากรเท่าไหร่?
3. **คุณค่า (Value)** — test นั้นให้ confidence แก่เราแค่ไหน?

สำหรับ ShopThai เรากำหนด **Test Pyramid** ดังนี้:

```
                    /\
                   /  \
                  / E2E \         ~ 5%  (Playwright, manual smoke tests)
                 /--------\
                / Contract \      ~ 5%  (Pact / WireMock for Payment)
               /------------\
              / Performance  \    ~ 5%  (NBomber load tests)
             /----------------\
            /   Integration    \  ~20%  (WebApplicationFactory + TestContainers)
           /--------------------\
          /     Application      \ ~25% (Command/Query handlers, mock repos)
         /------------------------\
        /         Domain           \ ~40% (Aggregates, Value Objects, Domain Services)
       /----------------------------\
```

### Test Coverage Targets

| Layer | Target Coverage | Tool |
|-------|----------------|------|
| Domain | ≥ 90% | coverlet + xunit |
| Application | ≥ 85% | coverlet + xunit |
| Integration | ≥ 70% | coverlet + xunit |
| Overall | ≥ 80% | ReportGenerator |

### NuGet Packages ที่จะใช้ในทุก test projects

```xml
<!-- Directory.Build.props สำหรับทุก test projects -->
<Project>
  <ItemGroup Condition="$(MSBuildProjectName.EndsWith('.Tests'))">
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.10.0" />
    <PackageReference Include="xunit" Version="2.9.0" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.2">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
    <PackageReference Include="FluentAssertions" Version="6.12.0" />
    <PackageReference Include="Moq" Version="4.20.70" />
    <PackageReference Include="coverlet.collector" Version="6.0.2">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
    <PackageReference Include="Bogus" Version="35.6.0" />
  </ItemGroup>
</Project>
```

### การตั้งค่าเริ่มต้น — สร้าง Test Projects

```bash
# สร้าง test projects ทั้งหมด
dotnet new xunit -n ShopThai.Domain.Tests -o tests/ShopThai.Domain.Tests
dotnet new xunit -n ShopThai.Application.Tests -o tests/ShopThai.Application.Tests
dotnet new xunit -n ShopThai.Integration.Tests -o tests/ShopThai.Integration.Tests
dotnet new xunit -n ShopThai.Performance.Tests -o tests/ShopThai.Performance.Tests
dotnet new xunit -n ShopThai.Architecture.Tests -o tests/ShopThai.Architecture.Tests
dotnet new xunit -n ShopThai.Contract.Tests -o tests/ShopThai.Contract.Tests

# เพิ่มเข้า solution
dotnet sln add tests/ShopThai.Domain.Tests/ShopThai.Domain.Tests.csproj
dotnet sln add tests/ShopThai.Application.Tests/ShopThai.Application.Tests.csproj
dotnet sln add tests/ShopThai.Integration.Tests/ShopThai.Integration.Tests.csproj
dotnet sln add tests/ShopThai.Performance.Tests/ShopThai.Performance.Tests.csproj
dotnet sln add tests/ShopThai.Architecture.Tests/ShopThai.Architecture.Tests.csproj
dotnet sln add tests/ShopThai.Contract.Tests/ShopThai.Contract.Tests.csproj
```

---

## Step 812: Domain Unit Tests (Order Aggregate และ Value Objects)

### แนวคิด

Domain tests คือ tests ที่ทำงานเร็วที่สุด เพราะไม่มีการพึ่งพา infrastructure ใดๆ เลย ทดสอบเฉพาะ business logic ล้วนๆ

### ทบทวน Domain Model ของ ShopThai

```csharp
// ShopThai.Domain/Orders/Order.cs — ทบทวนจาก Part 81
public class Order : AggregateRoot
{
    private readonly List<OrderItem> _items = new();

    public CustomerId CustomerId { get; private set; } = null!;
    public OrderStatus Status { get; private set; }
    public Money TotalAmount { get; private set; } = Money.Zero;
    public Address ShippingAddress { get; private set; } = null!;
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    public DateTime CreatedAt { get; private set; }
    public DateTime? ConfirmedAt { get; private set; }
    public DateTime? CancelledAt { get; private set; }
    public string? CancellationReason { get; private set; }

    private Order() { }

    public static Order Create(CustomerId customerId, Address shippingAddress)
    {
        ArgumentNullException.ThrowIfNull(customerId);
        ArgumentNullException.ThrowIfNull(shippingAddress);

        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            ShippingAddress = shippingAddress,
            Status = OrderStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };

        order.AddDomainEvent(new OrderCreatedEvent(order.Id, customerId));
        return order;
    }

    public void AddItem(ProductId productId, string productName, Money unitPrice, int quantity)
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException($"Cannot add items to order in {Status} status.");

        if (quantity <= 0)
            throw new ArgumentOutOfRangeException(nameof(quantity), "Quantity must be positive.");

        var existingItem = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existingItem is not null)
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
            throw new InvalidOperationException("Only pending orders can be confirmed.");

        if (!_items.Any())
            throw new InvalidOperationException("Cannot confirm an empty order.");

        Status = OrderStatus.Confirmed;
        ConfirmedAt = DateTime.UtcNow;
        AddDomainEvent(new OrderConfirmedEvent(Id, CustomerId, TotalAmount));
    }

    public void Cancel(string reason)
    {
        if (Status == OrderStatus.Shipped || Status == OrderStatus.Delivered)
            throw new InvalidOperationException($"Cannot cancel order that is already {Status}.");

        Status = OrderStatus.Cancelled;
        CancelledAt = DateTime.UtcNow;
        CancellationReason = reason;
        AddDomainEvent(new OrderCancelledEvent(Id, reason));
    }

    private void RecalculateTotal()
    {
        TotalAmount = _items.Aggregate(Money.Zero, (sum, item) => sum + item.Total);
    }
}
```

### Value Object Tests — Money

```csharp
// tests/ShopThai.Domain.Tests/ValueObjects/MoneyTests.cs
using ShopThai.Domain.Common;
using FluentAssertions;

namespace ShopThai.Domain.Tests.ValueObjects;

public class MoneyTests
{
    [Fact]
    public void Create_WithValidAmount_ShouldSucceed()
    {
        // Arrange & Act
        var money = new Money(100.50m, "THB");

        // Assert
        money.Amount.Should().Be(100.50m);
        money.Currency.Should().Be("THB");
    }

    [Theory]
    [InlineData(-1)]
    [InlineData(-100.5)]
    public void Create_WithNegativeAmount_ShouldThrow(decimal amount)
    {
        // Act
        var act = () => new Money(amount, "THB");

        // Assert
        act.Should().Throw<ArgumentOutOfRangeException>()
           .WithMessage("*amount*");
    }

    [Fact]
    public void Create_WithEmptyCurrency_ShouldThrow()
    {
        var act = () => new Money(100m, "");
        act.Should().Throw<ArgumentException>();
    }

    [Fact]
    public void Add_SameCurrency_ShouldSumAmounts()
    {
        // Arrange
        var money1 = new Money(100m, "THB");
        var money2 = new Money(250m, "THB");

        // Act
        var result = money1 + money2;

        // Assert
        result.Amount.Should().Be(350m);
        result.Currency.Should().Be("THB");
    }

    [Fact]
    public void Add_DifferentCurrencies_ShouldThrow()
    {
        var thb = new Money(100m, "THB");
        var usd = new Money(3m, "USD");

        var act = () => thb + usd;
        act.Should().Throw<InvalidOperationException>()
           .WithMessage("*currency*");
    }

    [Fact]
    public void Multiply_ByPositiveQuantity_ShouldScaleAmount()
    {
        var unitPrice = new Money(99.99m, "THB");

        var total = unitPrice * 3;

        total.Amount.Should().Be(299.97m);
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    public void Multiply_ByNonPositiveQuantity_ShouldThrow(int quantity)
    {
        var price = new Money(100m, "THB");
        var act = () => price * quantity;
        act.Should().Throw<ArgumentOutOfRangeException>();
    }

    [Fact]
    public void Equality_SameAmountAndCurrency_ShouldBeEqual()
    {
        var money1 = new Money(100m, "THB");
        var money2 = new Money(100m, "THB");

        money1.Should().Be(money2);
        (money1 == money2).Should().BeTrue();
    }

    [Fact]
    public void Equality_DifferentAmount_ShouldNotBeEqual()
    {
        var money1 = new Money(100m, "THB");
        var money2 = new Money(200m, "THB");

        money1.Should().NotBe(money2);
    }

    [Fact]
    public void Zero_ShouldHaveZeroAmount()
    {
        Money.Zero.Amount.Should().Be(0m);
    }

    [Fact]
    public void ToString_ShouldFormatCorrectly()
    {
        var money = new Money(1234.56m, "THB");
        money.ToString().Should().Be("1234.56 THB");
    }
}
```

### Order Aggregate Tests

```csharp
// tests/ShopThai.Domain.Tests/Orders/OrderTests.cs
using ShopThai.Domain.Orders;
using ShopThai.Domain.Common;
using ShopThai.Domain.Tests.Builders;
using FluentAssertions;

namespace ShopThai.Domain.Tests.Orders;

public class OrderTests
{
    private readonly CustomerId _customerId = CustomerId.Create(Guid.NewGuid());
    private readonly Address _address = new("123 ถนนสุขุมวิท", "กรุงเทพมหานคร", "10110", "TH");
    private readonly ProductId _productId = ProductId.Create(Guid.NewGuid());

    [Fact]
    public void Create_WithValidData_ShouldCreatePendingOrder()
    {
        // Act
        var order = Order.Create(_customerId, _address);

        // Assert
        order.Should().NotBeNull();
        order.Id.Should().NotBe(Guid.Empty);
        order.CustomerId.Should().Be(_customerId);
        order.Status.Should().Be(OrderStatus.Pending);
        order.Items.Should().BeEmpty();
        order.TotalAmount.Should().Be(Money.Zero);
        order.CreatedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(5));
    }

    [Fact]
    public void Create_ShouldRaiseOrderCreatedEvent()
    {
        var order = Order.Create(_customerId, _address);

        order.DomainEvents.Should().ContainSingle(e => e is OrderCreatedEvent);
        var evt = order.DomainEvents.OfType<OrderCreatedEvent>().Single();
        evt.CustomerId.Should().Be(_customerId);
    }

    [Fact]
    public void AddItem_ToPendingOrder_ShouldAddItemAndUpdateTotal()
    {
        // Arrange
        var order = Order.Create(_customerId, _address);
        var unitPrice = new Money(299.99m, "THB");

        // Act
        order.AddItem(_productId, "เสื้อยืด", unitPrice, 2);

        // Assert
        order.Items.Should().HaveCount(1);
        order.TotalAmount.Amount.Should().Be(599.98m);
    }

    [Fact]
    public void AddItem_SameProductTwice_ShouldMergeQuantities()
    {
        // Arrange
        var order = Order.Create(_customerId, _address);
        var unitPrice = new Money(100m, "THB");

        // Act
        order.AddItem(_productId, "กระเป๋า", unitPrice, 2);
        order.AddItem(_productId, "กระเป๋า", unitPrice, 3);

        // Assert
        order.Items.Should().HaveCount(1);
        order.Items[0].Quantity.Should().Be(5);
        order.TotalAmount.Amount.Should().Be(500m);
    }

    [Fact]
    public void AddItem_ToConfirmedOrder_ShouldThrow()
    {
        // Arrange
        var order = new OrderBuilder()
            .WithCustomer(_customerId)
            .WithAddress(_address)
            .WithItem(_productId, "สินค้า", new Money(100m, "THB"), 1)
            .InConfirmedState()
            .Build();

        // Act
        var act = () => order.AddItem(ProductId.Create(Guid.NewGuid()), "สินค้าใหม่", new Money(50m, "THB"), 1);

        // Assert
        act.Should().Throw<InvalidOperationException>()
           .WithMessage("*Confirmed*");
    }

    [Fact]
    public void AddItem_WithZeroQuantity_ShouldThrow()
    {
        var order = Order.Create(_customerId, _address);
        var act = () => order.AddItem(_productId, "สินค้า", new Money(100m, "THB"), 0);
        act.Should().Throw<ArgumentOutOfRangeException>();
    }

    [Fact]
    public void Confirm_WithItems_ShouldChangeStatusToConfirmed()
    {
        // Arrange
        var order = Order.Create(_customerId, _address);
        order.AddItem(_productId, "สินค้า", new Money(100m, "THB"), 1);

        // Act
        order.Confirm();

        // Assert
        order.Status.Should().Be(OrderStatus.Confirmed);
        order.ConfirmedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(5));
    }

    [Fact]
    public void Confirm_ShouldRaiseOrderConfirmedEvent()
    {
        var order = Order.Create(_customerId, _address);
        order.AddItem(_productId, "สินค้า", new Money(150m, "THB"), 1);
        order.ClearDomainEvents(); // ล้าง OrderCreatedEvent ออกก่อน

        order.Confirm();

        var evt = order.DomainEvents.OfType<OrderConfirmedEvent>().Should().ContainSingle().Subject;
        evt.TotalAmount.Amount.Should().Be(150m);
    }

    [Fact]
    public void Confirm_EmptyOrder_ShouldThrow()
    {
        var order = Order.Create(_customerId, _address);
        var act = () => order.Confirm();
        act.Should().Throw<InvalidOperationException>()
           .WithMessage("*empty*");
    }

    [Fact]
    public void Confirm_AlreadyConfirmedOrder_ShouldThrow()
    {
        var order = new OrderBuilder()
            .WithCustomer(_customerId)
            .WithAddress(_address)
            .WithItem(_productId, "สินค้า", new Money(100m, "THB"), 1)
            .InConfirmedState()
            .Build();

        var act = () => order.Confirm();
        act.Should().Throw<InvalidOperationException>()
           .WithMessage("*pending*");
    }

    [Theory]
    [InlineData(OrderStatus.Pending)]
    [InlineData(OrderStatus.Confirmed)]
    public void Cancel_FromAllowedStatus_ShouldCancelOrder(OrderStatus initialStatus)
    {
        // Arrange
        var builder = new OrderBuilder()
            .WithCustomer(_customerId)
            .WithAddress(_address)
            .WithItem(_productId, "สินค้า", new Money(100m, "THB"), 1);

        if (initialStatus == OrderStatus.Confirmed)
            builder.InConfirmedState();

        var order = builder.Build();

        // Act
        order.Cancel("ลูกค้าเปลี่ยนใจ");

        // Assert
        order.Status.Should().Be(OrderStatus.Cancelled);
        order.CancellationReason.Should().Be("ลูกค้าเปลี่ยนใจ");
        order.CancelledAt.Should().NotBeNull();
    }

    [Theory]
    [InlineData(OrderStatus.Shipped)]
    [InlineData(OrderStatus.Delivered)]
    public void Cancel_FromDisallowedStatus_ShouldThrow(OrderStatus status)
    {
        var order = new OrderBuilder()
            .WithCustomer(_customerId)
            .WithAddress(_address)
            .WithItem(_productId, "สินค้า", new Money(100m, "THB"), 1)
            .InStatus(status)
            .Build();

        var act = () => order.Cancel("เหตุผล");
        act.Should().Throw<InvalidOperationException>()
           .WithMessage($"*{status}*");
    }

    [Fact]
    public void Cancel_ShouldRaiseOrderCancelledEvent()
    {
        var order = Order.Create(_customerId, _address);
        order.AddItem(_productId, "สินค้า", new Money(100m, "THB"), 1);
        order.ClearDomainEvents();

        order.Cancel("ทดสอบ");

        order.DomainEvents.Should().ContainSingle(e => e is OrderCancelledEvent);
    }
}
```

---

## Step 813: Application Layer Tests (Command/Query Handlers)

### แนวคิด

Application layer tests ทดสอบ business use cases โดยใช้ Mock repositories แทน database จริง เพื่อให้ tests ทำงานเร็วและ deterministic

```csharp
// tests/ShopThai.Application.Tests/ShopThai.Application.Tests.csproj
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\..\src\ShopThai.Application\ShopThai.Application.csproj" />
    <PackageReference Include="Moq" Version="4.20.70" />
    <PackageReference Include="FluentAssertions" Version="6.12.0" />
  </ItemGroup>
</Project>
```

### Place Order Command Handler Tests

```csharp
// tests/ShopThai.Application.Tests/Orders/PlaceOrderCommandHandlerTests.cs
using Moq;
using FluentAssertions;
using ShopThai.Application.Orders.Commands;
using ShopThai.Application.Common.Interfaces;
using ShopThai.Domain.Orders;
using ShopThai.Domain.Products;
using ShopThai.Domain.Common;

namespace ShopThai.Application.Tests.Orders;

public class PlaceOrderCommandHandlerTests
{
    private readonly Mock<IOrderRepository> _orderRepoMock;
    private readonly Mock<IProductRepository> _productRepoMock;
    private readonly Mock<IUnitOfWork> _unitOfWorkMock;
    private readonly Mock<IEventPublisher> _eventPublisherMock;
    private readonly PlaceOrderCommandHandler _handler;

    public PlaceOrderCommandHandlerTests()
    {
        _orderRepoMock = new Mock<IOrderRepository>();
        _productRepoMock = new Mock<IProductRepository>();
        _unitOfWorkMock = new Mock<IUnitOfWork>();
        _eventPublisherMock = new Mock<IEventPublisher>();

        _handler = new PlaceOrderCommandHandler(
            _orderRepoMock.Object,
            _productRepoMock.Object,
            _unitOfWorkMock.Object,
            _eventPublisherMock.Object);
    }

    [Fact]
    public async Task Handle_ValidCommand_ShouldCreateAndReturnOrderId()
    {
        // Arrange
        var customerId = Guid.NewGuid();
        var productId = Guid.NewGuid();
        var product = Product.Create("เสื้อยืด", new Money(199m, "THB"), 100);

        _productRepoMock
            .Setup(r => r.GetByIdAsync(It.Is<ProductId>(id => id.Value == productId), It.IsAny<CancellationToken>()))
            .ReturnsAsync(product);

        _orderRepoMock
            .Setup(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
            .Returns(Task.CompletedTask);

        _unitOfWorkMock
            .Setup(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()))
            .ReturnsAsync(1);

        var command = new PlaceOrderCommand(
            CustomerId: customerId,
            Items: new List<OrderItemDto>
            {
                new(productId, Quantity: 2)
            },
            ShippingAddress: new AddressDto("456 ถนนพระราม9", "กรุงเทพฯ", "10310", "TH")
        );

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBe(Guid.Empty);
    }

    [Fact]
    public async Task Handle_ProductNotFound_ShouldReturnFailure()
    {
        // Arrange
        var productId = Guid.NewGuid();

        _productRepoMock
            .Setup(r => r.GetByIdAsync(It.IsAny<ProductId>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync((Product?)null);

        var command = new PlaceOrderCommand(
            CustomerId: Guid.NewGuid(),
            Items: new List<OrderItemDto> { new(productId, 1) },
            ShippingAddress: new AddressDto("ที่อยู่", "กรุงเทพฯ", "10000", "TH")
        );

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain(productId.ToString());
    }

    [Fact]
    public async Task Handle_InsufficientStock_ShouldReturnFailure()
    {
        // Arrange
        var productId = Guid.NewGuid();
        var product = Product.Create("สินค้า", new Money(100m, "THB"), stock: 1); // มีแค่ 1 ชิ้น

        _productRepoMock
            .Setup(r => r.GetByIdAsync(It.IsAny<ProductId>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(product);

        var command = new PlaceOrderCommand(
            CustomerId: Guid.NewGuid(),
            Items: new List<OrderItemDto> { new(productId, Quantity: 5) }, // สั่ง 5 ชิ้น
            ShippingAddress: new AddressDto("ที่อยู่", "กรุงเทพฯ", "10000", "TH")
        );

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("stock");
    }

    [Fact]
    public async Task Handle_ValidCommand_ShouldCallAddAsync_Once()
    {
        // Arrange
        var productId = Guid.NewGuid();
        var product = Product.Create("สินค้า", new Money(100m, "THB"), 10);

        _productRepoMock
            .Setup(r => r.GetByIdAsync(It.IsAny<ProductId>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(product);

        _unitOfWorkMock
            .Setup(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()))
            .ReturnsAsync(1);

        var command = new PlaceOrderCommand(
            CustomerId: Guid.NewGuid(),
            Items: new List<OrderItemDto> { new(productId, 1) },
            ShippingAddress: new AddressDto("ที่อยู่", "กรุงเทพฯ", "10000", "TH")
        );

        // Act
        await _handler.Handle(command, CancellationToken.None);

        // Assert — verify ว่าเรียก AddAsync ครั้งเดียว
        _orderRepoMock.Verify(
            r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()),
            Times.Once);
    }

    [Fact]
    public async Task Handle_ValidCommand_ShouldSaveChanges()
    {
        // Arrange
        var product = Product.Create("สินค้า", new Money(100m, "THB"), 10);

        _productRepoMock
            .Setup(r => r.GetByIdAsync(It.IsAny<ProductId>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(product);

        var command = new PlaceOrderCommand(
            CustomerId: Guid.NewGuid(),
            Items: new List<OrderItemDto> { new(Guid.NewGuid(), 1) },
            ShippingAddress: new AddressDto("ที่อยู่", "กรุงเทพฯ", "10000", "TH")
        );

        // Act
        await _handler.Handle(command, CancellationToken.None);

        // Assert
        _unitOfWorkMock.Verify(
            u => u.SaveChangesAsync(It.IsAny<CancellationToken>()),
            Times.Once);
    }
}
```

### Get Products Query Handler Tests

```csharp
// tests/ShopThai.Application.Tests/Products/GetProductsQueryHandlerTests.cs
using Moq;
using FluentAssertions;
using ShopThai.Application.Products.Queries;
using ShopThai.Application.Common.Interfaces;
using ShopThai.Domain.Products;
using ShopThai.Domain.Common;

namespace ShopThai.Application.Tests.Products;

public class GetProductsQueryHandlerTests
{
    private readonly Mock<IProductReadRepository> _readRepoMock;
    private readonly GetProductsQueryHandler _handler;

    public GetProductsQueryHandlerTests()
    {
        _readRepoMock = new Mock<IProductReadRepository>();
        _handler = new GetProductsQueryHandler(_readRepoMock.Object);
    }

    [Fact]
    public async Task Handle_WithNoFilter_ShouldReturnAllProducts()
    {
        // Arrange
        var products = Enumerable.Range(1, 5)
            .Select(i => new ProductDto(Guid.NewGuid(), $"สินค้า {i}", 100m * i, i * 10))
            .ToList();

        _readRepoMock
            .Setup(r => r.SearchAsync(
                It.IsAny<string?>(),
                It.IsAny<decimal?>(),
                It.IsAny<decimal?>(),
                It.IsAny<int>(),
                It.IsAny<int>(),
                It.IsAny<CancellationToken>()))
            .ReturnsAsync(new PagedResult<ProductDto>(products, 5, 1, 10));

        var query = new GetProductsQuery(SearchTerm: null, MinPrice: null, MaxPrice: null, Page: 1, PageSize: 10);

        // Act
        var result = await _handler.Handle(query, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.TotalCount.Should().Be(5);
        result.Value.Items.Should().HaveCount(5);
    }

    [Fact]
    public async Task Handle_WithSearchTerm_ShouldPassSearchTermToRepository()
    {
        // Arrange
        _readRepoMock
            .Setup(r => r.SearchAsync(
                "เสื้อ",
                It.IsAny<decimal?>(),
                It.IsAny<decimal?>(),
                It.IsAny<int>(),
                It.IsAny<int>(),
                It.IsAny<CancellationToken>()))
            .ReturnsAsync(new PagedResult<ProductDto>(new List<ProductDto>(), 0, 1, 10));

        var query = new GetProductsQuery(SearchTerm: "เสื้อ", Page: 1, PageSize: 10);

        // Act
        await _handler.Handle(query, CancellationToken.None);

        // Assert — verify ว่า searchTerm ถูกส่งไปที่ repository
        _readRepoMock.Verify(
            r => r.SearchAsync("เสื้อ", null, null, 1, 10, It.IsAny<CancellationToken>()),
            Times.Once);
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    public async Task Handle_WithInvalidPage_ShouldReturnFailure(int page)
    {
        var query = new GetProductsQuery(Page: page, PageSize: 10);
        var result = await _handler.Handle(query, CancellationToken.None);
        result.IsFailure.Should().BeTrue();
    }
}
```

---

## Step 814: Integration Tests ด้วย WebApplicationFactory และ TestContainers

### ติดตั้ง Packages

```bash
cd tests/ShopThai.Integration.Tests

dotnet add package Microsoft.AspNetCore.Mvc.Testing
dotnet add package Testcontainers.PostgreSql --version 3.9.0
dotnet add package Respawn --version 6.2.1
dotnet add package Bogus
```

### Custom WebApplicationFactory

```csharp
// tests/ShopThai.Integration.Tests/Infrastructure/ShopThaiWebFactory.cs
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.AspNetCore.TestHost;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;
using Testcontainers.PostgreSql;
using Respawn;
using ShopThai.Infrastructure.Persistence;

namespace ShopThai.Integration.Tests.Infrastructure;

public class ShopThaiWebFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _dbContainer = new PostgreSqlBuilder()
        .WithDatabase("shopthai_test")
        .WithUsername("testuser")
        .WithPassword("testpassword")
        .WithImage("postgres:16-alpine")
        .Build();

    private Respawner _respawner = null!;
    private string _connectionString = "";

    public async Task InitializeAsync()
    {
        // เริ่ม PostgreSQL container
        await _dbContainer.StartAsync();
        _connectionString = _dbContainer.GetConnectionString();

        // รัน migrations
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<ShopThaiDbContext>();
        await db.Database.MigrateAsync();

        // ตั้งค่า Respawner สำหรับ reset database ระหว่าง tests
        await using var conn = new Npgsql.NpgsqlConnection(_connectionString);
        await conn.OpenAsync();

        _respawner = await Respawner.CreateAsync(conn, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            SchemasToInclude = new[] { "public" },
            TablesToIgnore = new[] { "__EFMigrationsHistory" }
        });
    }

    public async Task ResetDatabaseAsync()
    {
        await using var conn = new Npgsql.NpgsqlConnection(_connectionString);
        await conn.OpenAsync();
        await _respawner.ResetAsync(conn);
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // แทนที่ DbContext ด้วย test database
            services.RemoveAll<DbContextOptions<ShopThaiDbContext>>();
            services.RemoveAll<ShopThaiDbContext>();

            services.AddDbContext<ShopThaiDbContext>(options =>
            {
                options.UseNpgsql(_connectionString);
                options.EnableSensitiveDataLogging();
            });

            // แทนที่ JWT authentication ด้วย test auth
            services.RemoveAll<Microsoft.AspNetCore.Authentication.IAuthenticationService>();
            services.AddAuthentication(TestAuthHandler.SchemeName)
                    .AddScheme<TestAuthHandlerOptions, TestAuthHandler>(
                        TestAuthHandler.SchemeName, _ => { });
        });

        builder.UseEnvironment("Testing");
    }

    public new async Task DisposeAsync()
    {
        await _dbContainer.DisposeAsync();
    }
}
```

### Test Authentication Handler

```csharp
// tests/ShopThai.Integration.Tests/Infrastructure/TestAuthHandler.cs
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace ShopThai.Integration.Tests.Infrastructure;

public class TestAuthHandlerOptions : AuthenticationSchemeOptions { }

public class TestAuthHandler : AuthenticationHandler<TestAuthHandlerOptions>
{
    public const string SchemeName = "TestScheme";

    // Headers ที่ client จะส่งมาเพื่อระบุตัวตน
    public const string UserIdHeader = "X-Test-UserId";
    public const string UserRoleHeader = "X-Test-UserRole";

    public TestAuthHandler(
        IOptionsMonitor<TestAuthHandlerOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder) : base(options, logger, encoder) { }

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue(UserIdHeader, out var userId))
            return Task.FromResult(AuthenticateResult.Fail("Missing test user header"));

        var role = Request.Headers.TryGetValue(UserRoleHeader, out var r) ? r.ToString() : "Customer";

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, userId.ToString()),
            new Claim(ClaimTypes.Email, $"test_{userId}@shopthai.test"),
            new Claim(ClaimTypes.Role, role),
        };

        var identity = new ClaimsIdentity(claims, SchemeName);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, SchemeName);

        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

### Base Integration Test Class

```csharp
// tests/ShopThai.Integration.Tests/Infrastructure/IntegrationTestBase.cs
using Microsoft.Extensions.DependencyInjection;
using ShopThai.Infrastructure.Persistence;

namespace ShopThai.Integration.Tests.Infrastructure;

[Collection("Integration")]
public abstract class IntegrationTestBase : IAsyncLifetime
{
    protected readonly ShopThaiWebFactory Factory;
    protected readonly HttpClient Client;

    protected IntegrationTestBase(ShopThaiWebFactory factory)
    {
        Factory = factory;
        Client = factory.CreateClient();
    }

    public Task InitializeAsync() => Factory.ResetDatabaseAsync();

    public Task DisposeAsync() => Task.CompletedTask;

    protected void AuthenticateAs(Guid userId, string role = "Customer")
    {
        Client.DefaultRequestHeaders.Remove(TestAuthHandler.UserIdHeader);
        Client.DefaultRequestHeaders.Remove(TestAuthHandler.UserRoleHeader);
        Client.DefaultRequestHeaders.Add(TestAuthHandler.UserIdHeader, userId.ToString());
        Client.DefaultRequestHeaders.Add(TestAuthHandler.UserRoleHeader, role);
    }

    protected async Task<T> GetServiceAsync<T>(Func<T, Task> action) where T : notnull
    {
        using var scope = Factory.Services.CreateScope();
        var service = scope.ServiceProvider.GetRequiredService<T>();
        await action(service);
        return service;
    }

    protected async Task SeedDataAsync(Func<ShopThaiDbContext, Task> seedAction)
    {
        using var scope = Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<ShopThaiDbContext>();
        await seedAction(db);
        await db.SaveChangesAsync();
    }
}

// Collection definition
[CollectionDefinition("Integration")]
public class IntegrationTestCollection : ICollectionFixture<ShopThaiWebFactory> { }
```

---

## Step 815: API Endpoint Tests พร้อม Authenticated Requests

### Product API Tests

```csharp
// tests/ShopThai.Integration.Tests/API/ProductsApiTests.cs
using System.Net;
using System.Net.Http.Json;
using FluentAssertions;
using ShopThai.Integration.Tests.Infrastructure;
using ShopThai.Integration.Tests.Factories;

namespace ShopThai.Integration.Tests.API;

public class ProductsApiTests : IntegrationTestBase
{
    public ProductsApiTests(ShopThaiWebFactory factory) : base(factory) { }

    [Fact]
    public async Task GET_Products_WithoutFilter_ShouldReturn200()
    {
        // Arrange — seed ข้อมูลทดสอบ
        await SeedDataAsync(async db =>
        {
            db.Products.AddRange(ProductFakeFactory.Generate(5));
            await db.SaveChangesAsync();
        });

        // Act
        var response = await Client.GetAsync("/api/v1/products");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var body = await response.Content.ReadFromJsonAsync<PagedResponse<ProductDto>>();
        body.Should().NotBeNull();
        body!.TotalCount.Should().Be(5);
    }

    [Fact]
    public async Task GET_Products_WithSearchTerm_ShouldFilterResults()
    {
        // Arrange
        await SeedDataAsync(async db =>
        {
            db.Products.Add(ProductFakeFactory.WithName("เสื้อยืดสีขาว"));
            db.Products.Add(ProductFakeFactory.WithName("กางเกงยีนส์"));
            db.Products.Add(ProductFakeFactory.WithName("เสื้อเชิ้ตลายสก็อต"));
            await db.SaveChangesAsync();
        });

        // Act
        var response = await Client.GetAsync("/api/v1/products?search=เสื้อ");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var body = await response.Content.ReadFromJsonAsync<PagedResponse<ProductDto>>();
        body!.TotalCount.Should().Be(2);
        body.Items.Should().AllSatisfy(p => p.Name.Should().Contain("เสื้อ"));
    }

    [Fact]
    public async Task GET_Product_ById_WhenExists_ShouldReturn200()
    {
        // Arrange
        var product = ProductFakeFactory.Generate();
        await SeedDataAsync(async db =>
        {
            db.Products.Add(product);
            await db.SaveChangesAsync();
        });

        // Act
        var response = await Client.GetAsync($"/api/v1/products/{product.Id}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var dto = await response.Content.ReadFromJsonAsync<ProductDto>();
        dto!.Id.Should().Be(product.Id);
    }

    [Fact]
    public async Task GET_Product_ById_WhenNotExists_ShouldReturn404()
    {
        var response = await Client.GetAsync($"/api/v1/products/{Guid.NewGuid()}");
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task POST_Product_AsAdmin_ShouldReturn201()
    {
        // Arrange — authenticate เป็น Admin
        AuthenticateAs(Guid.NewGuid(), role: "Admin");

        var request = new CreateProductRequest(
            Name: "สินค้าทดสอบ",
            Description: "รายละเอียด",
            Price: 199.99m,
            Stock: 50
        );

        // Act
        var response = await Client.PostAsJsonAsync("/api/v1/products", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
    }

    [Fact]
    public async Task POST_Product_AsCustomer_ShouldReturn403()
    {
        // Arrange — authenticate เป็น Customer ธรรมดา
        AuthenticateAs(Guid.NewGuid(), role: "Customer");

        var request = new CreateProductRequest("สินค้า", "รายละเอียด", 100m, 10);

        // Act
        var response = await Client.PostAsJsonAsync("/api/v1/products", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }
}
```

### Order API Tests

```csharp
// tests/ShopThai.Integration.Tests/API/OrdersApiTests.cs
using System.Net;
using System.Net.Http.Json;
using FluentAssertions;
using ShopThai.Integration.Tests.Infrastructure;
using ShopThai.Integration.Tests.Factories;

namespace ShopThai.Integration.Tests.API;

public class OrdersApiTests : IntegrationTestBase
{
    public OrdersApiTests(ShopThaiWebFactory factory) : base(factory) { }

    [Fact]
    public async Task POST_Orders_WithValidRequest_ShouldReturn201()
    {
        // Arrange
        var customerId = Guid.NewGuid();
        AuthenticateAs(customerId);

        var product = ProductFakeFactory.Generate();
        await SeedDataAsync(async db =>
        {
            db.Products.Add(product);
            await db.SaveChangesAsync();
        });

        var request = new PlaceOrderRequest(
            Items: new List<OrderItemRequest> { new(product.Id, Quantity: 2) },
            ShippingAddress: new AddressRequest("123 ถนน", "กรุงเทพฯ", "10000", "TH")
        );

        // Act
        var response = await Client.PostAsJsonAsync("/api/v1/orders", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var dto = await response.Content.ReadFromJsonAsync<OrderCreatedResponse>();
        dto!.OrderId.Should().NotBe(Guid.Empty);
    }

    [Fact]
    public async Task POST_Orders_WithoutAuth_ShouldReturn401()
    {
        // ไม่ได้ call AuthenticateAs → ไม่มี header
        var request = new PlaceOrderRequest(
            Items: new List<OrderItemRequest> { new(Guid.NewGuid(), 1) },
            ShippingAddress: new AddressRequest("ที่อยู่", "กรุงเทพฯ", "10000", "TH")
        );

        var response = await Client.PostAsJsonAsync("/api/v1/orders", request);
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task GET_Orders_ShouldReturnOnlyCurrentUserOrders()
    {
        // Arrange
        var user1 = Guid.NewGuid();
        var user2 = Guid.NewGuid();

        await SeedDataAsync(async db =>
        {
            db.Orders.AddRange(OrderFakeFactory.GenerateForCustomer(user1, count: 3));
            db.Orders.AddRange(OrderFakeFactory.GenerateForCustomer(user2, count: 2));
            await db.SaveChangesAsync();
        });

        AuthenticateAs(user1);

        // Act
        var response = await Client.GetAsync("/api/v1/orders");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var orders = await response.Content.ReadFromJsonAsync<List<OrderSummaryDto>>();
        orders.Should().HaveCount(3);
        orders!.All(o => o.CustomerId == user1).Should().BeTrue();
    }

    [Fact]
    public async Task DELETE_Order_Cancel_ShouldReturn200()
    {
        // Arrange
        var customerId = Guid.NewGuid();
        var order = OrderFakeFactory.PendingOrderForCustomer(customerId);

        await SeedDataAsync(async db =>
        {
            db.Orders.Add(order);
            await db.SaveChangesAsync();
        });

        AuthenticateAs(customerId);

        // Act
        var response = await Client.DeleteAsync($"/api/v1/orders/{order.Id}?reason=ทดสอบยกเลิก");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }
}
```

---

## Step 816: Performance Tests ด้วย NBomber

### ติดตั้ง NBomber

```bash
cd tests/ShopThai.Performance.Tests
dotnet add package NBomber --version 5.4.0
dotnet add package NBomber.Http --version 5.4.0
```

### Load Test สำหรับ Product Search Endpoint

```csharp
// tests/ShopThai.Performance.Tests/ProductSearchLoadTests.cs
using NBomber.CSharp;
using NBomber.Http.CSharp;
using FluentAssertions;

namespace ShopThai.Performance.Tests;

public class ProductSearchLoadTests
{
    private const string BaseUrl = "http://localhost:5000";

    [Fact(Skip = "Run manually only — requires running API")]
    public void ProductSearch_Under100ConcurrentUsers_ShouldMeetSLA()
    {
        // กำหนด scenario
        var httpClient = new HttpClient { BaseAddress = new Uri(BaseUrl) };

        var scenario = Scenario.Create("product_search", async context =>
            {
                // Random search terms ที่เหมือนจริง
                var searchTerms = new[] { "เสื้อ", "กางเกง", "รองเท้า", "กระเป๋า", "หมวก", "" };
                var term = searchTerms[context.InvocationNumber % searchTerms.Length];

                var response = await httpClient.GetAsync(
                    $"/api/v1/products?search={Uri.EscapeDataString(term)}&page=1&pageSize=20");

                return response.IsSuccessStatusCode
                    ? Response.Ok(statusCode: (int)response.StatusCode)
                    : Response.Fail(statusCode: (int)response.StatusCode);
            })
            .WithLoadSimulations(
                Simulation.RampingInject(rate: 10, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(10)),
                Simulation.Inject(rate: 100, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(30)),
                Simulation.RampingInject(rate: 0, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(10))
            );

        var stats = NBomberRunner
            .RegisterScenarios(scenario)
            .WithReportFolder("performance-reports")
            .WithReportFormats(ReportFormat.Html, ReportFormat.Csv)
            .Run();

        // Assert SLA requirements
        var productSearchStats = stats.ScenarioStats[0];

        // P99 latency ต้องน้อยกว่า 500ms
        productSearchStats.Ok.Latency.Percent99.Should().BeLessThan(500,
            "P99 latency must be under 500ms");

        // Error rate ต้องน้อยกว่า 1%
        productSearchStats.Fail.Request.Percent.Should().BeLessThan(1.0,
            "Error rate must be under 1%");

        // Throughput ต้องได้อย่างน้อย 50 RPS
        productSearchStats.Ok.Request.RPS.Should().BeGreaterThan(50,
            "Throughput must be at least 50 RPS");
    }

    [Fact(Skip = "Run manually only")]
    public void OrderPlacement_Under50ConcurrentUsers_ShouldMeetSLA()
    {
        var httpClient = new HttpClient { BaseAddress = new Uri(BaseUrl) };

        var scenario = Scenario.Create("place_order", async context =>
            {
                // เพิ่ม auth header จำลอง
                var request = new HttpRequestMessage(HttpMethod.Post, "/api/v1/orders")
                {
                    Headers = { { "X-Test-UserId", Guid.NewGuid().ToString() } },
                    Content = JsonContent.Create(new
                    {
                        items = new[] { new { productId = Guid.NewGuid(), quantity = 1 } },
                        shippingAddress = new
                        {
                            street = "123 ถนน",
                            city = "กรุงเทพฯ",
                            postalCode = "10000",
                            country = "TH"
                        }
                    })
                };

                var response = await httpClient.SendAsync(request);

                return response.IsSuccessStatusCode
                    ? Response.Ok()
                    : Response.Fail(message: await response.Content.ReadAsStringAsync());
            })
            .WithLoadSimulations(
                Simulation.RampingInject(rate: 5, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(10)),
                Simulation.Inject(rate: 50, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(20)),
                Simulation.RampingInject(rate: 0, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(5))
            );

        var stats = NBomberRunner
            .RegisterScenarios(scenario)
            .Run();

        var orderStats = stats.ScenarioStats[0];

        // Order placement P95 ต้องน้อยกว่า 1000ms
        orderStats.Ok.Latency.Percent95.Should().BeLessThan(1000);

        // Error rate ต้องเป็น 0% สำหรับ order placement
        orderStats.Fail.Request.Percent.Should().BeLessThan(0.5);
    }
}
```

### NBomber Custom Plugins สำหรับ Reporting

```csharp
// tests/ShopThai.Performance.Tests/Infrastructure/ShopThaiLoadTestConfig.cs
namespace ShopThai.Performance.Tests.Infrastructure;

public static class ShopThaiLoadTestConfig
{
    public static class SLA
    {
        // Product Search SLA
        public const double ProductSearchP99Ms = 500.0;
        public const double ProductSearchP95Ms = 300.0;
        public const double ProductSearchMaxErrorRatePercent = 1.0;
        public const double ProductSearchMinRps = 50.0;

        // Order Placement SLA
        public const double OrderPlacementP95Ms = 1000.0;
        public const double OrderPlacementMaxErrorRatePercent = 0.5;

        // Payment Processing SLA
        public const double PaymentProcessingP99Ms = 3000.0;
        public const double PaymentProcessingMaxErrorRatePercent = 0.1;
    }

    public static class Scenarios
    {
        public const int NormalLoad_UsersPerSecond = 10;
        public const int PeakLoad_UsersPerSecond = 100;
        public const int StressLoad_UsersPerSecond = 500;
    }
}
```

---

## Step 817: Architecture Tests ด้วย NetArchTest

### ติดตั้งและตั้งค่า

```bash
cd tests/ShopThai.Architecture.Tests
dotnet add package NetArchTest.Rules --version 1.3.2
```

### Architecture Constraint Tests

```csharp
// tests/ShopThai.Architecture.Tests/LayerDependencyTests.cs
using NetArchTest.Rules;
using FluentAssertions;
using System.Reflection;

namespace ShopThai.Architecture.Tests;

public class LayerDependencyTests
{
    // Assemblies ที่จะทดสอบ
    private static readonly Assembly DomainAssembly = 
        typeof(ShopThai.Domain.Orders.Order).Assembly;
    private static readonly Assembly ApplicationAssembly = 
        typeof(ShopThai.Application.Orders.Commands.PlaceOrderCommand).Assembly;
    private static readonly Assembly InfrastructureAssembly = 
        typeof(ShopThai.Infrastructure.Persistence.ShopThaiDbContext).Assembly;
    private static readonly Assembly ApiAssembly = 
        typeof(ShopThai.API.Program).Assembly;

    // ─── Domain Layer Rules ───────────────────────────────────────────────

    [Fact]
    public void Domain_ShouldNot_HaveDependencyOn_Application()
    {
        var result = Types.InAssembly(DomainAssembly)
            .ShouldNot()
            .HaveDependencyOn("ShopThai.Application")
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Domain layer must not depend on Application layer");
    }

    [Fact]
    public void Domain_ShouldNot_HaveDependencyOn_Infrastructure()
    {
        var result = Types.InAssembly(DomainAssembly)
            .ShouldNot()
            .HaveDependencyOn("ShopThai.Infrastructure")
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Domain layer must not depend on Infrastructure layer");
    }

    [Fact]
    public void Domain_ShouldNot_HaveDependencyOn_API()
    {
        var result = Types.InAssembly(DomainAssembly)
            .ShouldNot()
            .HaveDependencyOn("ShopThai.API")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    // ─── Application Layer Rules ──────────────────────────────────────────

    [Fact]
    public void Application_ShouldNot_HaveDependencyOn_Infrastructure()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .ShouldNot()
            .HaveDependencyOn("ShopThai.Infrastructure")
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Application layer must not directly depend on Infrastructure — use interfaces");
    }

    [Fact]
    public void Application_ShouldNot_HaveDependencyOn_API()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .ShouldNot()
            .HaveDependencyOn("ShopThai.API")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    // ─── Naming Convention Rules ──────────────────────────────────────────

    [Fact]
    public void CommandHandlers_Should_HaveHandlerSuffix()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .That()
            .ImplementInterface(typeof(MediatR.IRequestHandler<,>))
            .Should()
            .HaveNameEndingWith("Handler")
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "All command/query handlers must have 'Handler' suffix");
    }

    [Fact]
    public void Commands_ShouldBe_InCommandsNamespace()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .That()
            .HaveNameEndingWith("Command")
            .Should()
            .ResideInNamespaceContaining("Commands")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Queries_ShouldBe_InQueriesNamespace()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .That()
            .HaveNameEndingWith("Query")
            .Should()
            .ResideInNamespaceContaining("Queries")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void DomainEvents_Should_HaveEventSuffix()
    {
        var result = Types.InAssembly(DomainAssembly)
            .That()
            .ImplementInterface(typeof(ShopThai.Domain.Common.IDomainEvent))
            .Should()
            .HaveNameEndingWith("Event")
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "All domain events must have 'Event' suffix");
    }

    // ─── Encapsulation Rules ──────────────────────────────────────────────

    [Fact]
    public void AggregateRoots_Should_HavePrivateConstructor()
    {
        var result = Types.InAssembly(DomainAssembly)
            .That()
            .Inherit(typeof(ShopThai.Domain.Common.AggregateRoot))
            .Should()
            .NotBePublic()  // private constructor accessible only via factory methods
            .GetResult();

        // Note: ใช้ reflection ตรวจสอบเพิ่มเติม
        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Repositories_Should_OnlyBeIn_Infrastructure()
    {
        // Repository implementations ต้องอยู่ใน Infrastructure เท่านั้น
        var result = Types.InAssemblies(new[] { DomainAssembly, ApplicationAssembly, ApiAssembly })
            .That()
            .HaveNameEndingWith("Repository")
            .And()
            .AreNotInterfaces()
            .Should()
            .NotExist()
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Concrete repository implementations must only be in Infrastructure layer");
    }

    [Fact]
    public void Controllers_Should_OnlyBeIn_API()
    {
        var result = Types.InAssemblies(new[] { DomainAssembly, ApplicationAssembly, InfrastructureAssembly })
            .That()
            .HaveNameEndingWith("Controller")
            .Should()
            .NotExist()
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Controllers must only reside in the API layer");
    }

    // ─── Interface Rules ──────────────────────────────────────────────────

    [Fact]
    public void Interfaces_InDomain_ShouldStartWithI()
    {
        var result = Types.InAssembly(DomainAssembly)
            .That()
            .AreInterfaces()
            .Should()
            .HaveNameStartingWith("I")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void ValueObjects_Should_BeSealed()
    {
        var result = Types.InAssembly(DomainAssembly)
            .That()
            .Inherit(typeof(ShopThai.Domain.Common.ValueObject))
            .Should()
            .BeSealed()
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Value objects should be sealed to prevent inheritance");
    }
}
```

---

## Step 818: Contract Tests สำหรับ Payment Webhook

### แนวคิด Contract Testing

Contract tests ตรวจสอบว่า webhook ที่ payment provider ส่งมา (เช่น Omise / Stripe) มีรูปแบบตรงกับที่ระบบของเราคาดหวัง ป้องกันการ break เมื่อ provider เปลี่ยน API

```bash
cd tests/ShopThai.Contract.Tests
dotnet add package WireMock.Net --version 1.6.5
dotnet add package Microsoft.AspNetCore.Mvc.Testing
```

```csharp
// tests/ShopThai.Contract.Tests/Payment/PaymentWebhookContractTests.cs
using System.Net;
using System.Text;
using System.Text.Json;
using FluentAssertions;
using WireMock.Server;
using WireMock.RequestBuilders;
using WireMock.ResponseBuilders;
using ShopThai.Integration.Tests.Infrastructure;

namespace ShopThai.Contract.Tests.Payment;

/// <summary>
/// ทดสอบว่า webhook handler ของเราจัดการ payload ที่ได้รับจาก Omise ได้ถูกต้อง
/// </summary>
public class PaymentWebhookContractTests : IntegrationTestBase
{
    public PaymentWebhookContractTests(ShopThaiWebFactory factory) : base(factory) { }

    [Fact]
    public async Task POST_PaymentWebhook_WithChargeSucceededEvent_ShouldReturn200()
    {
        // Arrange — สร้าง payload ที่ตรงกับ Omise's charge.complete webhook
        var orderId = Guid.NewGuid();
        await SeedDataAsync(async db =>
        {
            // สร้าง pending order รอการชำระ
            var order = OrderFakeFactory.PendingOrderForCustomer(Guid.NewGuid(), orderId: orderId);
            db.Orders.Add(order);
            await db.SaveChangesAsync();
        });

        var webhookPayload = new OmiseWebhookPayload
        {
            Key = "charge.complete",
            Data = new OmiseChargeData
            {
                Id = $"chrg_{Guid.NewGuid():N}",
                Status = "successful",
                Amount = 59900, // 599.00 THB in satangs
                Currency = "thb",
                Metadata = new Dictionary<string, string>
                {
                    { "order_id", orderId.ToString() }
                }
            }
        };

        var json = JsonSerializer.Serialize(webhookPayload);
        var signature = ComputeOmiseSignature(json, "test_webhook_secret");

        var content = new StringContent(json, Encoding.UTF8, "application/json");
        Client.DefaultRequestHeaders.Add("OmiseNet-Signature", signature);

        // Act
        var response = await Client.PostAsync("/api/v1/webhooks/payment", content);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }

    [Fact]
    public async Task POST_PaymentWebhook_WithInvalidSignature_ShouldReturn401()
    {
        // Arrange
        var json = JsonSerializer.Serialize(new { key = "charge.complete" });
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        Client.DefaultRequestHeaders.Add("OmiseNet-Signature", "invalid_signature");

        // Act
        var response = await Client.PostAsync("/api/v1/webhooks/payment", content);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task POST_PaymentWebhook_WithChargeFailedEvent_ShouldReturn200()
    {
        // Arrange
        var orderId = Guid.NewGuid();
        await SeedDataAsync(async db =>
        {
            db.Orders.Add(OrderFakeFactory.PendingOrderForCustomer(Guid.NewGuid(), orderId: orderId));
            await db.SaveChangesAsync();
        });

        var payload = new OmiseWebhookPayload
        {
            Key = "charge.complete",
            Data = new OmiseChargeData
            {
                Id = $"chrg_{Guid.NewGuid():N}",
                Status = "failed",
                FailureCode = "insufficient_fund",
                Metadata = new Dictionary<string, string> { { "order_id", orderId.ToString() } }
            }
        };

        var json = JsonSerializer.Serialize(payload);
        var signature = ComputeOmiseSignature(json, "test_webhook_secret");
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        Client.DefaultRequestHeaders.Add("OmiseNet-Signature", signature);

        // Act
        var response = await Client.PostAsync("/api/v1/webhooks/payment", content);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);

        // ตรวจสอบว่า order status ถูก update เป็น PaymentFailed
        using var scope = Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<ShopThaiDbContext>();
        var order = await db.Orders.FindAsync(orderId);
        order!.Status.Should().Be(OrderStatus.PaymentFailed);
    }

    private static string ComputeOmiseSignature(string payload, string secret)
    {
        using var hmac = new System.Security.Cryptography.HMACSHA256(
            Encoding.UTF8.GetBytes(secret));
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(payload));
        return Convert.ToHexString(hash).ToLowerInvariant();
    }
}

public record OmiseWebhookPayload
{
    [JsonPropertyName("key")] public string Key { get; init; } = "";
    [JsonPropertyName("data")] public OmiseChargeData Data { get; init; } = new();
}

public record OmiseChargeData
{
    [JsonPropertyName("id")] public string Id { get; init; } = "";
    [JsonPropertyName("status")] public string Status { get; init; } = "";
    [JsonPropertyName("amount")] public int Amount { get; init; }
    [JsonPropertyName("currency")] public string Currency { get; init; } = "thb";
    [JsonPropertyName("failure_code")] public string? FailureCode { get; init; }
    [JsonPropertyName("metadata")] public Dictionary<string, string> Metadata { get; init; } = new();
}
```

---

## Step 819: Test Data Seeding และ Factories

### Bogus Data Factories

```csharp
// tests/ShopThai.Integration.Tests/Factories/ProductFakeFactory.cs
using Bogus;
using ShopThai.Domain.Products;
using ShopThai.Domain.Common;

namespace ShopThai.Integration.Tests.Factories;

public static class ProductFakeFactory
{
    private static readonly Faker<Product> _faker = new Faker<Product>("th")
        .CustomInstantiator(f => Product.Create(
            f.Commerce.ProductName(),
            new Money(
                Math.Round((decimal)f.Random.Double(50, 10000), 2),
                "THB"),
            f.Random.Int(0, 500)
        ));

    public static Product Generate() => _faker.Generate();

    public static List<Product> Generate(int count) => _faker.Generate(count);

    public static Product WithName(string name)
    {
        var faker = new Faker("th");
        return Product.Create(
            name,
            new Money(Math.Round((decimal)faker.Random.Double(50, 5000), 2), "THB"),
            faker.Random.Int(10, 200));
    }

    public static Product WithPrice(decimal price)
    {
        var faker = new Faker("th");
        return Product.Create(
            faker.Commerce.ProductName(),
            new Money(price, "THB"),
            faker.Random.Int(10, 200));
    }

    public static Product OutOfStock()
    {
        var faker = new Faker("th");
        return Product.Create(
            faker.Commerce.ProductName(),
            new Money(faker.Random.Decimal(50, 5000), "THB"),
            stock: 0);
    }
}
```

```csharp
// tests/ShopThai.Integration.Tests/Factories/OrderFakeFactory.cs
using Bogus;
using ShopThai.Domain.Orders;
using ShopThai.Domain.Common;

namespace ShopThai.Integration.Tests.Factories;

public static class OrderFakeFactory
{
    private static readonly Faker Faker = new("th");

    public static Order PendingOrderForCustomer(Guid customerId, Guid? orderId = null)
    {
        var address = new Address(
            Faker.Address.StreetAddress(),
            Faker.Address.City(),
            Faker.Address.ZipCode(),
            "TH"
        );

        var order = Order.Create(
            CustomerId.Create(customerId),
            address
        );

        // ถ้าต้องการ set orderId เฉพาะเจาะจง
        if (orderId.HasValue)
        {
            // ใช้ reflection สำหรับ test purposes
            typeof(Order)
                .GetProperty(nameof(Order.Id))!
                .SetValue(order, orderId.Value);
        }

        // เพิ่ม items
        var productId = ProductId.Create(Guid.NewGuid());
        order.AddItem(productId, "สินค้าทดสอบ", new Money(199.99m, "THB"), 1);

        return order;
    }

    public static List<Order> GenerateForCustomer(Guid customerId, int count = 3)
    {
        return Enumerable.Range(0, count)
            .Select(_ => PendingOrderForCustomer(customerId))
            .ToList();
    }

    public static Order ConfirmedOrderForCustomer(Guid customerId)
    {
        var order = PendingOrderForCustomer(customerId);
        order.Confirm();
        return order;
    }
}
```

### OrderBuilder — Fluent Test Builder

```csharp
// tests/ShopThai.Domain.Tests/Builders/OrderBuilder.cs
using ShopThai.Domain.Orders;
using ShopThai.Domain.Common;

namespace ShopThai.Domain.Tests.Builders;

/// <summary>
/// Fluent builder สำหรับสร้าง Order objects ในการทดสอบ
/// รองรับการสร้าง orders ในสถานะต่างๆ อย่างสะดวก
/// </summary>
public class OrderBuilder
{
    private CustomerId _customerId = CustomerId.Create(Guid.NewGuid());
    private Address _address = new("123 ถนน", "กรุงเทพฯ", "10000", "TH");
    private readonly List<(ProductId productId, string name, Money price, int qty)> _items = new();
    private OrderStatus _targetStatus = OrderStatus.Pending;

    public OrderBuilder WithCustomer(CustomerId customerId)
    {
        _customerId = customerId;
        return this;
    }

    public OrderBuilder WithCustomer(Guid customerId)
    {
        _customerId = CustomerId.Create(customerId);
        return this;
    }

    public OrderBuilder WithAddress(Address address)
    {
        _address = address;
        return this;
    }

    public OrderBuilder WithAddress(string street, string city, string postalCode, string country = "TH")
    {
        _address = new Address(street, city, postalCode, country);
        return this;
    }

    public OrderBuilder WithItem(ProductId productId, string name, Money unitPrice, int quantity)
    {
        _items.Add((productId, name, unitPrice, quantity));
        return this;
    }

    public OrderBuilder WithItem(string name, decimal price, int quantity = 1)
    {
        _items.Add((ProductId.Create(Guid.NewGuid()), name, new Money(price, "THB"), quantity));
        return this;
    }

    public OrderBuilder WithItems(int count)
    {
        for (var i = 1; i <= count; i++)
        {
            _items.Add((
                ProductId.Create(Guid.NewGuid()),
                $"สินค้า {i}",
                new Money(100m * i, "THB"),
                i
            ));
        }
        return this;
    }

    public OrderBuilder InConfirmedState()
    {
        _targetStatus = OrderStatus.Confirmed;
        return this;
    }

    public OrderBuilder InCancelledState()
    {
        _targetStatus = OrderStatus.Cancelled;
        return this;
    }

    public OrderBuilder InShippedState()
    {
        _targetStatus = OrderStatus.Shipped;
        return this;
    }

    public OrderBuilder InDeliveredState()
    {
        _targetStatus = OrderStatus.Delivered;
        return this;
    }

    public OrderBuilder InStatus(OrderStatus status)
    {
        _targetStatus = status;
        return this;
    }

    public Order Build()
    {
        // ถ้าไม่มี items แต่ต้องการสถานะที่ไม่ใช่ Pending ให้เพิ่ม default item
        if (!_items.Any() && _targetStatus != OrderStatus.Pending)
        {
            _items.Add((
                ProductId.Create(Guid.NewGuid()),
                "สินค้าเริ่มต้น",
                new Money(100m, "THB"),
                1
            ));
        }

        var order = Order.Create(_customerId, _address);

        foreach (var (productId, name, price, qty) in _items)
        {
            order.AddItem(productId, name, price, qty);
        }

        // Transition ไปยัง target status
        order = TransitionToStatus(order, _targetStatus);

        return order;
    }

    private static Order TransitionToStatus(Order order, OrderStatus target)
    {
        return target switch
        {
            OrderStatus.Pending => order,
            OrderStatus.Confirmed => ConfirmOrder(order),
            OrderStatus.Cancelled => CancelOrder(order),
            OrderStatus.Shipped => ShipOrder(order),
            OrderStatus.Delivered => DeliverOrder(order),
            _ => throw new ArgumentOutOfRangeException(nameof(target))
        };
    }

    private static Order ConfirmOrder(Order order)
    {
        order.Confirm();
        return order;
    }

    private static Order CancelOrder(Order order)
    {
        order.Cancel("ยกเลิกเพื่อการทดสอบ");
        return order;
    }

    private static Order ShipOrder(Order order)
    {
        order.Confirm();
        // ใช้ reflection เพื่อ set status ที่ยังไม่มี public method
        SetStatusViaReflection(order, OrderStatus.Shipped);
        return order;
    }

    private static Order DeliverOrder(Order order)
    {
        order.Confirm();
        SetStatusViaReflection(order, OrderStatus.Delivered);
        return order;
    }

    private static void SetStatusViaReflection(Order order, OrderStatus status)
    {
        typeof(Order)
            .GetProperty(nameof(Order.Status))!
            .SetValue(order, status);
    }
}
```

### Database Seeder สำหรับ Integration Tests

```csharp
// tests/ShopThai.Integration.Tests/Infrastructure/DatabaseSeeder.cs
using ShopThai.Domain.Common;
using ShopThai.Domain.Products;
using ShopThai.Domain.Orders;
using ShopThai.Infrastructure.Persistence;

namespace ShopThai.Integration.Tests.Infrastructure;

/// <summary>
/// เพิ่มข้อมูลพื้นฐานที่ใช้ร่วมกันใน integration tests
/// </summary>
public class DatabaseSeeder
{
    private readonly ShopThaiDbContext _db;

    public DatabaseSeeder(ShopThaiDbContext db)
    {
        _db = db;
    }

    public async Task<SeedResult> SeedBasicDataAsync()
    {
        var categories = await SeedCategoriesAsync();
        var products = await SeedProductsAsync(categories);
        return new SeedResult(categories, products);
    }

    private async Task<List<Category>> SeedCategoriesAsync()
    {
        var categories = new List<Category>
        {
            Category.Create("เสื้อผ้า", "clothing"),
            Category.Create("รองเท้า", "shoes"),
            Category.Create("กระเป๋า", "bags"),
            Category.Create("อุปกรณ์อิเล็กทรอนิกส์", "electronics"),
        };

        _db.Categories.AddRange(categories);
        await _db.SaveChangesAsync();
        return categories;
    }

    private async Task<List<Product>> SeedProductsAsync(List<Category> categories)
    {
        var clothing = categories.First(c => c.Slug == "clothing");
        var electronics = categories.First(c => c.Slug == "electronics");

        var products = new List<Product>
        {
            Product.Create("เสื้อยืดสีขาว", new Money(299m, "THB"), 100, clothing.Id),
            Product.Create("เสื้อยืดสีดำ", new Money(299m, "THB"), 80, clothing.Id),
            Product.Create("กางเกงยีนส์", new Money(890m, "THB"), 50, clothing.Id),
            Product.Create("หูฟังไร้สาย", new Money(2990m, "THB"), 30, electronics.Id),
            Product.Create("สายชาร์จ USB-C", new Money(299m, "THB"), 200, electronics.Id),
        };

        _db.Products.AddRange(products);
        await _db.SaveChangesAsync();
        return products;
    }
}

public record SeedResult(List<Category> Categories, List<Product> Products);
```

---

## Step 820: CI Pipeline พร้อม Test Coverage Gate

### GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI — Build, Test & Coverage

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: '9.0.x'
  SOLUTION_PATH: 'ShopThai.sln'
  COVERAGE_THRESHOLD: 80

jobs:
  # ─── Job 1: Build ─────────────────────────────────────────────────────────
  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Restore dependencies
        run: dotnet restore ${{ env.SOLUTION_PATH }}

      - name: Build
        run: dotnet build ${{ env.SOLUTION_PATH }} --no-restore --configuration Release

  # ─── Job 2: Unit Tests ────────────────────────────────────────────────────
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restore
        run: dotnet restore ${{ env.SOLUTION_PATH }}

      - name: Run Domain Tests
        run: |
          dotnet test tests/ShopThai.Domain.Tests \
            --no-restore \
            --configuration Release \
            --logger "trx;LogFileName=domain-tests.trx" \
            --collect:"XPlat Code Coverage" \
            --results-directory ./TestResults/domain

      - name: Run Application Tests
        run: |
          dotnet test tests/ShopThai.Application.Tests \
            --no-restore \
            --configuration Release \
            --logger "trx;LogFileName=application-tests.trx" \
            --collect:"XPlat Code Coverage" \
            --results-directory ./TestResults/application

      - name: Run Architecture Tests
        run: |
          dotnet test tests/ShopThai.Architecture.Tests \
            --no-restore \
            --configuration Release \
            --logger "trx;LogFileName=arch-tests.trx" \
            --results-directory ./TestResults/arch

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: unit-test-results
          path: TestResults/

  # ─── Job 3: Integration Tests ─────────────────────────────────────────────
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: build
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: shopthai_test
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpassword
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restore
        run: dotnet restore ${{ env.SOLUTION_PATH }}

      - name: Run Integration Tests
        env:
          ConnectionStrings__DefaultConnection: "Host=localhost;Database=shopthai_test;Username=testuser;Password=testpassword"
        run: |
          dotnet test tests/ShopThai.Integration.Tests \
            --no-restore \
            --configuration Release \
            --logger "trx;LogFileName=integration-tests.trx" \
            --collect:"XPlat Code Coverage" \
            --results-directory ./TestResults/integration

      - name: Run Contract Tests
        run: |
          dotnet test tests/ShopThai.Contract.Tests \
            --no-restore \
            --configuration Release \
            --logger "trx;LogFileName=contract-tests.trx" \
            --collect:"XPlat Code Coverage" \
            --results-directory ./TestResults/contract

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: integration-test-results
          path: TestResults/

  # ─── Job 4: Coverage Report & Gate ───────────────────────────────────────
  coverage:
    name: Coverage Report & Gate
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Download all test results
        uses: actions/download-artifact@v4
        with:
          path: TestResults/

      - name: Install ReportGenerator
        run: dotnet tool install --global dotnet-reportgenerator-globaltool

      - name: Generate Coverage Report
        run: |
          reportgenerator \
            -reports:"TestResults/**/*.xml" \
            -targetdir:"CoverageReport" \
            -reporttypes:"Html;Cobertura;Badges;MarkdownSummaryGithub" \
            -assemblyfilters:"+ShopThai.Domain;+ShopThai.Application;+ShopThai.Infrastructure;+ShopThai.API" \
            -classfilters:"-*Migrations*;-*Program"

      - name: Check Coverage Gate
        run: |
          COVERAGE=$(cat CoverageReport/Summary.xml | grep -oP '(?<=line-rate=")[^"]*' | head -1)
          COVERAGE_PERCENT=$(echo "$COVERAGE * 100" | bc)
          echo "Coverage: ${COVERAGE_PERCENT}%"

          if (( $(echo "$COVERAGE_PERCENT < ${{ env.COVERAGE_THRESHOLD }}" | bc -l) )); then
            echo "❌ Coverage ${COVERAGE_PERCENT}% is below threshold ${{ env.COVERAGE_THRESHOLD }}%"
            exit 1
          else
            echo "✅ Coverage ${COVERAGE_PERCENT}% passes threshold ${{ env.COVERAGE_THRESHOLD }}%"
          fi

      - name: Add Coverage Summary to PR
        if: github.event_name == 'pull_request'
        uses: marocchino/sticky-pull-request-comment@v2
        with:
          path: CoverageReport/SummaryGithub.md

      - name: Upload Coverage Report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: CoverageReport/

      - name: Upload Coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: CoverageReport/Cobertura.xml
          fail_ci_if_error: true
          token: ${{ secrets.CODECOV_TOKEN }}

  # ─── Job 5: Notify on Failure ─────────────────────────────────────────────
  notify:
    name: Notify on Failure
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests, coverage]
    if: failure()
    steps:
      - name: Send Slack notification
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "❌ CI Pipeline Failed for ShopThai",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*ShopThai CI Failed* ❌\nBranch: `${{ github.ref_name }}`\nCommit: `${{ github.sha }}`\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Details>"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Coverlet Configuration

```xml
<!-- coverlet.runsettings — ใส่ที่ root ของ solution -->
<?xml version="1.0" encoding="utf-8" ?>
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="XPlat code coverage">
        <Configuration>
          <Format>opencover,cobertura</Format>
          <Exclude>
            [*]*.Migrations.*,
            [*]*Program,
            [*]*Startup,
            [*]*.Tests.*
          </Exclude>
          <ExcludeByAttribute>
            Obsolete,
            GeneratedCodeAttribute,
            CompilerGeneratedAttribute,
            ExcludeFromCodeCoverageAttribute
          </ExcludeByAttribute>
          <ExcludeByFile>
            **/Migrations/**/*.cs,
            **/obj/**/*.cs
          </ExcludeByFile>
          <SingleHit>false</SingleHit>
          <UseSourceLink>true</UseSourceLink>
          <IncludeTestAssembly>false</IncludeTestAssembly>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```

### Local Coverage Script

```bash
#!/bin/bash
# scripts/run-coverage.sh — รัน tests พร้อม coverage report บนเครื่อง local

set -e

echo "🧹 Cleaning previous results..."
rm -rf TestResults/ CoverageReport/

echo "🧪 Running all tests with coverage..."
dotnet test ShopThai.sln \
  --configuration Release \
  --collect:"XPlat Code Coverage" \
  --results-directory TestResults/ \
  --settings coverlet.runsettings \
  -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover

echo "📊 Generating coverage report..."
dotnet tool run reportgenerator \
  -reports:"TestResults/**/*.xml" \
  -targetdir:"CoverageReport" \
  -reporttypes:"Html;Badges" \
  -assemblyfilters:"+ShopThai.Domain;+ShopThai.Application;+ShopThai.Infrastructure;+ShopThai.API"

echo "✅ Coverage report generated at: CoverageReport/index.html"

# เปิด browser อัตโนมัติบน macOS/Linux
if [[ "$OSTYPE" == "darwin"* ]]; then
  open CoverageReport/index.html
elif [[ "$OSTYPE" == "linux-gnu"* ]]; then
  xdg-open CoverageReport/index.html 2>/dev/null || echo "Open CoverageReport/index.html manually"
fi
```

### Makefile สำหรับความสะดวก

```makefile
# Makefile
.PHONY: test test-unit test-integration test-arch coverage clean

test: test-unit test-integration test-arch
	@echo "All tests completed"

test-unit:
	@echo "Running unit tests..."
	dotnet test tests/ShopThai.Domain.Tests --configuration Release --no-restore
	dotnet test tests/ShopThai.Application.Tests --configuration Release --no-restore

test-integration:
	@echo "Running integration tests (requires Docker)..."
	dotnet test tests/ShopThai.Integration.Tests --configuration Release --no-restore
	dotnet test tests/ShopThai.Contract.Tests --configuration Release --no-restore

test-arch:
	@echo "Running architecture tests..."
	dotnet test tests/ShopThai.Architecture.Tests --configuration Release --no-restore

coverage:
	@bash scripts/run-coverage.sh

build:
	dotnet build ShopThai.sln --configuration Release

clean:
	dotnet clean ShopThai.sln
	rm -rf TestResults/ CoverageReport/
```

---

## สรุปภาพรวม Testing Strategy ของ ShopThai

### Test Matrix

| Test Type | Project | Tools | Speed | Target |
|-----------|---------|-------|-------|--------|
| Domain Unit | ShopThai.Domain.Tests | xUnit, FluentAssertions | < 1s | 90% coverage |
| Application Unit | ShopThai.Application.Tests | xUnit, Moq, FluentAssertions | < 2s | 85% coverage |
| Architecture | ShopThai.Architecture.Tests | NetArchTest | < 5s | 100% rules |
| Integration API | ShopThai.Integration.Tests | WebApplicationFactory, TestContainers | 30-60s | 70% coverage |
| Contract | ShopThai.Contract.Tests | WireMock.Net | 10-20s | 100% contracts |
| Performance | ShopThai.Performance.Tests | NBomber | 2-5 min | SLA targets |

### Coverage Gate Summary

```
ShopThai Overall Coverage Gate: ≥ 80%
├── ShopThai.Domain         ≥ 90%   (business rules must be thoroughly tested)
├── ShopThai.Application    ≥ 85%   (use case logic must be well-covered)
├── ShopThai.Infrastructure ≥ 60%   (plumbing code, harder to unit test)
└── ShopThai.API            ≥ 70%   (covered by integration tests)
```

### Checklist ก่อน Merge PR

- [ ] Unit tests ผ่านทั้งหมด
- [ ] Integration tests ผ่านทั้งหมด  
- [ ] Architecture tests ไม่มีละเมิดกฎ
- [ ] Code coverage ≥ 80% (overall)
- [ ] Domain coverage ≥ 90%
- [ ] ไม่มี flaky tests (tests ที่ผลลัพธ์ไม่แน่นอน)
- [ ] Performance tests ผ่าน SLA (รันก่อน deploy ไป production)

---

## แนวปฏิบัติที่ดี (Best Practices)

### 1. ตั้งชื่อ Test อย่างมีความหมาย

```
[Method]_[Scenario]_[ExpectedBehavior]

ตัวอย่างที่ดี:
- Confirm_WithItems_ShouldChangeStatusToConfirmed
- AddItem_ToConfirmedOrder_ShouldThrow
- Handle_InsufficientStock_ShouldReturnFailure

ตัวอย่างที่ไม่ดี:
- Test1
- OrderTest
- TestConfirm
```

### 2. หลัก AAA — Arrange, Act, Assert

```csharp
[Fact]
public void Example_WellStructuredTest()
{
    // Arrange — เตรียมข้อมูล
    var order = new OrderBuilder().WithItem("สินค้า", 100m, 2).Build();

    // Act — ดำเนินการ
    order.Confirm();

    // Assert — ตรวจสอบผลลัพธ์
    order.Status.Should().Be(OrderStatus.Confirmed);
}
```

### 3. หลีกเลี่ยง Test Interdependency

```csharp
// ❌ ผิด — tests พึ่งพากัน
public class BadOrderTests
{
    private static Order? _sharedOrder; // shared state ทำให้ tests ขึ้นอยู่กัน

    [Fact]
    public void Test1_CreateOrder() { _sharedOrder = Order.Create(...); }

    [Fact]
    public void Test2_AddItem() { _sharedOrder!.AddItem(...); } // ขึ้นกับ Test1
}

// ✅ ถูก — แต่ละ test สร้างข้อมูลตัวเอง
public class GoodOrderTests
{
    [Fact]
    public void CreateOrder_ShouldSucceed()
    {
        var order = Order.Create(...); // สร้างใหม่ทุกครั้ง
        order.Should().NotBeNull();
    }

    [Fact]
    public void AddItem_ToPendingOrder_ShouldWork()
    {
        var order = Order.Create(...); // สร้างใหม่อีกครั้ง
        order.AddItem(...);
        order.Items.Should().HaveCount(1);
    }
}
```

### 4. ใช้ Theory สำหรับ Parameterized Tests

```csharp
// ✅ ดี — ทดสอบหลาย input ในครั้งเดียว
[Theory]
[InlineData("", false)]
[InlineData("a", false)]
[InlineData("valid@email.com", true)]
[InlineData("invalid-email", false)]
[InlineData("user+tag@domain.co.th", true)]
public void Email_Validation_ShouldMatchExpected(string email, bool expectedValid)
{
    var act = () => new Email(email);
    if (expectedValid)
        act.Should().NotThrow();
    else
        act.Should().Throw<ArgumentException>();
}
```

---

**ก่อนหน้า → [Part 81: Capstone E-Commerce](part81-capstone-ecommerce.md)**  
**ต่อไป → [Part 83: Capstone - DevOps & Deployment](part83-capstone-devops.md)**
