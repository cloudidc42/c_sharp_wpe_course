# Part 73: Advanced Testing (การทดสอบขั้นสูง)

## Steps 721–730 — การทดสอบซอฟต์แวร์ระดับมืออาชีพ

---

## Step 721: Test Pyramid และ Test Doubles

### แนวคิด Test Pyramid

Test Pyramid คือแนวคิดที่ช่วยให้เราจัดสรรการทดสอบได้อย่างเหมาะสม โดยแบ่งออกเป็น 3 ระดับ:

```
         /\
        /E2E\        ← End-to-End Tests (น้อย, ช้า, แพง)
       /------\
      /Integrat\     ← Integration Tests (ปานกลาง)
     /------------\
    / Unit Tests  \  ← Unit Tests (มาก, เร็ว, ถูก)
   /--------------\
```

**หลักการ:**
- **Unit Tests** ทดสอบแต่ละหน่วยแยกกัน ไม่มี I/O หรือ Database จริง
- **Integration Tests** ทดสอบการทำงานร่วมกันของหลาย Component
- **E2E Tests** ทดสอบระบบทั้งหมดจาก UI ถึง Database

### Test Doubles ทั้ง 4 ประเภท

Test Doubles คือ "ตัวแทน" ที่ใช้แทนของจริงในการทดสอบ

```csharp
// ตัวอย่าง Interface ที่จะทดสอบ
public interface IEmailService
{
    Task SendAsync(string to, string subject, string body);
    Task<bool> ValidateEmailAsync(string email);
}

public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id);
    Task SaveAsync(Order order);
    Task<IEnumerable<Order>> GetAllAsync();
}

public class Order
{
    public int Id { get; set; }
    public string CustomerId { get; set; } = "";
    public decimal TotalAmount { get; set; }
    public OrderStatus Status { get; set; }
    public DateTime CreatedAt { get; set; }
}

public enum OrderStatus { Pending, Confirmed, Shipped, Delivered, Cancelled }
```

#### 1. Stub — ส่งคืนค่าที่กำหนดไว้ล่วงหน้า

```csharp
// Stub: คืนค่าที่กำหนดเองโดยไม่มี logic
public class StubOrderRepository : IOrderRepository
{
    private readonly Order? _order;

    public StubOrderRepository(Order? order = null)
    {
        _order = order;
    }

    public Task<Order?> GetByIdAsync(int id) 
        => Task.FromResult(_order);

    public Task SaveAsync(Order order) 
        => Task.CompletedTask;

    public Task<IEnumerable<Order>> GetAllAsync() 
        => Task.FromResult<IEnumerable<Order>>(new List<Order>());
}

[Fact]
public async Task OrderService_GetOrder_ReturnsOrder_WhenExists()
{
    // Arrange
    var expectedOrder = new Order { Id = 1, CustomerId = "C001", TotalAmount = 100m };
    var stub = new StubOrderRepository(expectedOrder);
    var service = new OrderService(stub, new NullEmailService());

    // Act
    var result = await service.GetOrderAsync(1);

    // Assert
    Assert.NotNull(result);
    Assert.Equal(1, result.Id);
}
```

#### 2. Mock — ตรวจสอบว่าเมธอดถูกเรียกหรือไม่

```csharp
// ใช้ Moq library สำหรับ Mock
using Moq;

[Fact]
public async Task OrderService_ConfirmOrder_SendsConfirmationEmail()
{
    // Arrange
    var order = new Order 
    { 
        Id = 1, 
        CustomerId = "customer@example.com", 
        TotalAmount = 500m,
        Status = OrderStatus.Pending
    };
    
    var mockRepo = new Mock<IOrderRepository>();
    mockRepo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(order);
    mockRepo.Setup(r => r.SaveAsync(It.IsAny<Order>())).Returns(Task.CompletedTask);

    var mockEmail = new Mock<IEmailService>();
    mockEmail.Setup(e => e.SendAsync(
        It.IsAny<string>(), 
        It.IsAny<string>(), 
        It.IsAny<string>()
    )).Returns(Task.CompletedTask);

    var service = new OrderService(mockRepo.Object, mockEmail.Object);

    // Act
    await service.ConfirmOrderAsync(1);

    // Assert — ตรวจสอบว่า SendAsync ถูกเรียกครั้งเดียว
    mockEmail.Verify(
        e => e.SendAsync(
            "customer@example.com",
            It.Is<string>(s => s.Contains("Confirmed")),
            It.IsAny<string>()
        ),
        Times.Once
    );
}
```

#### 3. Fake — ใช้ Implementation จริงแต่เบาและง่ายกว่า

```csharp
// Fake: มี logic จริงแต่ใช้ in-memory แทน database
public class FakeOrderRepository : IOrderRepository
{
    private readonly Dictionary<int, Order> _store = new();
    private int _nextId = 1;

    public Task<Order?> GetByIdAsync(int id)
    {
        _store.TryGetValue(id, out var order);
        return Task.FromResult(order);
    }

    public Task SaveAsync(Order order)
    {
        if (order.Id == 0)
            order.Id = _nextId++;
        _store[order.Id] = order;
        return Task.CompletedTask;
    }

    public Task<IEnumerable<Order>> GetAllAsync()
        => Task.FromResult<IEnumerable<Order>>(_store.Values.ToList());

    public int Count => _store.Count;
}

[Fact]
public async Task OrderService_SaveMultipleOrders_AllPersisted()
{
    // Arrange
    var fakeRepo = new FakeOrderRepository();
    var service = new OrderService(fakeRepo, new NullEmailService());

    // Act
    await service.CreateOrderAsync("C001", 100m);
    await service.CreateOrderAsync("C002", 200m);
    await service.CreateOrderAsync("C003", 300m);

    // Assert
    var orders = await fakeRepo.GetAllAsync();
    Assert.Equal(3, orders.Count());
}
```

#### 4. Spy — บันทึกการเรียกเมธอดเพื่อตรวจสอบภายหลัง

```csharp
// Spy: เก็บ log การเรียกเพื่อ assert ทีหลัง
public class SpyEmailService : IEmailService
{
    public List<(string To, string Subject, string Body)> SentEmails { get; } = new();
    public int SendCallCount => SentEmails.Count;

    public Task SendAsync(string to, string subject, string body)
    {
        SentEmails.Add((to, subject, body));
        return Task.CompletedTask;
    }

    public Task<bool> ValidateEmailAsync(string email)
        => Task.FromResult(email.Contains("@"));
}

[Fact]
public async Task OrderService_CancelOrder_SendsCancellationEmail()
{
    // Arrange
    var fakeRepo = new FakeOrderRepository();
    var spyEmail = new SpyEmailService();
    var order = new Order { CustomerId = "user@test.com", TotalAmount = 100m };
    await fakeRepo.SaveAsync(order);
    var service = new OrderService(fakeRepo, spyEmail);

    // Act
    await service.CancelOrderAsync(order.Id);

    // Assert
    Assert.Equal(1, spyEmail.SendCallCount);
    Assert.Contains("Cancelled", spyEmail.SentEmails[0].Subject);
}
```

---

## Step 722: Architecture Tests ด้วย NetArchTest

### ทำไมต้องทดสอบสถาปัตยกรรม?

การทดสอบสถาปัตยกรรมช่วยบังคับให้โค้ดเป็นไปตามหลักการที่ตกลงกันไว้ เช่น Clean Architecture, DDD Layer dependencies ฯลฯ

### ติดตั้ง NetArchTest

```xml
<!-- ใน .csproj ของ test project -->
<PackageReference Include="NetArchTest.Rules" Version="1.3.2" />
```

### ตัวอย่าง Architecture Tests

```csharp
using NetArchTest.Rules;

public class ArchitectureTests
{
    private const string DomainNamespace = "MyApp.Domain";
    private const string ApplicationNamespace = "MyApp.Application";
    private const string InfrastructureNamespace = "MyApp.Infrastructure";
    private const string WebNamespace = "MyApp.Web";

    // กฎ: Domain layer ต้องไม่ขึ้นกับ Application, Infrastructure หรือ Web
    [Fact]
    public void Domain_ShouldNotHaveDependencyOn_ApplicationLayer()
    {
        var result = Types.InAssembly(typeof(Domain.Entities.Order).Assembly)
            .That()
            .ResideInNamespace(DomainNamespace)
            .ShouldNot()
            .HaveDependencyOn(ApplicationNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful, 
            $"Domain layer depends on Application: {string.Join(", ", result.FailingTypes?.Select(t => t.Name) ?? Array.Empty<string>())}");
    }

    [Fact]
    public void Domain_ShouldNotHaveDependencyOn_InfrastructureLayer()
    {
        var result = Types.InAssembly(typeof(Domain.Entities.Order).Assembly)
            .That()
            .ResideInNamespace(DomainNamespace)
            .ShouldNot()
            .HaveDependencyOn(InfrastructureNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    // กฎ: Application layer ต้องไม่ขึ้นกับ Infrastructure โดยตรง
    [Fact]
    public void Application_ShouldNotHaveDependencyOn_Infrastructure()
    {
        var result = Types.InAssembly(typeof(Application.Commands.CreateOrderCommand).Assembly)
            .That()
            .ResideInNamespace(ApplicationNamespace)
            .ShouldNot()
            .HaveDependencyOn(InfrastructureNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    // กฎ: Controllers ต้องอยู่ใน namespace ที่ถูกต้อง
    [Fact]
    public void Controllers_ShouldResideIn_ControllersNamespace()
    {
        var result = Types.InAssembly(typeof(Web.Controllers.OrdersController).Assembly)
            .That()
            .HaveNameEndingWith("Controller")
            .Should()
            .ResideInNamespace($"{WebNamespace}.Controllers")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    // กฎ: Service interfaces ต้องอยู่ใน Application layer
    [Fact]
    public void ServiceInterfaces_ShouldResideIn_ApplicationLayer()
    {
        var result = Types.InAssembly(typeof(Application.Commands.CreateOrderCommand).Assembly)
            .That()
            .HaveNameStartingWith("I")
            .And()
            .HaveNameEndingWith("Service")
            .Should()
            .ResideInNamespace(ApplicationNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    // กฎ: Repository implementations ต้องอยู่ใน Infrastructure
    [Fact]
    public void RepositoryImplementations_ShouldResideIn_InfrastructureLayer()
    {
        var result = Types.InAssembly(typeof(Infrastructure.Repositories.OrderRepository).Assembly)
            .That()
            .HaveNameEndingWith("Repository")
            .And()
            .DoNotHaveNameStartingWith("I")
            .Should()
            .ResideInNamespace(InfrastructureNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    // กฎ: Entity ต้องไม่เป็น static class
    [Fact]
    public void Entities_ShouldNotBeStatic()
    {
        var result = Types.InAssembly(typeof(Domain.Entities.Order).Assembly)
            .That()
            .ResideInNamespace($"{DomainNamespace}.Entities")
            .ShouldNot()
            .BeStatic()
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    // กฎ: Command handlers ต้องสืบทอดจาก Interface ที่ถูกต้อง
    [Fact]
    public void CommandHandlers_ShouldImplementICommandHandler()
    {
        var result = Types.InAssembly(typeof(Application.Commands.CreateOrderCommand).Assembly)
            .That()
            .HaveNameEndingWith("CommandHandler")
            .Should()
            .ImplementInterface(typeof(Application.Interfaces.ICommandHandler<>))
            .GetResult();

        Assert.True(result.IsSuccessful);
    }
}
```

### ตัวอย่างเพิ่มเติมสำหรับ Fluent API

```csharp
public class AdvancedArchitectureTests
{
    // ตรวจสอบ naming conventions
    [Fact]
    public void Validators_ShouldHaveValidatorSuffix()
    {
        var result = Types.InCurrentDomain()
            .That()
            .ImplementInterface(typeof(FluentValidation.IValidator<>))
            .Should()
            .HaveNameEndingWith("Validator")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    // ตรวจสอบว่า class ใน domain ไม่ใช้ Framework
    [Fact]
    public void DomainEntities_ShouldNotDependOn_EntityFramework()
    {
        var result = Types.InCurrentDomain()
            .That()
            .ResideInNamespace("MyApp.Domain")
            .ShouldNot()
            .HaveDependencyOn("Microsoft.EntityFrameworkCore")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    // ตรวจสอบ accessibility
    [Fact]
    public void DomainValueObjects_ShouldBeSealed()
    {
        var result = Types.InCurrentDomain()
            .That()
            .ResideInNamespace("MyApp.Domain.ValueObjects")
            .Should()
            .BeSealed()
            .GetResult();

        Assert.True(result.IsSuccessful);
    }
}
```

---

## Step 723: Contract Testing ด้วย Pact.NET

### Consumer-Driven Contract Testing คืออะไร?

Contract Testing แก้ปัญหา Integration tests ที่ต้องรัน service จริงทั้งหมด โดย:
1. **Consumer** กำหนด "contract" ว่าต้องการ API response แบบไหน
2. **Provider** ทดสอบว่าตัวเองตรงตาม contract

```xml
<PackageReference Include="PactNet" Version="4.5.0" />
<PackageReference Include="PactNet.Output.Xunit" Version="4.5.0" />
```

### Consumer Side — เขียน Contract

```csharp
using PactNet;
using PactNet.Output.Xunit;
using Xunit.Abstractions;

public class OrderApiConsumerTests : IDisposable
{
    private readonly IPactBuilderV4 _pact;
    private readonly int _port = 9001;
    private readonly ITestOutputHelper _output;

    public OrderApiConsumerTests(ITestOutputHelper output)
    {
        _output = output;
        var config = new PactConfig
        {
            PactDir = "../../../pacts",
            Outputters = new[] { new XunitOutput(output) },
            DefaultJsonSettings = new JsonSerializerSettings
            {
                ContractResolver = new CamelCasePropertyNamesContractResolver()
            }
        };

        _pact = Pact.V4("OrderClient", "OrderAPI", config).WithHttpInteractions(_port);
    }

    [Fact]
    public async Task GetOrder_WhenOrderExists_ReturnsOrderDetails()
    {
        // Arrange — กำหนด expected interaction
        _pact
            .UponReceiving("a request for order 1")
            .Given("order with id 1 exists")
            .WithRequest(HttpMethod.Get, "/api/orders/1")
            .WillRespond()
            .WithStatus(HttpStatusCode.OK)
            .WithHeader("Content-Type", "application/json; charset=utf-8")
            .WithJsonBody(new
            {
                id = 1,
                customerId = Match.Type("C001"),
                totalAmount = Match.Decimal(100.50m),
                status = Match.Regex("Pending|Confirmed|Shipped", "Pending")
            });

        await _pact.VerifyAsync(async ctx =>
        {
            // Act — เรียก API client จริง
            var client = new OrderApiClient(ctx.MockServerUri);
            var order = await client.GetOrderAsync(1);

            // Assert
            Assert.NotNull(order);
            Assert.Equal(1, order.Id);
        });
    }

    [Fact]
    public async Task CreateOrder_WithValidData_ReturnsCreated()
    {
        _pact
            .UponReceiving("a request to create an order")
            .Given("customer C001 exists")
            .WithRequest(HttpMethod.Post, "/api/orders")
            .WithHeader("Content-Type", "application/json")
            .WithJsonBody(new
            {
                customerId = "C001",
                totalAmount = 250.00m
            })
            .WillRespond()
            .WithStatus(HttpStatusCode.Created)
            .WithHeader("Location", Match.Regex("/api/orders/\\d+", "/api/orders/1"))
            .WithJsonBody(new
            {
                id = Match.Integer(1),
                customerId = "C001",
                status = "Pending"
            });

        await _pact.VerifyAsync(async ctx =>
        {
            var client = new OrderApiClient(ctx.MockServerUri);
            var result = await client.CreateOrderAsync("C001", 250.00m);

            Assert.NotNull(result);
            Assert.Equal(OrderStatus.Pending, result.Status);
        });
    }

    public void Dispose() => _pact.Dispose();
}
```

### Provider Side — ตรวจสอบ Contract

```csharp
public class OrderApiProviderTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly ITestOutputHelper _output;

    public OrderApiProviderTests(
        WebApplicationFactory<Program> factory,
        ITestOutputHelper output)
    {
        _factory = factory;
        _output = output;
    }

    [Fact]
    public void Provider_ShouldSatisfyAllConsumerContracts()
    {
        // เริ่ม test server
        var server = _factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // ใช้ in-memory database สำหรับ tests
                services.AddDbContext<AppDbContext>(opts =>
                    opts.UseInMemoryDatabase("PactTest"));
            });
        });

        var config = new PactVerifierConfig
        {
            Outputters = new[] { new XunitOutput(_output) }
        };

        string pactFile = Path.Combine("..", "..", "..", "pacts", "OrderClient-OrderAPI.json");

        new PactVerifier(config)
            .ServiceProvider("OrderAPI", server.CreateClient())
            .WithFileSource(new FileInfo(pactFile))
            .WithProviderStateUrl(new Uri("http://localhost/provider-states"))
            .Verify();
    }

    // Provider State Setup
    [Route("provider-states")]
    [ApiController]
    public class ProviderStatesController : ControllerBase
    {
        private readonly AppDbContext _context;

        public ProviderStatesController(AppDbContext context)
        {
            _context = context;
        }

        [HttpPost]
        public async Task<IActionResult> SetupState([FromBody] ProviderState state)
        {
            switch (state.State)
            {
                case "order with id 1 exists":
                    await _context.Orders.AddAsync(new Order 
                    { 
                        Id = 1, 
                        CustomerId = "C001",
                        TotalAmount = 100.50m,
                        Status = OrderStatus.Pending
                    });
                    await _context.SaveChangesAsync();
                    break;

                case "customer C001 exists":
                    await _context.Customers.AddAsync(new Customer 
                    { 
                        Id = "C001", 
                        Email = "customer@example.com" 
                    });
                    await _context.SaveChangesAsync();
                    break;
            }

            return Ok();
        }
    }
}
```

---

## Step 724: Property-Based Testing ด้วย FsCheck

### Property-Based Testing คืออะไร?

แทนที่จะเขียน test case ทีละอัน Property-Based Testing สร้าง input แบบสุ่มหลายพันรายการ เพื่อตรวจสอบว่า "property" หรือคุณสมบัติของโค้ดยังคงเป็นจริง

```xml
<PackageReference Include="FsCheck" Version="2.16.6" />
<PackageReference Include="FsCheck.Xunit" Version="2.16.6" />
```

### ตัวอย่าง Property-Based Tests

```csharp
using FsCheck;
using FsCheck.Xunit;

public class PriceCalculatorPropertyTests
{
    // Property: ราคาหลัง discount ต้องไม่เกินราคาเดิม
    [Property]
    public Property DiscountedPrice_ShouldNeverExceedOriginalPrice(
        decimal originalPrice, 
        int discountPercent)
    {
        // Filter: ราคาต้องเป็นบวก, discount 0-100%
        return (originalPrice > 0m && discountPercent >= 0 && discountPercent <= 100)
            .Implies(() =>
            {
                var calculator = new PriceCalculator();
                var discounted = calculator.ApplyDiscount(originalPrice, discountPercent);
                return discounted <= originalPrice;
            });
    }

    // Property: discount 0% ไม่ควรเปลี่ยนราคา
    [Property]
    public Property ZeroDiscount_ShouldReturnOriginalPrice(PositiveInt amount)
    {
        var price = (decimal)amount.Get;
        var calculator = new PriceCalculator();
        var result = calculator.ApplyDiscount(price, 0);
        return (result == price).ToProperty();
    }

    // Property: discount 100% ควรได้ราคา 0
    [Property]
    public Property FullDiscount_ShouldReturnZero(PositiveInt amount)
    {
        var price = (decimal)amount.Get;
        var calculator = new PriceCalculator();
        var result = calculator.ApplyDiscount(price, 100);
        return (result == 0m).ToProperty();
    }

    // Property: Sort ลิสต์แล้ว sort อีกครั้ง ผลต้องเหมือนกัน (idempotent)
    [Property]
    public Property Sort_IsIdempotent(List<int> numbers)
    {
        var once = numbers.OrderBy(x => x).ToList();
        var twice = once.OrderBy(x => x).ToList();
        return once.SequenceEqual(twice).ToProperty();
    }

    // Property: เพิ่มสินค้าแล้วลบออก ปริมาณต้องเท่าเดิม
    [Property]
    public Property AddThenRemove_ShouldReturnOriginalQuantity(
        int initialQty, 
        int addQty)
    {
        return (initialQty >= 0 && addQty > 0).Implies(() =>
        {
            var inventory = new Inventory(initialQty);
            inventory.Add(addQty);
            inventory.Remove(addQty);
            return inventory.Quantity == initialQty;
        });
    }
}

// Custom generator สำหรับ Domain objects
public class OrderGenerators
{
    public static Arbitrary<Order> OrderArbitrary()
    {
        var gen = from customerId in Arb.Generate<NonEmptyString>()
                  from amount in Gen.Choose(1, 100000).Select(x => (decimal)x)
                  from status in Gen.Elements(Enum.GetValues<OrderStatus>())
                  select new Order
                  {
                      CustomerId = customerId.Get,
                      TotalAmount = amount / 100m,
                      Status = status,
                      CreatedAt = DateTime.UtcNow
                  };
        return Arb.From(gen);
    }
}

public class OrderPropertyTests
{
    public OrderPropertyTests()
    {
        Arb.Register<OrderGenerators>();
    }

    // Property: Serialize แล้ว Deserialize ต้องได้ข้อมูลเหมือนเดิม
    [Property]
    public Property SerializeDeserialize_Roundtrip(Order order)
    {
        var json = JsonSerializer.Serialize(order);
        var deserialized = JsonSerializer.Deserialize<Order>(json);

        return (deserialized!.CustomerId == order.CustomerId &&
                deserialized.TotalAmount == order.TotalAmount &&
                deserialized.Status == order.Status).ToProperty();
    }
}
```

---

## Step 725: Mutation Testing ด้วย Stryker.NET

### Mutation Testing คืออะไร?

Mutation Testing ทดสอบคุณภาพของ tests โดยการ "กลายพันธุ์" โค้ด (เปลี่ยน `>` เป็น `>=`, เปลี่ยน `+` เป็น `-`) แล้วดูว่า tests ของเราจับความผิดพลาดได้หรือไม่

### ติดตั้ง Stryker.NET

```bash
# ติดตั้งแบบ global tool
dotnet tool install -g dotnet-stryker

# รัน mutation testing
dotnet stryker

# รันพร้อม config
dotnet stryker --config-file stryker-config.json
```

### ไฟล์ Configuration

```json
{
  "stryker-config": {
    "project": "MyApp.csproj",
    "test-projects": ["MyApp.Tests/MyApp.Tests.csproj"],
    "reporters": ["html", "json", "console"],
    "threshold-high": 80,
    "threshold-low": 60,
    "threshold-break": 50,
    "mutate": [
      "src/**/*.cs",
      "!src/**/*Migrations*.cs",
      "!src/**/*Designer*.cs"
    ],
    "exclude-mutations": ["String", "Linq"],
    "dashboard-api-key": "",
    "since": false
  }
}
```

### ทำความเข้าใจ Mutation Score

```csharp
// โค้ดต้นฉบับ
public class DiscountService
{
    public decimal CalculateDiscount(decimal price, CustomerTier tier)
    {
        return tier switch
        {
            CustomerTier.Gold => price * 0.20m,      // 20% discount
            CustomerTier.Silver => price * 0.10m,    // 10% discount
            CustomerTier.Bronze => price * 0.05m,    // 5% discount
            _ => 0m
        };
    }

    public bool IsEligibleForFreeShipping(decimal orderTotal)
    {
        return orderTotal >= 1000m;
    }
}

// Tests ที่อาจล้มเหลวกับ mutation
public class DiscountServiceTests
{
    [Theory]
    [InlineData(1000m, CustomerTier.Gold, 200m)]
    [InlineData(1000m, CustomerTier.Silver, 100m)]
    [InlineData(1000m, CustomerTier.Bronze, 50m)]
    [InlineData(1000m, CustomerTier.None, 0m)]
    public void CalculateDiscount_ReturnsCorrectAmount(
        decimal price, CustomerTier tier, decimal expected)
    {
        var service = new DiscountService();
        var result = service.CalculateDiscount(price, tier);
        Assert.Equal(expected, result);
    }

    // ทดสอบ boundary สำหรับ IsEligibleForFreeShipping
    [Theory]
    [InlineData(999.99m, false)]  // ต่ำกว่า threshold
    [InlineData(1000m, true)]     // เท่ากับ threshold
    [InlineData(1000.01m, true)]  // สูงกว่า threshold
    public void IsEligibleForFreeShipping_ChecksBoundary(
        decimal total, bool expected)
    {
        var service = new DiscountService();
        var result = service.IsEligibleForFreeShipping(total);
        Assert.Equal(expected, result);
    }
}
```

### Mutation Survivors — ปัญหาที่พบบ่อย

```csharp
// Mutation: Stryker อาจเปลี่ยน >= เป็น > 
// หากไม่มี test ที่ทดสอบ exact boundary จะไม่ถูกจับ

// Bad test — ไม่จับ boundary mutation
[Fact]
public void IsEligibleForFreeShipping_WhenOver500_ReturnsTrue()
{
    // ใช้ 2000 ซึ่งผ่านทั้ง >= 1000 และ > 1000
    Assert.True(service.IsEligibleForFreeShipping(2000m));
}

// Good test — จับ boundary mutation ได้
[Theory]
[InlineData(999.99m, false)]
[InlineData(1000.00m, true)]  // ← จำเป็นมากเพื่อจับ >= vs > mutation
public void IsEligibleForFreeShipping_Boundary(decimal total, bool expected)
{
    Assert.Equal(expected, service.IsEligibleForFreeShipping(total));
}
```

---

## Step 726: Load Testing ด้วย NBomber

### ติดตั้ง NBomber

```xml
<PackageReference Include="NBomber" Version="5.4.0" />
<PackageReference Include="NBomber.Http" Version="5.4.0" />
```

### Load Test ขั้นพื้นฐาน

```csharp
using NBomber.CSharp;
using NBomber.Http.CSharp;

public class OrderApiLoadTests
{
    [Fact]
    public void OrderApi_ShouldHandleExpectedLoad()
    {
        using var httpClient = new HttpClient();
        httpClient.BaseAddress = new Uri("http://localhost:5000");

        // กำหนด Scenario สำหรับ GET order
        var getOrderScenario = Scenario.Create("get_order", async context =>
        {
            var orderId = Random.Shared.Next(1, 1000);
            var response = await httpClient.GetAsync($"/api/orders/{orderId}");

            return response.IsSuccessStatusCode || response.StatusCode == HttpStatusCode.NotFound
                ? Response.Ok(statusCode: (int)response.StatusCode)
                : Response.Fail(statusCode: (int)response.StatusCode);
        })
        .WithLoadSimulations(
            // Inject 50 req/sec เป็นเวลา 30 วินาที
            Simulation.Inject(rate: 50, interval: TimeSpan.FromSeconds(1), 
                            during: TimeSpan.FromSeconds(30))
        );

        // กำหนด Scenario สำหรับ POST order
        var createOrderScenario = Scenario.Create("create_order", async context =>
        {
            var payload = JsonSerializer.Serialize(new
            {
                customerId = $"C{Random.Shared.Next(1, 100):D3}",
                totalAmount = Random.Shared.NextDouble() * 1000
            });

            var content = new StringContent(payload, Encoding.UTF8, "application/json");
            var response = await httpClient.PostAsync("/api/orders", content);

            return response.StatusCode == HttpStatusCode.Created
                ? Response.Ok()
                : Response.Fail(statusCode: (int)response.StatusCode);
        })
        .WithLoadSimulations(
            // Ramp up จาก 0 ถึง 20 req/sec ใน 10 วินาที
            Simulation.RampingInject(rate: 20, interval: TimeSpan.FromSeconds(1),
                                    during: TimeSpan.FromSeconds(10)),
            // คงที่ที่ 20 req/sec เป็น 20 วินาที
            Simulation.Inject(rate: 20, interval: TimeSpan.FromSeconds(1),
                            during: TimeSpan.FromSeconds(20))
        );

        // รัน Load Test
        var stats = NBomberRunner
            .RegisterScenarios(getOrderScenario, createOrderScenario)
            .WithReportFolder("load-test-reports")
            .WithReportFormats(ReportFormat.Html, ReportFormat.Csv)
            .Run();

        // Assert ผลลัพธ์
        var getStats = stats.ScenarioStats.First(s => s.ScenarioName == "get_order");
        var createStats = stats.ScenarioStats.First(s => s.ScenarioName == "create_order");

        // P99 latency ต้องน้อยกว่า 500ms
        Assert.True(getStats.Ok.Latency.Percent99 < 500,
            $"GET order P99 latency too high: {getStats.Ok.Latency.Percent99}ms");

        // Error rate ต้องน้อยกว่า 1%
        Assert.True(getStats.Fail.Request.Percent < 1.0,
            $"GET order error rate too high: {getStats.Fail.Request.Percent}%");

        Assert.True(createStats.Ok.Latency.Percent99 < 1000,
            $"Create order P99 latency too high: {createStats.Ok.Latency.Percent99}ms");
    }

    [Fact]
    public void OrderApi_StressTest_ShouldDegradeGracefully()
    {
        using var httpClient = new HttpClient();
        httpClient.BaseAddress = new Uri("http://localhost:5000");
        httpClient.Timeout = TimeSpan.FromSeconds(5);

        var scenario = Scenario.Create("stress_test", async context =>
        {
            try
            {
                var response = await httpClient.GetAsync("/api/orders");
                return Response.Ok(statusCode: (int)response.StatusCode);
            }
            catch (TaskCanceledException)
            {
                return Response.Fail(message: "Request timeout");
            }
        })
        .WithLoadSimulations(
            // เพิ่ม concurrent users จาก 10 ถึง 200
            Simulation.RampingConstant(copies: 200, during: TimeSpan.FromSeconds(60))
        );

        var stats = NBomberRunner
            .RegisterScenarios(scenario)
            .Run();

        var result = stats.ScenarioStats.First();

        // แม้ stress สูง error rate ควร < 5%
        Assert.True(result.Fail.Request.Percent < 5.0,
            $"Stress test failure rate too high: {result.Fail.Request.Percent}%");
    }
}
```

### NBomber กับ Data Feed

```csharp
public class LoadTestWithDataFeed
{
    [Fact]
    public void OrderApi_WithRealisticData_HandlesLoad()
    {
        // สร้าง test data ด้วย Bogus
        var faker = new Faker<CreateOrderRequest>()
            .RuleFor(o => o.CustomerId, f => f.Random.AlphaNumeric(8))
            .RuleFor(o => o.TotalAmount, f => f.Finance.Amount(10, 5000));

        var testData = faker.Generate(1000);

        // สร้าง Data Feed
        var feed = DataFeed.Circular(testData);

        using var httpClient = new HttpClient();
        httpClient.BaseAddress = new Uri("http://localhost:5000");

        var scenario = Scenario.Create("order_with_data", async context =>
        {
            var orderData = feed.GetNextItem(context.ScenarioInfo);
            var payload = JsonSerializer.Serialize(orderData);
            var content = new StringContent(payload, Encoding.UTF8, "application/json");

            var response = await httpClient.PostAsync("/api/orders", content);
            return Response.Ok(statusCode: (int)response.StatusCode,
                             dataTransferBytes: payload.Length);
        })
        .WithLoadSimulations(
            Simulation.Inject(rate: 100, interval: TimeSpan.FromSeconds(1),
                            during: TimeSpan.FromSeconds(60))
        );

        NBomberRunner.RegisterScenarios(scenario).Run();
    }
}
```

---

## Step 727: Snapshot Testing

### Snapshot Testing คืออะไร?

Snapshot Testing บันทึก output ของ function ครั้งแรก แล้วเปรียบเทียบกับครั้งต่อๆ ไป เหมาะสำหรับ HTML, JSON response, หรือ complex objects

```xml
<PackageReference Include="Verify.Xunit" Version="22.5.0" />
<PackageReference Include="Verify.Http" Version="6.2.0" />
```

### ตัวอย่าง Snapshot Tests

```csharp
using VerifyXunit;

[UsesVerify]
public class OrderSnapshotTests
{
    // ครั้งแรกที่รัน จะสร้างไฟล์ .verified.txt/.verified.json
    // ครั้งต่อไปจะเปรียบเทียบกับไฟล์นั้น
    [Fact]
    public async Task OrderDto_SerializesToExpectedJson()
    {
        var order = new OrderDto
        {
            Id = 1,
            CustomerId = "C001",
            Items = new[]
            {
                new OrderItemDto { ProductName = "Widget", Quantity = 2, UnitPrice = 25.00m },
                new OrderItemDto { ProductName = "Gadget", Quantity = 1, UnitPrice = 75.00m }
            },
            TotalAmount = 125.00m,
            Status = "Confirmed",
            CreatedAt = new DateTime(2024, 1, 15, 10, 30, 0, DateTimeKind.Utc)
        };

        await Verify(order);
    }

    // ไฟล์ snapshot จะมีลักษณะนี้:
    // OrderSnapshotTests.OrderDto_SerializesToExpectedJson.verified.json
    /*
    {
      "Id": 1,
      "CustomerId": "C001",
      "Items": [
        { "ProductName": "Widget", "Quantity": 2, "UnitPrice": 25.00 },
        { "ProductName": "Gadget", "Quantity": 1, "UnitPrice": 75.00 }
      ],
      "TotalAmount": 125.00,
      "Status": "Confirmed",
      "CreatedAt": "2024-01-15T10:30:00Z"
    }
    */

    [Fact]
    public async Task OrderSummaryHtml_MatchesSnapshot()
    {
        var service = new OrderReportService();
        var html = await service.GenerateOrderSummaryHtmlAsync(new[]
        {
            new Order { Id = 1, TotalAmount = 100m, Status = OrderStatus.Confirmed },
            new Order { Id = 2, TotalAmount = 200m, Status = OrderStatus.Shipped }
        });

        await Verify(html).UseExtension("html");
    }

    [Fact]
    public async Task ApiResponse_ForOrderList_MatchesSnapshot()
    {
        var client = _factory.CreateClient();
        var response = await client.GetAsync("/api/orders?page=1&pageSize=5");

        await Verify(response)
            .IgnoreHeader("Date")     // ข้ามเพราะเปลี่ยนทุกครั้ง
            .IgnoreHeader("X-Request-Id");
    }
}

// การตั้งค่า Global สำหรับ Snapshot Testing
public static class SnapshotConfiguration
{
    [ModuleInitializer]
    public static void Initialize()
    {
        VerifierSettings.DerivePathInfo((sourceFile, projectDirectory, type, method) =>
            new PathInfo(
                directory: Path.Combine(projectDirectory, "Snapshots"),
                typeName: type.Name,
                methodName: method.Name
            ));

        // ข้าม properties ที่ไม่ต้องการ compare
        VerifierSettings.AddExtraSettings(settings =>
        {
            settings.DefaultValueHandling = DefaultValueHandling.Ignore;
        });
    }
}
```

---

## Step 728: Test Data Management ด้วย Bogus และ Builder Pattern

### Builder Pattern สำหรับ Test Data

```csharp
// Domain Objects
public class OrderBuilder
{
    private int _id = 1;
    private string _customerId = "C001";
    private decimal _totalAmount = 100m;
    private OrderStatus _status = OrderStatus.Pending;
    private DateTime _createdAt = DateTime.UtcNow;
    private List<OrderItem> _items = new();

    public static OrderBuilder Default() => new OrderBuilder();

    public OrderBuilder WithId(int id)
    {
        _id = id;
        return this;
    }

    public OrderBuilder WithCustomer(string customerId)
    {
        _customerId = customerId;
        return this;
    }

    public OrderBuilder WithAmount(decimal amount)
    {
        _totalAmount = amount;
        return this;
    }

    public OrderBuilder WithStatus(OrderStatus status)
    {
        _status = status;
        return this;
    }

    public OrderBuilder WithItem(string product, int qty, decimal price)
    {
        _items.Add(new OrderItem 
        { 
            ProductName = product, 
            Quantity = qty, 
            UnitPrice = price 
        });
        return this;
    }

    public OrderBuilder AsConfirmed() => WithStatus(OrderStatus.Confirmed);
    public OrderBuilder AsShipped() => WithStatus(OrderStatus.Shipped);
    public OrderBuilder AsHighValue() => WithAmount(10000m);

    public Order Build() => new Order
    {
        Id = _id,
        CustomerId = _customerId,
        TotalAmount = _totalAmount > 0 && _items.Any() 
            ? _items.Sum(i => i.Quantity * i.UnitPrice) 
            : _totalAmount,
        Status = _status,
        CreatedAt = _createdAt,
        Items = _items
    };
}

// ใช้งาน Builder
[Fact]
public void OrderService_CalculateTax_ForHighValueOrder()
{
    var order = OrderBuilder.Default()
        .WithCustomer("VIP001")
        .AsHighValue()
        .WithItem("Laptop", 2, 2500m)
        .WithItem("Monitor", 1, 800m)
        .Build();

    var tax = taxService.Calculate(order);
    Assert.Equal(expected: order.TotalAmount * 0.07m, actual: tax);
}
```

### Bogus Faker — สร้าง Realistic Test Data

```csharp
using Bogus;

// สร้าง Fake data generator
public static class TestDataFactory
{
    private static readonly Faker<Customer> CustomerFaker = new Faker<Customer>("th")
        .RuleFor(c => c.Id, f => f.Random.AlphaNumeric(8).ToUpper())
        .RuleFor(c => c.FirstName, f => f.Name.FirstName())
        .RuleFor(c => c.LastName, f => f.Name.LastName())
        .RuleFor(c => c.Email, (f, c) => f.Internet.Email(c.FirstName, c.LastName))
        .RuleFor(c => c.Phone, f => f.Phone.PhoneNumber("0#-####-####"))
        .RuleFor(c => c.Address, f => new Address
        {
            Street = f.Address.StreetAddress(),
            City = f.PickRandom("กรุงเทพมหานคร", "เชียงใหม่", "ภูเก็ต", "ขอนแก่น"),
            PostalCode = f.Random.Replace("#####"),
            Country = "Thailand"
        })
        .RuleFor(c => c.RegisteredAt, f => f.Date.Past(3));

    private static readonly Faker<Product> ProductFaker = new Faker<Product>()
        .RuleFor(p => p.Id, f => f.IndexFaker + 1)
        .RuleFor(p => p.Name, f => f.Commerce.ProductName())
        .RuleFor(p => p.Description, f => f.Commerce.ProductDescription())
        .RuleFor(p => p.Price, f => decimal.Parse(f.Commerce.Price()))
        .RuleFor(p => p.Category, f => f.Commerce.Department())
        .RuleFor(p => p.Stock, f => f.Random.Int(0, 500))
        .RuleFor(p => p.IsActive, f => f.Random.Bool(0.9f));  // 90% active

    private static readonly Faker<Order> OrderFaker = new Faker<Order>()
        .RuleFor(o => o.Id, f => f.IndexFaker + 1)
        .RuleFor(o => o.CustomerId, f => f.Random.AlphaNumeric(8).ToUpper())
        .RuleFor(o => o.TotalAmount, f => Math.Round(f.Finance.Amount(50, 5000), 2))
        .RuleFor(o => o.Status, f => f.PickRandom<OrderStatus>())
        .RuleFor(o => o.CreatedAt, f => f.Date.Recent(30))
        .RuleFor(o => o.Notes, f => f.Lorem.Sentence());

    public static Customer GenerateCustomer() => CustomerFaker.Generate();
    public static List<Customer> GenerateCustomers(int count) => CustomerFaker.Generate(count);

    public static Product GenerateProduct() => ProductFaker.Generate();
    public static List<Product> GenerateProducts(int count) => ProductFaker.Generate(count);

    public static Order GenerateOrder() => OrderFaker.Generate();
    public static List<Order> GenerateOrders(int count) => OrderFaker.Generate(count);
}

// ตัวอย่างการใช้งาน
public class CustomerServiceTests
{
    [Fact]
    public async Task BulkImportCustomers_SavesAllRecords()
    {
        // สร้าง 100 customers ที่สมจริง
        var customers = TestDataFactory.GenerateCustomers(100);
        var fakeRepo = new FakeCustomerRepository();
        var service = new CustomerService(fakeRepo);

        await service.BulkImportAsync(customers);

        Assert.Equal(100, await fakeRepo.CountAsync());
    }

    [Theory]
    [MemberData(nameof(GetTestOrders))]
    public async Task ProcessOrder_WithVariousAmounts_CalculatesCorrectly(Order order)
    {
        var service = new OrderProcessingService();
        var result = await service.ProcessAsync(order);
        Assert.True(result.IsSuccess);
    }

    public static IEnumerable<object[]> GetTestOrders()
    {
        return TestDataFactory.GenerateOrders(10)
            .Select(o => new object[] { o });
    }
}
```

### AutoFixture กับ [AutoData]

```csharp
using AutoFixture;
using AutoFixture.AutoMoq;
using AutoFixture.Xunit2;

// Custom AutoData attribute ที่รวม AutoFixture + Moq
public class AutoMoqDataAttribute : AutoDataAttribute
{
    public AutoMoqDataAttribute()
        : base(() =>
        {
            var fixture = new Fixture();
            fixture.Customize(new AutoMoqCustomization { ConfigureMembers = true });
            fixture.Behaviors.Add(new OmitOnRecursionBehavior());
            return fixture;
        })
    { }
}

public class InlineAutoMoqDataAttribute : InlineAutoDataAttribute
{
    public InlineAutoMoqDataAttribute(params object[] values)
        : base(new AutoMoqDataAttribute(), values) { }
}

// ใช้งาน [AutoData] — AutoFixture สร้าง values อัตโนมัติ
public class OrderProcessorTests
{
    // AutoFixture สร้าง Order และ Mock dependencies อัตโนมัติ
    [Theory, AutoMoqData]
    public async Task ProcessOrder_WithValidOrder_CallsRepository(
        Order order,
        [Frozen] Mock<IOrderRepository> mockRepo,
        [Frozen] Mock<IEmailService> mockEmail,
        OrderProcessor sut)  // sut = System Under Test
    {
        // Arrange — กำหนด status ให้ชัดเจน
        order.Status = OrderStatus.Pending;
        mockRepo.Setup(r => r.GetByIdAsync(order.Id)).ReturnsAsync(order);

        // Act
        await sut.ProcessAsync(order.Id);

        // Assert
        mockRepo.Verify(r => r.SaveAsync(It.Is<Order>(o => o.Id == order.Id)), Times.Once);
    }

    [Theory, AutoMoqData]
    public void Order_TotalAmount_CalculatedFromItems(
        [Frozen] List<OrderItem> items,
        OrderCalculator calculator)
    {
        var total = calculator.CalculateTotal(items);
        var expected = items.Sum(i => i.Quantity * i.UnitPrice);
        Assert.Equal(expected, total);
    }
}
```

---

## Step 729: Testing Time-Dependent Code

### ปัญหาของ DateTime.Now ใน Tests

```csharp
// ❌ โค้ดที่ทดสอบยาก — ขึ้นกับเวลาจริง
public class SubscriptionService
{
    public bool IsSubscriptionActive(Subscription subscription)
    {
        return subscription.ExpiresAt > DateTime.UtcNow;  // ← ทดสอบยากมาก!
    }

    public void RenewSubscription(Subscription subscription)
    {
        subscription.RenewedAt = DateTime.UtcNow;  // ← ค่าเปลี่ยนทุกครั้ง
        subscription.ExpiresAt = DateTime.UtcNow.AddDays(30);
    }
}
```

### วิธีที่ 1: ใช้ ITimeProvider Interface (สร้างเอง)

```csharp
// Interface ที่ทดสอบได้
public interface ITimeProvider
{
    DateTimeOffset UtcNow { get; }
    DateTime LocalNow { get; }
}

// Implementation สำหรับ Production
public class SystemTimeProvider : ITimeProvider
{
    public DateTimeOffset UtcNow => DateTimeOffset.UtcNow;
    public DateTime LocalNow => DateTime.Now;
}

// Implementation สำหรับ Tests
public class FakeTimeProvider : ITimeProvider
{
    private DateTimeOffset _currentTime;

    public FakeTimeProvider(DateTimeOffset? initialTime = null)
    {
        _currentTime = initialTime ?? DateTimeOffset.UtcNow;
    }

    public DateTimeOffset UtcNow => _currentTime;
    public DateTime LocalNow => _currentTime.LocalDateTime;

    // เมธอดช่วยสำหรับ tests
    public void Advance(TimeSpan duration) => _currentTime = _currentTime.Add(duration);
    public void SetTo(DateTimeOffset time) => _currentTime = time;
    public void AdvanceDays(int days) => Advance(TimeSpan.FromDays(days));
    public void AdvanceHours(int hours) => Advance(TimeSpan.FromHours(hours));
}

// Service ที่ทดสอบได้
public class SubscriptionService
{
    private readonly ITimeProvider _timeProvider;
    private readonly ISubscriptionRepository _repository;

    public SubscriptionService(ITimeProvider timeProvider, ISubscriptionRepository repository)
    {
        _timeProvider = timeProvider;
        _repository = repository;
    }

    public bool IsSubscriptionActive(Subscription subscription)
    {
        return subscription.ExpiresAt > _timeProvider.UtcNow;
    }

    public async Task RenewSubscriptionAsync(int subscriptionId)
    {
        var subscription = await _repository.GetByIdAsync(subscriptionId)
            ?? throw new NotFoundException($"Subscription {subscriptionId} not found");

        subscription.RenewedAt = _timeProvider.UtcNow;
        subscription.ExpiresAt = _timeProvider.UtcNow.AddDays(30);

        await _repository.SaveAsync(subscription);
    }

    public async Task<IEnumerable<Subscription>> GetExpiringSubscriptionsAsync(int daysAhead)
    {
        var cutoff = _timeProvider.UtcNow.AddDays(daysAhead);
        return await _repository.GetExpiringBeforeAsync(cutoff);
    }
}
```

### วิธีที่ 2: ใช้ TimeProvider จาก .NET 8 (แนะนำ)

```csharp
// .NET 8 มี TimeProvider abstract class built-in
// Microsoft.Extensions.Time.Testing มี FakeTimeProvider

// ติดตั้ง package
// <PackageReference Include="Microsoft.Extensions.TimeProvider.Testing" Version="8.x" />

public class ModernSubscriptionService
{
    private readonly TimeProvider _timeProvider;
    private readonly ISubscriptionRepository _repository;

    // ใช้ TimeProvider.System ใน Production
    public ModernSubscriptionService(
        TimeProvider timeProvider,
        ISubscriptionRepository repository)
    {
        _timeProvider = timeProvider;
        _repository = repository;
    }

    public bool IsActive(Subscription subscription)
    {
        return subscription.ExpiresAt > _timeProvider.GetUtcNow();
    }

    public async Task RenewAsync(int subscriptionId)
    {
        var sub = await _repository.GetByIdAsync(subscriptionId)
            ?? throw new KeyNotFoundException();

        sub.RenewedAt = _timeProvider.GetUtcNow();
        sub.ExpiresAt = _timeProvider.GetUtcNow().AddDays(30);

        await _repository.SaveAsync(sub);
    }
}

// Tests ด้วย Microsoft FakeTimeProvider
public class SubscriptionServiceTimeTests
{
    [Fact]
    public void IsActive_WhenNotExpired_ReturnsTrue()
    {
        // Arrange
        var fakeTime = new FakeTimeProvider();
        fakeTime.SetUtcNow(new DateTimeOffset(2024, 6, 1, 0, 0, 0, TimeSpan.Zero));

        var subscription = new Subscription
        {
            ExpiresAt = new DateTimeOffset(2024, 12, 31, 0, 0, 0, TimeSpan.Zero)
        };

        var service = new ModernSubscriptionService(fakeTime, new FakeSubscriptionRepository());

        // Act
        var isActive = service.IsActive(subscription);

        // Assert
        Assert.True(isActive);
    }

    [Fact]
    public void IsActive_WhenExpired_ReturnsFalse()
    {
        var fakeTime = new FakeTimeProvider();
        fakeTime.SetUtcNow(new DateTimeOffset(2025, 1, 1, 0, 0, 0, TimeSpan.Zero));

        var subscription = new Subscription
        {
            ExpiresAt = new DateTimeOffset(2024, 12, 31, 0, 0, 0, TimeSpan.Zero)
        };

        var service = new ModernSubscriptionService(fakeTime, new FakeSubscriptionRepository());

        Assert.False(service.IsActive(subscription));
    }

    [Fact]
    public async Task Renew_SetsExpirationTo30DaysFromNow()
    {
        // Arrange
        var knownTime = new DateTimeOffset(2024, 6, 15, 12, 0, 0, TimeSpan.Zero);
        var fakeTime = new FakeTimeProvider();
        fakeTime.SetUtcNow(knownTime);

        var fakeRepo = new FakeSubscriptionRepository();
        var sub = new Subscription { Id = 1, ExpiresAt = knownTime.AddDays(-1) };
        await fakeRepo.SaveAsync(sub);

        var service = new ModernSubscriptionService(fakeTime, fakeRepo);

        // Act
        await service.RenewAsync(1);

        // Assert
        var updated = await fakeRepo.GetByIdAsync(1);
        Assert.Equal(knownTime.AddDays(30), updated!.ExpiresAt);
    }

    [Fact]
    public async Task ExpiringSubscriptions_FoundBeforeCutoff()
    {
        // Test ที่ซับซ้อนกว่า — เลื่อนเวลาไปข้างหน้า
        var fakeTime = new FakeTimeProvider();
        var startTime = new DateTimeOffset(2024, 1, 1, 0, 0, 0, TimeSpan.Zero);
        fakeTime.SetUtcNow(startTime);

        var fakeRepo = new FakeSubscriptionRepository();
        await fakeRepo.SaveAsync(new Subscription { Id = 1, ExpiresAt = startTime.AddDays(3) });
        await fakeRepo.SaveAsync(new Subscription { Id = 2, ExpiresAt = startTime.AddDays(10) });
        await fakeRepo.SaveAsync(new Subscription { Id = 3, ExpiresAt = startTime.AddDays(60) });

        var service = new ModernSubscriptionService(fakeTime, fakeRepo);

        // ค้นหา subscriptions ที่จะหมดอายุใน 7 วัน
        var expiring = await service.GetExpiringSubscriptionsAsync(7);

        // ควรพบแค่ subscription id=1 (หมดใน 3 วัน)
        Assert.Single(expiring);
        Assert.Equal(1, expiring.First().Id);
    }

    // TimeProvider.System vs Custom — DI Registration
    public static void ConfigureServices(IServiceCollection services)
    {
        // Production: ใช้เวลาจริง
        services.AddSingleton(TimeProvider.System);
        services.AddScoped<ModernSubscriptionService>();
    }

    public static void ConfigureTestServices(IServiceCollection services)
    {
        // Tests: ใช้ fake time
        var fakeTime = new FakeTimeProvider();
        fakeTime.SetUtcNow(new DateTimeOffset(2024, 1, 1, 0, 0, 0, TimeSpan.Zero));
        services.AddSingleton<TimeProvider>(fakeTime);
        services.AddScoped<ModernSubscriptionService>();
    }
}
```

---

## Step 730: Full Test Suite สำหรับ Web API

### โครงสร้าง Test Project

```
MyApp.Tests/
├── Unit/
│   ├── Services/
│   │   ├── OrderServiceTests.cs
│   │   └── DiscountServiceTests.cs
│   ├── Domain/
│   │   └── OrderTests.cs
│   └── Validators/
│       └── CreateOrderValidatorTests.cs
├── Integration/
│   ├── Api/
│   │   └── OrdersControllerTests.cs
│   └── Infrastructure/
│       └── OrderRepositoryTests.cs
├── Architecture/
│   └── ArchitectureTests.cs
├── Infrastructure/
│   ├── CustomWebApplicationFactory.cs
│   ├── TestAuthHandler.cs
│   └── DatabaseFixture.cs
└── MyApp.Tests.csproj
```

### Custom WebApplicationFactory

```csharp
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.EntityFrameworkCore;

public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    private readonly string _dbName = Guid.NewGuid().ToString();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // แทนที่ real database ด้วย in-memory
            var dbDescriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            
            if (dbDescriptor != null)
                services.Remove(dbDescriptor);

            services.AddDbContext<AppDbContext>(options =>
                options.UseInMemoryDatabase(_dbName));

            // แทนที่ external services ด้วย Fakes
            var emailDescriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(IEmailService));
            
            if (emailDescriptor != null)
                services.Remove(emailDescriptor);

            services.AddSingleton<IEmailService, FakeEmailService>();

            // ใช้ FakeTimeProvider สำหรับ deterministic tests
            services.AddSingleton<TimeProvider>(
                new FakeTimeProvider(new DateTimeOffset(2024, 1, 1, 0, 0, 0, TimeSpan.Zero)));

            // Auth: ใช้ Test authentication
            services.AddAuthentication(TestAuthHandler.SchemeName)
                .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(
                    TestAuthHandler.SchemeName, null);
        });

        builder.UseEnvironment("Testing");
    }

    // Helper: สร้าง client ที่ authenticated
    public HttpClient CreateAuthenticatedClient(string userId = "test-user", string[] roles = null!)
    {
        var client = CreateClient();
        client.DefaultRequestHeaders.Add("X-Test-UserId", userId);
        
        if (roles?.Length > 0)
            client.DefaultRequestHeaders.Add("X-Test-Roles", string.Join(",", roles));
        
        return client;
    }

    // Helper: seed test data
    public async Task SeedDatabaseAsync(Action<AppDbContext> seeder)
    {
        using var scope = Services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await context.Database.EnsureCreatedAsync();
        seeder(context);
        await context.SaveChangesAsync();
    }
}
```

### Test Authentication Handler

```csharp
public class TestAuthHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    public const string SchemeName = "TestScheme";

    public TestAuthHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder,
        ISystemClock clock)
        : base(options, logger, encoder, clock) { }

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        // อ่าน test identity จาก headers
        if (!Request.Headers.TryGetValue("X-Test-UserId", out var userId))
            return Task.FromResult(AuthenticateResult.NoResult());

        var roles = Request.Headers.TryGetValue("X-Test-Roles", out var rolesHeader)
            ? rolesHeader.ToString().Split(',')
            : Array.Empty<string>();

        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, userId!),
            new(ClaimTypes.Name, userId!)
        };

        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var identity = new ClaimsIdentity(claims, SchemeName);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, SchemeName);

        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

### Integration Tests สำหรับ Orders Controller

```csharp
public class OrdersControllerIntegrationTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly CustomWebApplicationFactory _factory;
    private readonly HttpClient _client;

    public OrdersControllerIntegrationTests(CustomWebApplicationFactory factory)
    {
        _factory = factory;
        _client = factory.CreateAuthenticatedClient("user-001", new[] { "Customer" });
    }

    [Fact]
    public async Task GetOrders_ReturnsOk_WithOrderList()
    {
        // Arrange — seed test data
        await _factory.SeedDatabaseAsync(ctx =>
        {
            ctx.Orders.AddRange(
                new Order { CustomerId = "user-001", TotalAmount = 100m, Status = OrderStatus.Confirmed },
                new Order { CustomerId = "user-001", TotalAmount = 200m, Status = OrderStatus.Shipped }
            );
        });

        // Act
        var response = await _client.GetAsync("/api/orders");

        // Assert
        response.EnsureSuccessStatusCode();
        var orders = await response.Content.ReadFromJsonAsync<List<OrderDto>>();
        Assert.NotNull(orders);
        Assert.Equal(2, orders.Count);
    }

    [Fact]
    public async Task CreateOrder_WithValidData_ReturnsCreated()
    {
        // Arrange
        var createRequest = new CreateOrderRequest
        {
            CustomerId = "user-001",
            Items = new[]
            {
                new CreateOrderItemRequest { ProductId = 1, Quantity = 2 }
            }
        };

        // Act
        var response = await _client.PostAsJsonAsync("/api/orders", createRequest);

        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        Assert.NotNull(response.Headers.Location);

        var created = await response.Content.ReadFromJsonAsync<OrderDto>();
        Assert.NotNull(created);
        Assert.Equal("user-001", created.CustomerId);
        Assert.Equal(OrderStatus.Pending.ToString(), created.Status);
    }

    [Fact]
    public async Task CreateOrder_WithoutAuth_ReturnsUnauthorized()
    {
        var unauthClient = _factory.CreateClient(); // ไม่มี auth headers
        var response = await unauthClient.PostAsJsonAsync("/api/orders", new CreateOrderRequest());
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }

    [Fact]
    public async Task ConfirmOrder_WhenNotAdmin_ReturnsForbidden()
    {
        await _factory.SeedDatabaseAsync(ctx =>
        {
            ctx.Orders.Add(new Order { Id = 99, CustomerId = "user-001", Status = OrderStatus.Pending });
        });

        // Customer role ไม่มีสิทธิ์ confirm
        var response = await _client.PostAsync("/api/orders/99/confirm", null);
        Assert.Equal(HttpStatusCode.Forbidden, response.StatusCode);
    }

    [Fact]
    public async Task ConfirmOrder_AsAdmin_ReturnsOk()
    {
        var adminClient = _factory.CreateAuthenticatedClient("admin-001", new[] { "Admin" });
        
        await _factory.SeedDatabaseAsync(ctx =>
        {
            ctx.Orders.Add(new Order { Id = 100, CustomerId = "user-001", Status = OrderStatus.Pending });
        });

        var response = await adminClient.PostAsync("/api/orders/100/confirm", null);
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }

    [Fact]
    public async Task GetOrder_WhenNotFound_ReturnsNotFound()
    {
        var response = await _client.GetAsync("/api/orders/99999");
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }

    [Theory]
    [InlineData("")]
    [InlineData("invalid-data")]
    public async Task CreateOrder_WithInvalidData_ReturnsBadRequest(string customerId)
    {
        var request = new CreateOrderRequest { CustomerId = customerId };
        var response = await _client.PostAsJsonAsync("/api/orders", request);
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    }
}
```

### Unit Tests สำหรับ Business Logic

```csharp
public class OrderServiceUnitTests
{
    private readonly Mock<IOrderRepository> _mockRepo;
    private readonly Mock<IEmailService> _mockEmail;
    private readonly FakeTimeProvider _fakeTime;
    private readonly OrderService _sut;

    public OrderServiceUnitTests()
    {
        _mockRepo = new Mock<IOrderRepository>();
        _mockEmail = new Mock<IEmailService>();
        _fakeTime = new FakeTimeProvider(
            new DateTimeOffset(2024, 6, 1, 0, 0, 0, TimeSpan.Zero));
        
        _sut = new OrderService(_mockRepo.Object, _mockEmail.Object, _fakeTime);
    }

    [Fact]
    public async Task CreateOrder_ValidRequest_SavesAndSendsEmail()
    {
        // Arrange
        _mockRepo.Setup(r => r.SaveAsync(It.IsAny<Order>()))
                 .Returns(Task.CompletedTask);
        _mockEmail.Setup(e => e.SendAsync(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<string>()))
                  .Returns(Task.CompletedTask);

        // Act
        var result = await _sut.CreateOrderAsync("customer@test.com", 150m);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(OrderStatus.Pending, result.Status);
        Assert.Equal(_fakeTime.GetUtcNow(), result.CreatedAt);
        
        _mockRepo.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Once);
        _mockEmail.Verify(e => e.SendAsync(
            "customer@test.com",
            It.Is<string>(s => s.Contains("Order Confirmation")),
            It.IsAny<string>()), Times.Once);
    }

    [Fact]
    public async Task CancelOrder_AlreadyCancelled_ThrowsInvalidOperation()
    {
        var order = new Order { Id = 1, Status = OrderStatus.Cancelled };
        _mockRepo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(order);

        await Assert.ThrowsAsync<InvalidOperationException>(
            () => _sut.CancelOrderAsync(1));
    }

    [Theory]
    [InlineData(OrderStatus.Shipped)]
    [InlineData(OrderStatus.Delivered)]
    public async Task CancelOrder_WhenAlreadyShippedOrDelivered_ThrowsInvalidOperation(
        OrderStatus status)
    {
        var order = new Order { Id = 1, Status = status };
        _mockRepo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(order);

        await Assert.ThrowsAsync<InvalidOperationException>(
            () => _sut.CancelOrderAsync(1));
    }
}
```

### Full Architecture Test Suite

```csharp
public class FullArchitectureTests
{
    [Fact]
    public void AllPublicClasses_InDomain_ShouldBeSealed_OrAbstract()
    {
        var domainAssembly = typeof(Order).Assembly;
        
        var violatingTypes = domainAssembly.GetExportedTypes()
            .Where(t => t.IsClass 
                     && !t.IsAbstract 
                     && !t.IsSealed
                     && t.Namespace?.Contains("ValueObjects") == true)
            .ToList();

        Assert.Empty(violatingTypes);
    }

    [Fact]
    public void AllControllers_ShouldHaveAuthorizeAttribute_OrAllowAnonymous()
    {
        var controllerTypes = typeof(Program).Assembly.GetExportedTypes()
            .Where(t => t.Name.EndsWith("Controller") && t.IsClass);

        foreach (var controller in controllerTypes)
        {
            var hasAuthorize = controller.GetCustomAttribute<AuthorizeAttribute>() != null;
            var hasAllowAnon = controller.GetCustomAttribute<AllowAnonymousAttribute>() != null;
            
            Assert.True(hasAuthorize || hasAllowAnon,
                $"Controller {controller.Name} must have [Authorize] or [AllowAnonymous]");
        }
    }

    [Fact]
    public void AllRepositories_ShouldBeInfrastructure_AndImplementInterface()
    {
        var result = Types.InCurrentDomain()
            .That()
            .HaveNameEndingWith("Repository")
            .And()
            .AreNotInterfaces()
            .Should()
            .ResideInNamespaceContaining("Infrastructure")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void NoCircularDependencies_BetweenServices()
    {
        // ตรวจสอบด้วย NetArchTest
        var result = Types.InCurrentDomain()
            .That()
            .ResideInNamespace("MyApp.Application.Services")
            .ShouldNot()
            .HaveDependencyOnAny("MyApp.Infrastructure")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }
}
```

### สรุป: Test Coverage Strategy

```csharp
// appsettings.Testing.json — config สำหรับ test environment
{
    "ConnectionStrings": {
        "DefaultConnection": "InMemory"
    },
    "Features": {
        "SendEmails": false,
        "UseCache": false
    },
    "Logging": {
        "LogLevel": {
            "Default": "Warning"
        }
    }
}

// Program.cs — ตรวจสอบ environment สำหรับ test
var builder = WebApplication.CreateBuilder(args);

// เพิ่ม test-specific services
if (builder.Environment.IsEnvironment("Testing"))
{
    // ใช้ in-memory services
    builder.Services.AddSingleton<IEmailService, FakeEmailService>();
}
else
{
    builder.Services.AddTransient<IEmailService, SmtpEmailService>();
}
```

### Test กฎที่ควรจำ

```csharp
// ARRANGE - ACT - ASSERT Pattern
public class OrderCalculatorTests
{
    // ✅ ชื่อ test บอก: [Method]_[Condition]_[ExpectedResult]
    [Fact]
    public void CalculateTax_ForOrderOver1000_AppliesReducedRate()
    {
        // Arrange — เตรียม test data
        var calculator = new TaxCalculator();
        var order = new Order { TotalAmount = 1500m };

        // Act — เรียก method ที่ทดสอบ
        var tax = calculator.Calculate(order);

        // Assert — ตรวจสอบผลลัพธ์
        Assert.Equal(1500m * 0.05m, tax); // 5% reduced rate
    }

    // ✅ ใช้ TestCase สำหรับหลาย inputs
    [Theory]
    [InlineData(500m, 0.07m)]    // 7% สำหรับ <= 1000
    [InlineData(1000m, 0.07m)]   // boundary
    [InlineData(1001m, 0.05m)]   // 5% สำหรับ > 1000
    [InlineData(5000m, 0.05m)]   // high value
    public void CalculateTax_ByAmount_AppliesCorrectRate(decimal amount, decimal expectedRate)
    {
        var calculator = new TaxCalculator();
        var order = new Order { TotalAmount = amount };
        
        var tax = calculator.Calculate(order);
        
        Assert.Equal(amount * expectedRate, tax);
    }
}
```

---

## สรุป Steps 721–730

| Step | หัวข้อ | เครื่องมือ | ความสำคัญ |
|------|--------|-----------|----------|
| 721 | Test Pyramid & Doubles | Moq, Manual | สูงมาก |
| 722 | Architecture Tests | NetArchTest | สูง |
| 723 | Contract Testing | Pact.NET | ปานกลาง-สูง |
| 724 | Property-Based Testing | FsCheck | ปานกลาง |
| 725 | Mutation Testing | Stryker.NET | ปานกลาง |
| 726 | Load Testing | NBomber | สูง |
| 727 | Snapshot Testing | Verify | ปานกลาง |
| 728 | Test Data Management | Bogus, AutoFixture | สูงมาก |
| 729 | Time Testing | FakeTimeProvider | สูง |
| 730 | Full Test Suite | WebApplicationFactory | สูงมาก |

### NuGet Packages ที่ใช้ใน Part นี้

```xml
<!-- Testing Framework -->
<PackageReference Include="xunit" Version="2.6.6" />
<PackageReference Include="xunit.runner.visualstudio" Version="2.5.6" />
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />

<!-- Mocking -->
<PackageReference Include="Moq" Version="4.20.70" />
<PackageReference Include="AutoFixture.AutoMoq" Version="4.18.1" />
<PackageReference Include="AutoFixture.Xunit2" Version="4.18.1" />

<!-- Architecture Testing -->
<PackageReference Include="NetArchTest.Rules" Version="1.3.2" />

<!-- Contract Testing -->
<PackageReference Include="PactNet" Version="4.5.0" />

<!-- Property-Based Testing -->
<PackageReference Include="FsCheck.Xunit" Version="2.16.6" />

<!-- Mutation Testing (dotnet tool) -->
<!-- dotnet tool install -g dotnet-stryker -->

<!-- Load Testing -->
<PackageReference Include="NBomber" Version="5.4.0" />
<PackageReference Include="NBomber.Http" Version="5.4.0" />

<!-- Snapshot Testing -->
<PackageReference Include="Verify.Xunit" Version="22.5.0" />

<!-- Test Data -->
<PackageReference Include="Bogus" Version="35.3.0" />

<!-- Time Testing -->
<PackageReference Include="Microsoft.Extensions.TimeProvider.Testing" Version="8.7.0" />

<!-- Web API Testing -->
<PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.2" />
```

---

**ก่อนหน้า → [Part 72: Resilience](part72-resilience.md)**

**ต่อไป → [Part 74: Message Bus Patterns](part74-message-bus.md)**
